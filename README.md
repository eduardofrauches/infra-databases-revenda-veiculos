# infra-databases-revenda-veiculos

Infraestrutura de banco de dados compartilhada pela plataforma de
revenda de veiculos: `docker-compose.yml` (execucao local) e manifests
Kubernetes (Kustomize).

## O que este repositorio e

Os dois bancos PostgreSQL usados pelos microsservicos da plataforma:

| StatefulSet | Banco | Usado por |
|---|---|---|
| `postgres-core` | `veiculos_core_db` | [sistema-principal-veiculos](https://github.com/eduardofrauches/sistema-principal-veiculos) |
| `postgres-vendas` | `veiculos_vendas_db` | [servico-vendas-veiculos](https://github.com/eduardofrauches/servico-vendas-veiculos) |

Cada banco e um `StatefulSet` com 1 replica e um `PersistentVolumeClaim`
(1Gi) proprio, alem de um `Service` (ClusterIP), um `ConfigMap` (dados
nao sensiveis, ex.: nome do banco) e um `Secret` (credenciais).

## Por que este repositorio e separado dos dois servicos de aplicacao

Os bancos **nao pertencem** a nenhum dos dois microsservicos
individualmente — sao infraestrutura compartilhada que os dois
consomem, cada um so o seu proprio banco (bancos fisicamente separados,
nunca compartilhados entre os dois servicos). Colocar os manifests
dentro do repositorio de um dos servicos criaria uma dependencia
errada (o outro servico teria que depender do repositorio "errado"
para funcionar). Por isso ficam em um terceiro repositorio, dedicado
so a essa infraestrutura.

## Estrutura

```
.
├── docker-compose.yml                <- os dois Postgres para execucao local (sem Kubernetes)
├── postgres-core-statefulset.yaml    <- StatefulSet + PVC do banco do sistema-principal-veiculos
├── postgres-core-service.yaml        <- Service ClusterIP "postgres-core"
├── postgres-core-configmap.yaml      <- POSTGRES_DB (nao sensivel)
├── postgres-core-secret.yaml         <- POSTGRES_USER / POSTGRES_PASSWORD (placeholder)
├── postgres-vendas-statefulset.yaml  <- StatefulSet + PVC do banco do servico-vendas-veiculos
├── postgres-vendas-service.yaml      <- Service ClusterIP "postgres-vendas"
├── postgres-vendas-configmap.yaml    <- POSTGRES_DB (nao sensivel)
├── postgres-vendas-secret.yaml       <- POSTGRES_USER / POSTGRES_PASSWORD (placeholder)
└── kustomization.yaml                <- agrega todos os recursos acima
```

## Instruções de Execução

### Pré-requisitos
- Docker e Docker Compose instalados e rodando

### 1. Clonar o repositório
```bash
git clone https://github.com/eduardofrauches/infra-databases-revenda-veiculos.git
cd infra-databases-revenda-veiculos
```

### 2. Subir a infraestrutura local
```bash
docker compose up -d
```

Isso sobe os dois bancos PostgreSQL:

| Container | Banco | Porta no host | Usuário / senha |
|---|---|---|---|
| `revenda-postgres-core` | `veiculos_core_db` | `5432` | `postgres` / `postgres` |
| `revenda-postgres-vendas` | `veiculos_vendas_db` | `5433` | `postgres` / `postgres` |

Esses valores batem com o `application.yml` de cada serviço. Para
derrubar: `docker compose down` (adicione `-v` para apagar também os
dados).

## Como aplicar no Kubernetes

Com um cluster Kubernetes acessivel (ex.: Minikube ja rodando):

```bash
kubectl apply -k .
```

Aguarde os dois StatefulSets ficarem prontos antes de subir os
servicos de aplicacao (eles dependem do banco no startup):

```bash
kubectl rollout status statefulset/postgres-core --timeout=120s
kubectl rollout status statefulset/postgres-vendas --timeout=120s
```

## ⚠️ Aviso sobre o Secret

Os valores em `postgres-core-secret.yaml` e `postgres-vendas-secret.yaml`
(`POSTGRES_USER: postgres`, `POSTGRES_PASSWORD: postgres`) sao
**placeholders para uso local (Minikube/desenvolvimento)**. **Nunca**
use esses valores — nem esse formato de `Secret` puro versionado no
Git — em produção. Em um ambiente real, isso deveria vir de um cofre
de segredos (Sealed Secrets, Vault, External Secrets Operator, etc.),
nunca commitado em texto claro.

## Repositorios que dependem desta infraestrutura

Os dois microsservicos abaixo precisam que os manifests deste
repositorio estejam aplicados (bancos `Running`/`Ready`) **antes** de
subir seus proprios manifests Kubernetes, ja que ambos dependem do seu
respectivo banco no startup:

- [sistema-principal-veiculos](https://github.com/eduardofrauches/sistema-principal-veiculos) — cadastro e edicao de veiculos (usa `postgres-core`)
- [servico-vendas-veiculos](https://github.com/eduardofrauches/servico-vendas-veiculos) — listagem, venda e webhook de pagamento (usa `postgres-vendas`)
