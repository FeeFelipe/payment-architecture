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

<img width="2356" height="1697" alt="image" src="https://github.com/user-attachments/assets/bb39fb1b-665e-4647-994c-07b74c0f0c9b" />

# Escalabilidade e Alta Disponibilidade — Estratégia de Arquitetura

| **Camada / Componente**      | **Estratégia Recomendada**                                          | **Justificativa / Como aplicamos**                                                                                                                                                                    |
|------------------------------|---------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Arquitetura**              | **Microserviços stateless e event-driven**                          | Cada domínio (Pagamentos, Contas, Ledger) escala independentemente conforme carga. Serviços são *stateless* (usam Redis/Kafka para estado e mensageria), permitindo auto-scaling horizontal imediato. |
| **Infraestrutura / Deploy**  | **Kubernetes multi-AZ com auto-scaling (HPA/VPA)**                  | Clusters distribuídos em múltiplas zonas de disponibilidade. *Horizontal Pod Autoscaler* ajusta réplicas por CPU/memória/latência; *Vertical Pod Autoscaler* otimiza limites automaticamente.         |
| **Banco de Dados (MongoDB)** | **Cluster Sharded + réplica set (3 nós)**                           | Sharding por `tenantId` garante distribuição linear da carga. Réplicas fornecem HA e failover automático em caso de falha de nó ou zona.                                                              |
| **Ledger (Cassandra)**       | **Cluster distribuído multi-datacenter com Replication Factor ≥ 3** | Escrita em quorum (`LOCAL_QUORUM`) garante consistência e disponibilidade mesmo sob falhas parciais. Escala linear com novos nós.                                                                     |
| **Mensageria (Kafka)**       | **Cluster replicado (3 brokers) com partitioning e DLQ**            | Partições por `tenantId` e replicação (`min.insync.replicas=2`) garantem throughput e resiliência a falhas.                                                                                           |
| **Gateway / API Edge**       | **Kong com failover e autoscaling**                                 | Multi-replica, com health-checks e *rate limiting* dinâmico por tenant. Failover automático entre zonas.                                                                                              |
| **Autenticação (Keycloak)**  | **Cluster ativo-ativo + cache de sessão distribuído (Infinispan)**  | Nenhum ponto único de falha; replicação de sessão entre nós; escalável horizontalmente.                                                                                                               |
| **Cache (Redis)**            | **Cluster Redis Sentinel ou Redis Enterprise**                      | Detecta falhas e promove réplicas automaticamente; TTLs controlam uso eficiente de memória.                                                                                                           |
| **CI/CD**                    | **GitOps (ArgoCD) + Canary Deployment / Blue-Green**                | Permite *rollout gradual* e *rollback imediato* sem downtime; automatiza recuperação.                                                                                                                 |
| **Observabilidade**          | **OpenTelemetry + Prometheus + Loki + Tempo + Grafana**             | Detecta gargalos, erros e saturações em tempo real; métricas e alertas baseados em SLOs garantem proatividade.                                                                                        |
| **Resiliência de Rede**      | **Envoy / Service Mesh (mTLS, retries, circuit breaker)**           | Reduz impacto de falhas transitórias; mantém comunicações seguras e controladas entre serviços.                                                                                                       |
| **Backpressure e Proteção**  | **Fila Kafka + Dead Letter Queue + Retry policy**                   | Absorve picos de carga e desacopla produtores/consumidores, evitando sobrecarga.                                                                                                                      |


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

# HTTP Header

| headr             | origem      | exemplo                | descricao                                                                                                   |
| ----------------- | ----------- | ---------------------- | ----------------------------------------------------------------------------------------------------------- |
| `Authorization`   | Cliente/BFF | `Bearer eyJhbGciOi...` | JWT do usuário (ou token técnico no BFF) para autenticação e autorização.                                   |
| `X-Tenant-Id`     | BFF/Gateway | `acme-123`             | Identidade do tenant. Deve **bater** com `tenant_id` do token; usada para roteamento e RLS.                 |
| `Idempotency-Key` | Cliente/BFF | `idem_01J8Q9P2W8Z7`    | Chave única por requisição **POST** para evitar reprocessamento. Reutilize a mesma em retries.              |
| `X-Timestamp`     | BFF         | `2025-11-10T23:15:00Z` | Timestamp ISO-8601 UTC para janela anti-replay (ex.: ±5 min).                                               |
| `X-Key-Id`        | BFF         | `pay-bff-2025-01`      | Identificador da chave HMAC ativa (permite rotação/revogação).                                              |
| `X-Signature`     | BFF         | `v1=Q3Bq3vJ0b6v...`    | Assinatura **HMAC-SHA256** da string canônica (método, path, query, SHA256(body), tenant, idem, timestamp). |
| `X-Request-Id`    | Cliente/BFF | `req_01J8Q9S4K3R2`     | ID de correlação/idempotência de logs e tracing ponta-a-ponta.                                              |

# Service Level Objectives (SLOs)

| Categoria                | Indicador (SLI)                                 | Meta (SLO)                         | Justificativa                                                                 |
|---------------------------|-------------------------------------------------|------------------------------------|--------------------------------------------------------------------------------|
| **API – Disponibilidade** | % de requisições HTTP 2xx/3xx                  | ≥ 99.95% por mês                   | Alta confiabilidade em operações críticas (pagamentos e ledger).              |
| **API – Latência**        | P95 do tempo de resposta                       | ≤ 300 ms                           | Garantir experiência responsiva e previsível.                                 |
| **API – Erros**           | % de erros 5xx                                 | ≤ 0.1%                             | Controlar falhas internas e alertar anomalias.                                |
| **API – Idempotência**    | % de reprocessamentos detectados               | ≤ 0.01%                            | Prevenir duplicidade de transações financeiras.                               |
| **Ledger – Consistência** | % de eventos aplicados com sucesso             | 100% (auditado diariamente)        | Ledger é fonte de verdade; nenhum evento pode ser perdido.                    |
| **Ledger – Latência**     | Tempo entre evento e gravação persistida       | P99 ≤ 500 ms                       | Evitar backlog de contabilidade e garantir consistência quase em tempo real.  |
| **Eventos – Entrega**     | % de mensagens processadas com sucesso         | ≥ 99.9%                            | Garantir processamento confiável de todos os eventos.                         |
| **Eventos – Latência E2E**| Tempo entre publicação e consumo                | P95 ≤ 1 seg                        | Evitar acúmulo e atrasos de propagação.                                       |
| **Eventos – Reprocessamento** | % de mensagens reprocessadas                | ≤ 0.1%                             | Identificar instabilidade e erros de idempotência.                            |
| **DLQ – Volume**          | % de mensagens que caem na DLQ                 | ≤ 0.01%                            | Minimizar mensagens com falha irrecuperável.                                  |
| **DLQ – Tempo de limpeza**| Tempo médio para reprocessar DLQ               | ≤ 15 min                           | DLQ limpa e auditada constantemente; impacto mínimo.                          |
| **Pagamentos – Sucesso**  | % de pagamentos concluídos sem erro            | ≥ 99.9%                            | Métrica central de sucesso do negócio.                                        |
| **Pagamentos – Tempo médio** | Tempo total (request → commit)             | P95 ≤ 5 s                          | Mantém experiência previsível e evita SLIs de backlog.                        |
| **Conciliação**           | % de transações reconciliadas automaticamente  | ≥ 99.5%                            | Reduz esforço manual e risco de erro contábil.                                |

# Parâmetros de Continuidade

| Métrica                            | Meta            | Descrição                                                           |
| ---------------------------------- | --------------- | ------------------------------------------------------------------- |
| **RTO (Recovery Time Objective)**  | ≤ **5 minutos** | Tempo máximo para restaurar o serviço em caso de falha.             |
| **RPO (Recovery Point Objective)** | ≤ **1 minuto**  | Tempo máximo de perda de dados tolerável.                           |


# Estratégia de Deploy

## Modelo de Entrega

| Item | Estratégia |
|------|-------------|
| **Padrão principal** | Deploy contínuo via **GitOps (Argo CD)** |
| **Tipo de rollout** | **Canary progressivo** com fallback **Blue-Green** |
| **Controle de promoção** | Baseado em métricas e **SLOs de latência, erro e estabilidade** |
| **Orquestração** | **Argo Rollouts** com análise automática |
| **Rollback** | Automático em caso de violação de SLOs |
| **Ambiente** | Kubernetes com versionamento de manifests (Helm/Kustomize) |
| **Auditoria** | Todo deploy rastreável por commit/tag (GitOps + TraceID) |

---

## Fluxo de Deploy

| Etapa | Ação principal | Meta / Critério |
|-------|----------------|-----------------|
| **1. Build & Test** | CI executa build, testes e análise estática | Tudo verde |
| **2. Gerar imagem e manifestos** | Tag imutável + commit em branch de release | Imagem versionada e assinada |
| **3. Argo CD Sync** | Argo aplica versão nos clusters-alvo | Deploy iniciado |
| **4. Canary** | Tráfego 5% → 25% → 50% → 100% | Nenhum SLO violado |
| **5. Observação pós-rollout** | Monitorar métricas críticas por 30–60 min | Estabilidade confirmada |
| **6. Blue-Green fallback** | Reversão imediata se métricas degradarem | Rollback automático |
| **7. Aprovação de produção** | Confirmação final via Pull Request GitOps | Deploy completo e versionado |

---

## Estratégia de Canary

| Fase | Percentual de tráfego | Critério de promoção |
|------|-----------------------|----------------------|
| Fase 1 | 5% | Nenhum erro 5xx e latência P95 < 300 ms |
| Fase 2 | 25% | DLQ ≤ 0.01% e lag Kafka ≤ 1 s |
| Fase 3 | 50% | Todos os SLOs dentro das metas |
| Fase 4 | 100% | Estabilidade ≥ 30 min sem regressão |
| Rollback | — | Violação de qualquer SLO ou falha de readiness |

---

## Rollback Automático

| Condição de falha | Ação automática | Impacto |
|-------------------|-----------------|----------|
| Aumento de erros 5xx | Reverter para última versão estável | Tráfego retorna em <1 min |
| Latência P95 > 300 ms | Pausar rollout e reverter | Evita degradação global |
| Consumer lag > 5 s | Parar promoção e reverter consumers | Garante estabilidade de eventos |
| DLQ > 0.05% | Pausar deploy e gerar alerta SRE | Previne perda de mensagens |
| Falha de readiness / liveness | Rollback imediato via Argo | Garantia de disponibilidade |

---

## Blue-Green (Fallback)

| Item | Descrição |
|------|------------|
| **Estratégia** | Implantar nova versão em ambiente paralelo (“green”), mantendo atual (“blue”) ativa |
| **Switch** | Redirecionar tráfego via Ingress ou Service mesh |
| **Rollback** | Instantâneo, apenas revertendo o alias de tráfego |
| **Tempo médio de reversão** | < 30 s |



