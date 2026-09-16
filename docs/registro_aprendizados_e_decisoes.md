# Registro de Aprendizados, Decisões e Melhorias (Agente + Analista)

## Objetivo deste documento

Consolidar o que foi identificado durante a elaboração da documentação arquitetural do HUBVENDAS, separando:

- melhorias sugeridas pelo agente;
- decisões tomadas pelo analista;
- falhas encontradas e corrigidas;
- inferências corretas do modelo;
- ajustes necessários feitos manualmente;
- lacunas que ainda precisam ser documentadas para permitir implementação por agentes sem invenção de decisões.

---

## 1) Melhorias identificadas pelo agente

1. Estruturar o `README` como documento arquitetural de referência, e não apenas descrição breve do repositório.
2. Incluir diagramas Mermaid para cobrir visão estrutural e comportamental.
3. Explicitar conceitos de mensageria (DLQ, retry, backoff e reprocessamento) para evitar ambiguidade tecnica.
4. Adicionar seções de diretrizes e checklist de aderência arquitetural para orientar implementações futuras.
5. Tornar os fluxos de autenticação e retorno de status visíveis nos diagramas.

---

## 2) O que foi analisado e decidido implementar

1. **Documentação principal em `README.md`:**
   concentrar descrição, diagramas, decisões e diretrizes no arquivo raiz.

2. **Topologia de filas por sequência de negócio:**
   organização explícita em:
   - pedido de venda;
   - lançamento financeiro;
   - baixa do lançamento financeiro;
   com priorização por tipo de evento.

3. **Autenticação OAuth2 sob responsabilidade do ERP:**
   o controle de emissão/validação de token para chamadas da integração ao ERP foi definido como responsabilidade do ERP.

4. **Feedback obrigatório para o sistema de origem:**
   retorno de status final da transação não é opcional; deve ocorrer obrigatoriamente.

5. **Manutenção de arquitetura assíncrona com resiliência:**
   persistência crítica via filas/workers, com retry e DLQ.

---

## 3) Falhas encontradas e corrigidas

1. **Diagrama de contexto desorganizado na renderização Mermaid:**
   - causa: excesso de aninhamento e cruzamento de setas;
   - correção: simplificação de layout para melhorar legibilidade.

2. **Trechos com marcadores de linha indevidos no `README`:**
   - sintoma: aparição de prefixos como `10|`, `20|` no conteúdo;
   - correção: limpeza do conteúdo e normalização do texto.

3. **Modelagem inicial de autenticação como IdP externo:**
   - divergência: regra de negócio informada depois definiu OAuth2 gerenciado pelo ERP;
   - correção: ajuste nos diagramas e na seção de decisões.

4. **Notificação de retorno marcada como opcional no início:**
   - divergência: retorno deveria ser obrigatório;
   - correção: ajuste no diagrama de sequência e no texto de decisões.

---

## 4) O que o modelo inferiu corretamente

1. Necessidade de uma camada ACL para desacoplar front-office e ERP.
2. Uso de RabbitMQ como mecanismo central de assincronia e resiliência.
3. Importância de idempotência para evitar duplicidade em cenários de reenvio.
4. Necessidade de tratamento de falhas com retentativa e DLQ.
5. Relevancia de documentar os fluxos fim a fim (entrada, processamento, envio ao ERP e retorno).

---

## 5) O que precisou de ajuste manual/humano

1. Definição exata da responsabilidade da autenticação OAuth2 (ERP).
2. Definição da obrigatoriedade do feedback de retorno para o sistema de origem.
3. Definição da organização das filas por sequência funcional específica do domínio.
4. Refino de comunicação do documento para linguagem mais prescritiva para implementação.

---

## 6) O que a documentação ainda precisa ter para um agente implementar sem inventar decisões

Para reduzir inferências e variação de implementação, o ideal é complementar com os itens abaixo:

1. **Contratos canônicos de eventos**
   - schemas versionados por evento (`pedido_venda`, `lancamento_financeiro`, `baixa_financeira`);
   - obrigatoriedade de campos, tipos, validações e exemplos.

2. **Especificação de topologia RabbitMQ detalhada**
   - nomes oficiais de exchanges, filas, DLQs e routing keys;
   - política de retry/backoff (intervalos, limite, critérios de falha transiente/definitiva).

3. **Matriz de regras por BU**
   - regras de validação, tributação, contabilização e comissionamento por BU;
   - prioridade e precedência de regras em caso de conflito.

4. **Contrato de integração com ERP**
   - endpoints, payloads, códigos de retorno e tabela de erros;
   - políticas de timeout, circuit breaker e idempotency key.

5. **Fluxo de autenticação OAuth2 operacional**
   - como obter token, renovação, expiração, revogação e armazenamento seguro;
   - comportamento em erro de autenticação/autorização.

6. **SLA e observabilidade**
   - tempos esperados por etapa, limites de fila, alarmes;
   - logs estruturados, correlação (correlation-id) e trilha de auditoria mínima.

7. **Política de notificação obrigatória de retorno**
   - canal/protocolo de retorno para origem;
   - payload de status final (sucesso/falha), códigos e mensagens padronizadas.

8. **Critérios de pronto para implementação**
   - checklist técnico mínimo por módulo;
   - cenários de teste obrigatórios (funcionais, erro, reprocessamento, idempotência).

---

## 7) Recomendação de uso deste repositório por agentes

- Tratar `README.md` como fonte primária de arquitetura.
- Tratar este arquivo como trilha de decisões e riscos de interpretação.
- Antes de codificar, validar se existe definição explícita para cada decisão técnica.
- Quando faltar definição, registrar lacuna e solicitar decisão, em vez de assumir comportamento.
