# FIAP SOAT Tech Challenge - App

[![Quality Gate](https://sonarcloud.io/api/project_badges/measure?project=zmathmatos_fiap-soat-tech-challenge-app2&metric=alert_status)](https://sonarcloud.io/summary/overall?id=zmathmatos_fiap-soat-tech-challenge-app2)
[![Coverage](https://sonarcloud.io/api/project_badges/measure?project=zmathmatos_fiap-soat-tech-challenge-app2&metric=coverage)](https://sonarcloud.io/component_measures?id=zmathmatos_fiap-soat-tech-challenge-app2&metric=coverage)

API REST para gerenciamento de ordens de serviço em oficinas mecânicas.
Implementada com Node.js, Express, TypeScript, PostgreSQL e Sequelize.

Este repositório faz parte de uma arquitetura com 4 repositórios separados:

| Repositório | Conteúdo |
|---|---|
| **[fiap-soat-tech-challenge-app](https://github.com/zmathmatos/fiap-soat-tech-challenge-app)** | ← Este repo — Código da aplicação |
| [fiap-soat-tech-challenge-lambda](https://github.com/zmathmatos/fiap-soat-tech-challenge-lambda) | Lambda de autenticação via CPF (API Gateway + AWS Lambda) |
| [fiap-soat-tech-challenge-infra-k8s](https://github.com/zmathmatos/fiap-soat-tech-challenge-infra-k8s) | Infraestrutura Kubernetes (VPC, EKS) via Terraform |
| [fiap-soat-tech-challenge-infra-db](https://github.com/zmathmatos/fiap-soat-tech-challenge-infra-db) | Infraestrutura do banco de dados (RDS PostgreSQL) via Terraform |

## Desenvolvimento local

### Pré-requisitos

- Docker e Docker Compose

### Variáveis de ambiente

| Variável | Descrição | Exemplo |
|---|---|---|
| `DB_HOST` | Host do PostgreSQL | `localhost` ou `postgres-service` |
| `DB_PORT` | Porta do PostgreSQL | `5432` |
| `DB_NAME` | Nome do banco | `fiap_soat` |
| `DB_USER` | Usuário do banco | `postgres` |
| `DB_PASSWORD` | Senha do banco | `postgres` |
| `APP_PORT` | Porta da aplicação | `3000` |
| `JWT_SECRET` | Chave secreta JWT | `your-super-secret-key` |
| `JWT_EXPIRES_IN` | Expiração do token | `24h` |
| `LOG_LEVEL` | Nível de log (`debug`/`info`/`warn`/`error`) | `info` |
| `NEW_RELIC_LICENSE_KEY` | License key do New Relic (opcional em dev) | `NRAK-...` |
| `NEW_RELIC_APP_NAME` | Nome do serviço no NR | `fiap-web-dev` |
| `NEW_RELIC_ENABLED` | Habilita o agente APM | `true` |

### Subir a aplicação

```bash
npm run docker:dev
```

A API estará disponível em `http://localhost:3000`.

### Rodar os testes

```bash
# Subir banco de dados
npm run docker:dev

# Rodar todos os testes
npm test

# Apenas testes unitários
npm run test:unit

# Apenas testes de integração
npm run test:integration

# Com cobertura
npm run test:coverage
```

### Credenciais de teste

- **Admin**: `admin@techchallenge.com` / `admin123`
- **Customer**: `joao.silva@email.com` / `senha123`

## Observabilidade

A aplicação emite **logs JSON estruturados** via `pino`, com `correlationId` propagado em todas as requisições através do header `X-Correlation-Id` (gerado se ausente).

Eventos de domínio (mudança de status de OS) são emitidos como logs estruturados com os campos `order.id`, `order.status`, `service_order_number` — consumidos pelo dashboard do New Relic.

### New Relic APM

O agente Node.js (`newrelic`) é carregado no boot do servidor. Configuração via env vars:

- `NEW_RELIC_LICENSE_KEY` — license INGEST do NR
- `NEW_RELIC_APP_NAME` — nome do serviço (default `fiap-web-dev`)
- `NEW_RELIC_ENABLED` — `true` para ativar (default `false` em dev/test)
- `NEW_RELIC_LOG_LEVEL` — `info`/`debug`

Em K8s, essas envs vêm do `Secret` provisionado pelo módulo de observabilidade no repo `infra-k8s`. Dashboards, alertas e Synthetics são provisionados via Terraform — ver [`infra-k8s/docs/observability-setup.md`](https://github.com/zmathmatos/fiap-soat-tech-challenge-infra-k8s/blob/main/docs/observability-setup.md).

## CI/CD

### CI (`.github/workflows/ci.yml`)
Roda em `push`/`pull_request` para `master` e `develop`:

1. **lint-and-test** — testes com PostgreSQL
2. **sonarqube** — análise SonarCloud (somente push em `master`/`develop`)
3. **build** — build TypeScript + upload de artifact

### CD (`.github/workflows/cd.yml`)
Roda automaticamente em `push` para `master` (produção) e `develop` (homologação), com fallback manual via `workflow_dispatch`:

1. **build-and-push** — build da imagem Docker e push para Amazon ECR (tags `<sha>` e `latest`)
2. **deploy** — `aws eks update-kubeconfig`, aplica ConfigMap/Secret a partir de GitHub Secrets, aplica manifests em `k8s/`, roda o Job de migração e aguarda rollout do Deployment

### GitHub Secrets necessários

| Secret | Descrição |
|---|---|
| `AWS_ACCESS_KEY_ID` | Credencial AWS Academy |
| `AWS_SECRET_ACCESS_KEY` | Credencial AWS Academy |
| `AWS_SESSION_TOKEN` | Token de sessão (obrigatório no AWS Academy) |
| `AWS_REGION` | Região AWS (ex.: `us-east-1`) |
| `ECR_REPOSITORY` | Nome do repositório ECR (ex.: `fiap-soat-tech-challenge-app`) |
| `EKS_CLUSTER_NAME` | Nome do cluster EKS provisionado pelo repo de infra |
| `DB_HOST` | Endpoint do RDS (output do repo `infra-db`) |
| `DB_PORT` | Porta do RDS (ex.: `5432`) |
| `DB_NAME` | Nome do banco |
| `DB_USER` | Usuário do banco |
| `DB_PASSWORD` | Senha do banco |
| `JWT_SECRET` | Chave secreta JWT (produção) |
| `NEW_RELIC_LICENSE_KEY` | License key do New Relic (APM) |
| `SONAR_TOKEN` | Token do SonarCloud |

### GitHub Variables (opcional)

| Variável | Default |
|---|---|
| `K8S_NAMESPACE` | `fiap-tech-challenge` |
| `JWT_EXPIRES_IN` | `24h` |
| `NEW_RELIC_APP_NAME` | `fiap-web-dev` |

### Manifests Kubernetes (`k8s/`)

| Arquivo | Objeto |
|---|---|
| `01-migrate-job.yaml` | `Job` que roda `sequelize-cli db:migrate` + `db:seed:all` |
| `02-deployment.yaml` | `Deployment` (2 réplicas, probes `/health`, resources, runAsNonRoot) |
| `03-service.yaml` | `Service` tipo `LoadBalancer` (NLB AWS) |
| `04-hpa.yaml` | `HorizontalPodAutoscaler` (CPU 70% / mem 80%, 2–6 réplicas) |

`ConfigMap` (`fiap-soat-tech-challenge-app-config`) e `Secret` (`fiap-soat-tech-challenge-app-secret`) são criados no pipeline a partir dos GitHub Secrets — nunca commitados.

A placeholder `${IMAGE_URI}` nos manifests é substituída via `envsubst` no job de deploy.

## Arquitetura

O projeto segue Clean Architecture com 4 camadas:

```
src/
├── domain/          # Entidades e interfaces de repositório
├── application/     # Use cases e serviços de aplicação
├── infrastructure/  # Banco de dados (Sequelize), web (Express), repositórios
└── interface/       # Controllers, middleware, presenters
```

### Diagramas

- [`docs/application-components.mmd`](docs/application-components.mmd) — Componentes (nuvem, APIs, banco, monitoramento)
- [`docs/sequence-auth-and-os.mmd`](docs/sequence-auth-and-os.mmd) — Sequência (autenticação CPF + abertura de OS)
- [`docs/er-diagram.mmd`](docs/er-diagram.mmd) — Modelo Entidade-Relacionamento do banco
- [`docs/deploy-flux.mmd`](docs/deploy-flux.mmd) — Fluxo de CI/CD
- [`docs/provisioned-infrastructure.mmd`](docs/provisioned-infrastructure.mmd) — Infra provisionada
- [`docs/RFC-001.md`](docs/RFC-001.md) / [`docs/RFC-002.md`](docs/RFC-002.md) — RFCs
- [`docs/ADR-001.md`](docs/ADR-001.md) — ADR PostgreSQL
- [`docs/observability-nrql.md`](docs/observability-nrql.md) — NRQL para dashboard e alertas no New Relic
- [`docs/postman-collection.json`](docs/postman-collection.json) — Coleção Postman v2.1 (importar no Postman/Insomnia)

### Endpoints

| Método | Path | Descrição | Auth |
|---|---|---|---|
| `POST` | `/auth/login` | Autenticação (admin via email/senha; customer via CPF na Lambda) | — |
| `GET` | `/health` | Health check completo (processo + DB) | — |
| `GET` | `/health/live` | Liveness probe (processo apenas, usado pelo K8s) | — |
| `GET` | `/health/ready` | Readiness probe (DB conectado, usado pelo K8s) | — |
| `GET/POST/PUT/DELETE` | `/admin/users` | CRUD usuários | Admin |
| `GET/POST/PUT/DELETE` | `/admin/vehicles` | CRUD veículos | Admin |
| `GET/POST/PUT/DELETE` | `/admin/parts` | CRUD peças | Admin |
| `GET/POST/PUT/DELETE` | `/admin/services` | CRUD serviços | Admin |
| `GET/POST/PUT/DELETE` | `/admin/service-orders` | CRUD ordens de serviço | Admin |
| `GET` | `/customer/service-orders` | Ordens do cliente | Customer |
