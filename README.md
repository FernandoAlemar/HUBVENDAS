# HUBVENDAS

Camada de integracao responsavel por centralizar, orquestrar e transformar o fluxo de dados operacionais originados no sistema especializado de turismo em direcao ao ERP de backoffice.

## Descricao da Solucao

O HUBVENDAS atua como um barramento intermediario inteligente entre o front-office de turismo e o ERP corporativo, com foco em:

- desacoplamento entre origem de vendas e core de backoffice;
- processamento assincrono com RabbitMQ;
- isolamento de regras de negocio por Unidade de Negocio (10 BUs);
- resiliencia, idempotencia e rastreabilidade ponta a ponta;
- seguranca por token, incluindo OAuth2 gerenciado pelo ERP nas chamadas da integracao para o ERP, e comunicacao restrita a VPN.

A camada recebe transacoes de pedidos de venda e titulos a receber, valida e normaliza os dados, aplica regras por BU, publica/consome filas e encaminha cargas estruturadas para um stage no ERP, onde jobs internos distribuem o processamento para os modulos corporativos.

## Diagramas Estruturais (Mermaid)

### 1) Contexto de Integracao (visao macro)

```mermaid
flowchart LR
    turismo["Sistema Especializado de Turismo\n(Front-office)"]
    erpAuth["OAuth2 ERP\n(emissao/validacao de token)"]

    subgraph vpn["Perimetro Seguro - VPN Corporativa"]
        hub["HUBVENDAS\n(Camada de Integracao / ACL)"]
        mq["RabbitMQ\n(Filas, Exchanges, DLQ)"]
        erp["ERP Backoffice Corporativo\n(Financeiro, Contabil, Fiscal)"]
    end

    turismo -->|"Pedidos + Titulos"| hub
    hub -->|"Publicacao e consumo assincrono"| mq
    hub -->|"Autenticacao OAuth2"| erpAuth
    hub -->|"Carga normalizada (stage ERP)"| erp
    erp -.->|"IDs, status, rejeicoes"| hub
```

### 2) Containers Internos da Camada HUBVENDAS

```mermaid
flowchart TB
    subgraph hub["HUBVENDAS - Containers internos"]
        inbound["API Inbound\nREST/Webhook/Pooling"]
        auth["Auth Gateway\nValidacao inbound + OAuth2 ERP outbound"]
        orchestration["Orquestrador de Integracao"]
        rules["Motor de Regras por BU\n(ACL / Adaptadores)"]
        canonical["Normalizador Canonico\n(Venda/Titulos)"]
        idempotency["Servico de Idempotencia\n(controle de duplicidade)"]
        audit["Log + Auditoria + Status"]
        retry["Retentativa/Reprocessamento"]
        outbound["Conector ERP\n(Envio para stage)"]
        mqPub["Publisher RabbitMQ"]
        mqCon["Consumer RabbitMQ"]
    end

    inbound --> auth
    auth --> orchestration
    orchestration --> canonical
    canonical --> rules
    rules --> idempotency
    idempotency --> mqPub
    mqCon --> retry
    retry --> outbound
    orchestration --> audit
    rules --> audit
    retry --> audit
```

### 3) Topologia Logica de Mensageria (RabbitMQ)

```mermaid
flowchart LR
    prod["Produtores\n(Sistema Turismo / HUB Inbound)"]
    ex["Exchange principal\n(vendas.titulos.exchange)"]

    q1["Fila BU01..BU10\n(particionamento logico)"]
    q2["Fila prioritaria\n(criticidade alta)"]
    q3["Fila padrao\n(criticidade normal)"]
    dlx["Dead Letter Exchange"]
    dlq["Dead Letter Queue"]
    consumer["Workers HUBVENDAS"]

    prod --> ex
    ex -->|"routing key por BU"| q1
    ex -->|"routing key prioridade"| q2
    ex -->|"fallback"| q3
    q1 --> consumer
    q2 --> consumer
    q3 --> consumer
    q1 -->|"falha apos N tentativas"| dlx
    q2 -->|"falha apos N tentativas"| dlx
    q3 -->|"falha apos N tentativas"| dlx
    dlx --> dlq
```

## Diagrama Comportamental (Mermaid)

### 4) Sequencia - Jornada critica de pedido/titulo ate confirmacao no ERP

```mermaid
sequenceDiagram
    autonumber
    participant T as Sistema Turismo
    participant A as API HUBVENDAS
    participant G as Auth Gateway
    participant O as Orquestrador/ACL
    participant M as RabbitMQ
    participant W as Worker HUBVENDAS
    participant EA as OAuth2 ERP
    participant E as ERP (stage + job)
    participant D as DLQ

    T->>A: Envia pedido + titulos (payload)
    A->>G: Validar token OAuth2
    G-->>A: Token valido
    A->>O: Encaminha mensagem valida
    O->>O: Valida sintaxe/conteudo + identifica BU
    O->>O: Normaliza e aplica regras da BU
    O->>O: Verifica idempotencia (chave transacao)
    O->>M: Publica mensagem em fila de trabalho
    M-->>W: Entrega mensagem
    W->>EA: Solicita token para integracao com ERP
    EA-->>W: Token de acesso
    W->>E: Envia carga estruturada ao stage ERP
    E-->>W: Retorna IDs/status ou rejeicao

    alt sucesso
        W->>O: Atualiza status = Processado
    else falha transiente
        W->>M: Reencaminha para retentativa
    else falha definitiva
        W->>D: Encaminha para DLQ
        W->>O: Atualiza status = Falha tecnica
    end
    O-->>T: Notificacao obrigatoria de retorno (status final)
```

## Conceitos de Mensageria

- **Fila de trabalho:** fila principal onde as transacoes validas aguardam processamento assincrono.
- **Retentativa (retry):** nova tentativa automatica quando ocorre falha transiente (ex.: indisponibilidade temporaria do ERP).
- **Backoff:** estrategia de espacamento entre tentativas (ex.: progressivo/exponencial) para reduzir sobrecarga e aumentar chance de recuperacao.
- **DLQ (Dead Letter Queue):** fila de isolamento para mensagens que falharam apos o limite de tentativas ou possuem erro nao recuperavel.
- **Reprocessamento:** rotina operacional para analisar mensagens da DLQ, corrigir causa raiz e reenviar com seguranca.

## Decisoes e Ajustes de Modelagem

A especificacao traz alguns pontos em aberto. Para viabilizar uma representacao arquitetural coerente e renderizavel em Mermaid, foram adotados os ajustes abaixo:

1. **Topologia de filas representada como hibrida (BU + criticidade):**
   como a especificacao lista a topologia como lacuna, o diagrama mostra uma alternativa que combina roteamento por BU e por prioridade para refletir escalabilidade e controle operacional.

2. **Conector de ERP modelado com stage intermediario:**
   foi explicitado no fluxo que o HUB envia para um stage do ERP e que um job interno do ERP conclui a distribuicao aos modulos, alinhado ao texto da especificacao.

3. **OAuth2 representado como responsabilidade do ERP:**
   o fluxo de autenticacao para chamadas da integracao ao ERP foi modelado como servico OAuth2 gerenciado pelo proprio ERP, deixando explicita essa fronteira de responsabilidade.

4. **Feedback para sistema de origem definido como obrigatorio:**
   a notificacao de retorno foi modelada como etapa obrigatoria ao final do processamento, garantindo visibilidade de status para o sistema de origem.

5. **Containers internos organizados por responsabilidade (nao por tecnologia):**
   os componentes internos da camada foram definidos por capacidades (auth, orquestracao, regras BU, idempotencia, retry, auditoria) para manter aderencia ao nivel logico pedido.

6. **DLQ e retentativas separadas no comportamento:**
   a jornada critica explicita falha transiente versus falha definitiva para evidenciar resiliencia, sem fixar politicas numericas ainda nao definidas.

## Fonte da Especificacao

Documento base: `.vscode/especificacao_camada_integracao.md`.
