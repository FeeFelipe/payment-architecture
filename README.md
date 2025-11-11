# Payment Architecture

# Classificação dos Domínios

| Categoria              | Domínio                        | Descrição                                                                                              |
|------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------|
| **Core Domain**        | **Pagamentos**                 | Responsável por criar, autorizar, liquidar e estornar pagamentos entre contas de pagamento.            |
| **Core Domain**        | **Ledger**                     | Registra lançamentos imutáveis, garantindo integridade contábil e rastreabilidade.    |
| **Core Domain**        | **Contas de Pagamento**        | Gerencia o ciclo de vida das contas, vínculo com usuários e disponibilidade de saldo para transações.  |
| **Supporting Domain**  | **Gestão de Usuários**         | Administra o cadastro, perfis, autenticação e status de clientes e lojistas.                           |
| **Supporting Domain**  | **Saldo e Extratos**           | Consolida movimentações derivadas do Ledger e apresenta o histórico e saldo das contas aos usuários.   |
| **Generic Domain**     | **Integrações com Provedores** | Orquestra a comunicação com bancos, PIX e gateways externos, traduzindo comandos e recebendo callbacks.|
| **Generic Domain**     | **Notificações**               | Gera comunicações automáticas (e-mail, push, SMS) baseadas em eventos do sistema.                      |


# Mapa de Contexto

<img width="2899" height="1722" alt="Sem título-2025-11-08-2103" src="https://github.com/user-attachments/assets/c275066e-0bee-4106-a06b-4f188cabfdcd" />

# System context diagram

<img width="3750" height="1502" alt="Sem título-2025-11-08-21032" src="https://github.com/user-attachments/assets/4345ba58-cc65-47ac-8d24-28130004fdb7" />

# Architecture Blueprint

<img width="2244" height="1781" alt="image" src="https://github.com/user-attachments/assets/7c83b2b3-f05c-4879-b897-532334dd9fee" />

# Stack Tecnológica 

| **Camada / Componente**          | **Tecnologia Recomendada**            | **Justificativa** |
|----------------------------------|---------------------------------------|--------------------|
| **Banco de dados (Core)**        | MongoDB                               | Banco NoSQL orientado a documentos, oferece flexibilidade de schema e suporte a transações ACID distribuídas para contas e pagamentos. |
| **Ledger (Contábil)**            | Apache Cassandra                      | Banco distribuído *append-only*, ideal para grandes volumes de lançamentos imutáveis e auditoria de transações financeiras. |
| **Cache / Estado de Saga**       | Redis                                 | Armazena estado temporário e chaves idempotentes com baixa latência e TTL configurável. |
| **Mensageria / Eventos**         | Apache Kafka                          | Backbone *event-driven* para orquestração entre domínios; garante ordenação, alta vazão e reprocessamento via DLQ. |
| **Autenticação / Autorização**   | Keycloak (OIDC + MFA)                 | Gerencia identidades multi-tenant, autenticação segura (MFA/WebAuthn) e RBAC/ABAC centralizado. |
| **API Edge**                     | Kong Gateway                          | Roteamento externo, *rate limiting*, OIDC, WAF e métricas nativas. |
| **Service Mesh**                 | Envoy                                 | Oferece mTLS, RBAC, *retries*, *circuit breaking* e telemetria distribuída. |
| **Infraestrutura / Orquestração**| Kubernetes (multi-AZ)                 | Alta disponibilidade, escalabilidade horizontal e isolamento de serviços. |
| **CI/CD**                        | GitHub Actions + Argo CD (GitOps)     | Entrega contínua automatizada e auditável, com rollback controlado e versionamento declarativo. |
| **Segredos / Criptografia**      | Vault                                 | Gerencia segredos e chaves com rotação automática, criptografia *envelope* e controle de acesso granular. |
| **Logs**                         | Loki                                  | Centraliza e indexa logs estruturados com baixo custo e integração com Grafana. |
| **Métricas**                     | Prometheus                            | Coleta métricas de desempenho e SLOs para alertas e *capacity planning*. |
| **Traces distribuídos**          | OpenTelemetry                         | Fornece rastreabilidade ponta a ponta entre microserviços, crucial para diagnosticar falhas em Sagas e fluxos assíncronos. |
| **Dashboards / Observabilidade** | Grafana                               | Unifica métricas, logs e traces em painéis de monitoração e alertas centralizados. |
| **Feature Flags (opcional)**     | Unleash                               | Permite ativar/desativar recursos dinamicamente por *tenant*, reduzindo risco em deploys contínuos. |

# Estratégia de Multi-Tenant

| **Camada / Componente**     | **Estratégia Recomendada**                                           | **Justificativa / Como aplicar** |
|------------------------------|---------------------------------------------------------------------|----------------------------------|
| **Modelo de Dados (MongoDB)** | Shared Cluster + Collections com `tenantId` obrigatório             | Todas as coleções incluem `tenantId` e possuem índice `{ tenantId: 1 }`. O valor é extraído do JWT (Keycloak) e aplicado automaticamente em todas as queries. O `Schema Validation` impede inserções sem `tenantId`, garantindo isolamento lógico e alta escalabilidade. |
| **Cassandra (Ledger)**       | Partição por `tenant_id` + chave composta                            | Segrega dados por tenant na partição, mantém ordenação temporal para *append-only* e facilita purge/TTL por tenant. Evita partições gigantes (shard por `account_id`). |
| **Roteamento de Requisições**| API Gateway (Kong) extrai `tenant_id` do OIDC (Keycloak) e injeta `X-Tenant-Id` | Todos os serviços confiam no token assinado, não em parâmetros do cliente. Middleware de persistência usa esse valor para escopo de queries. |



