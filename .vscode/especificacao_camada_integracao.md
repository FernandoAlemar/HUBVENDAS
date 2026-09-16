### Escopo

Desenvolver uma camada de integração responsável por centralizar, orquestrar e transformar o fluxo de dados operacionais originados no sistema especializado de turismo em direção ao ERP de backoffice.

A solução contemplará:

- Integração entre as 10 Unidades de Negócio (BUs) do Grupo;
- Tratamento individualizado dos dados por unidade de negócio;
- Recebimento e processamento de pedidos de venda e títulos a receber;
- Centralização das validações e regras de negócio específicas por BU dentro da própria aplicação de integração, visando conferir escalabilidade operacional e permitir a fácil substituição do sistema especialista de vendas no futuro com baixo impacto no ERP;
- Uso de filas gerenciadas pelo RabbitMQ para garantir o desacoplamento, controle e processamento assíncrono das mensagens;
- Envio dos dados tratados ao ERP responsável pelo backoffice financeiro, contábil e fiscal;
- Comunicação protegida por autenticação baseada em token;
- Disponibilização e operação da camada de integração dentro da VPN do Grupo.

### Nível da Visão

Visão de Arquitetura de Solução e Integração, em nível lógico e sistêmico.

O foco reside no estabelecimento de um barramento intermediário inteligente e agnóstico à origem, promovendo:

- Desacoplamento arquitetural estratégico entre a ponta de vendas e o core corporativo;
- Portabilidade do sistema de vendas, reduzindo o acoplamento sistêmico (*vendor lock-in*) e simplificando substituições ou adições de novos sistemas transacionais de front-office;
- Escalabilidade horizontal e modular para comportar o crescimento das 10 BUs e novos canais sem sobrecarregar o ERP;
- Controle e rastreabilidade ponta a ponta das transações;
- Processamento assíncrono e resiliente por meio do RabbitMQ;
- Segurança na comunicação entre os sistemas;
- Segregação de domínio e dados conforme a respectiva Unidade de Negócio.

### Limites e Responsabilidades

#### Sistema Especializado de Turismo

Responsável por:

- Gerenciar as operações de front-office relacionadas ao negócio de turismo;
- Registrar e manter o ciclo de vida dos pedidos de venda;
- Originar os dados operacionais dos títulos a receber vinculados às transações;
- Disponibilizar os dados brutos ou normalizados para a camada de integração;
- Manter as informações operacionais de reservas, viagens ou serviços comercializados.

O sistema especializado não concentra a governança contábil/fiscal corporativa nem deve conter amarrações rígidas às tabelas e processos internos do ERP.

#### Camada de Integração

Responsável por:

- Consumir ou receber os dados brutos de vendas e títulos do sistema especialista;
- Validar a estrutura sintática e o conteúdo das mensagens;
- Identificar a Unidade de Negócio correspondente;
- **Isolar e executar as regras de negócio de cada BU**, atuando como camada de adaptação (*Anti-Corruption Layer* - ACL) para garantir escalabilidade e blindar o ERP e o front-office de mudanças mútuas;
- Transformar os modelos de dados de venda para o formato exigido pelo ERP;
- Publicar e consumir mensagens em filas gerenciadas pelo RabbitMQ;
- Controlar o processamento assíncrono das transações;
- Implementar mecanismos de retentativa, reprocessamento e tratamento em caso de falha;
- Garantir a idempotência das operações;
- Registrar logs técnicos e auditoria funcional do ciclo de vida das mensagens;
- Controlar os status de processamento de cada transação;
- Encaminhar os dados normalizados ao ERP;
- Proteger os endpoints por meio de autenticação baseada em token;
- Restringir a comunicação de rede ao perímetro seguro da VPN do Grupo.

#### ERP — Backoffice Corporativo

Responsável por:

- Registrar e faturar os pedidos de venda recebidos;
- Gerenciar a carteira de títulos a receber e tesouraria;
- Controlar o fluxo de caixa corporativo;
- Executar os processos e apurações fiscais;
- Realizar os lançamentos contábeis oficiais;
- Manter os registros canônicos financeiros, contábeis e fiscais do Grupo;
- Retornar à camada de integração os identificadores gerados, status de execução e eventuais códigos de rejeição.

### Integrações

#### Sistema Especializado de Turismo → Camada de Integração

A integração deverá possibilitar a extração ou recebimento das transações de vendas e títulos a receber originados no sistema especialista.

O protocolo (APIs REST/SOAP, webhooks, pooling ou arquivos) transferirá os dados para a camada de integração de forma desacoplada, viabilizando que esse sistema de vendas possa ser migrado ou substituído pontualmente sem alterar as interfaces do ERP. As mensagens recebidas serão enfileiradas no RabbitMQ.

#### RabbitMQ

O RabbitMQ atuará como barramento de mensageria assíncrona, viabilizando:

- Desacoplamento temporal e estrutural entre o sistema especialista, o motor de integração e o ERP;
- Amortecimento de picos de carga de vendas (buffer/throttling) para proteger a infraestrutura do ERP;
- Processamento assíncrono e ordenado conforme regras de concorrência por BU;
- Gerenciamento de filas de trabalho e Dead Letter Queues (DLQs) para isolamento e análise de mensagens não processadas;
- Políticas de retentativa automática configuráveis por tipo de falha.

#### Camada de Integração → ERP

A camada de integração orquestrará as chamadas aos serviços oficiais do ERP, enviando cargas estruturadas e previamente enriquecidas com as regras corporativas e fiscais calculadas, os dados enviados pela camada de integração são recebidos pelo ERP em um stage onde o Job construindo no ERP busque essas informações e envie para os modulos respequitivos dentro do ERP.

As operações contemplarão:

- Criação ou atualização de pedidos de venda;
- Criação e desdobramento de títulos a receber;
- Vinculação correta de empresa, filial, BU e centro de custo;
- Envio das parametrizações contábeis e fiscais necessárias para os módulos de faturamento e financeiro;
- Tratamento e consumo do retorno do ERP (IDs gerados, mensagens de erro, alertas de validação).

#### Segurança e Rede

Os fluxos de integração deverão cumprir os seguintes critérios de segurança:

- Autenticação e autorização via token  OAuth2 proprio em todos os pontos de entrada da camada;
- Políticas rígidas de ciclo de vida do token (expiração, renovação e revogação);
- Tráfego de rede exclusivamente restrito ao perímetro da VPN corporativa do Grupo;
- Políticas de firewall liberando apenas portas e IPs homologados entre os servidores do sistema de turismo, os nós do RabbitMQ, a camada de integração e o ERP;
- Criptografia em trânsito (HTTPS/TLS).

### Restrições e Lacunas

#### Restrições

- **Isolamento de Negócio na Camada de Integração:** As regras operacionais  de cada BU devem residir na aplicação de integração, mantendo o sistema de turismo e o ERP o mais agnósticos e padronizados possível, garantindo flexibilidade para eventual substituição do software de turismo;
- **Segregação Multi-BU:** Manutenção estrita do isolamento contábil, fiscal e operacional entre as 10 BUs;
- **Arquitetura Assíncrona Obrigatória:** Toda persistência crítica no ERP deve passar pelo controle do RabbitMQ;
- **Acesso Restrito:** Nenhuma comunicação direta com a camada ou com o ERP poderá ser exposta à internet pública, exigindo tráfego sob VPN;
- **Proteção por Token:** Nenhuma rota de serviço poderá ser acessada sem validação prévia de token;
- **Idempotência:** A camada deve possuir mecanismos de controle de duplicidade para evitar cobranças ou pedidos replicados no ERP em cenários de reenvio;
- **Resiliência:** A indisponibilidade de comunicação com o ERP não pode acarretar em perda de transações geradas pelo sistema de turismo.

#### Lacunas a Definir

- **Estratégia de Acoplamento da Origem:** Se a camada consumirá contratos genéricos/canônicos independentes do sistema de turismo atual para acelerar uma substituição futura, ou se fará a tradução direta do payload proprietário do software atual;
- **Topologia de Filas no RabbitMQ:** Se haverá filas e *exchanges* particionadas por BU, por criticidade, ou se haverá uma fila unificada com roteamento via *routing keys*;
- **Política de Retentativas e DLQ:** Definição dos intervalos de retentativa (*exponential backoff*), limite de tentativas e rotina operacional para reprocessamento manual via DLQ;
- **Provedor de Identidade (IdP):** Qual serviço será responsável por emitir, assinar e autenticar os tokens (ex.: Keycloak, OAuth próprio, Active Directory/Entra ID);
- **Topologia de VPN e Infraestrutura:** Definição de zonas de rede (VPC/subnets), roteamento de gateways entre a nuvem/on-premises do sistema de turismo e o datacenter do ERP;
- **Catálogo de Regras por BU:** Documentação detalhada dos cálculos, regras de comissionamento, tributação e parâmetros contábeis específicos de cada uma das 10 BUs;
- **Governança de De-Para e Cadastros Mestres:** Definição de onde residirá a tabela de equivalência para clientes, planos de contas, centros de custo, tipos de serviço e filiais;
- **Feedback Loop para a Origem:** Definição se o sistema especialista de turismo precisa receber atualizações de status vindas do ERP (como número do título faturado, NF gerada ou cancelamentos).
