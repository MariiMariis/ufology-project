# 🛸 Operação Ufology — Documentação do Projeto

Infraestrutura Kubernetes para a divisão **Ufology Investigation Unit**.

---

## Missão 1 — Banco de Dados da Operação

**Objetivo:** Implantar o banco de dados PostgreSQL oficial da operação no cluster Kubernetes.

### Recursos Criados

| Recurso | Nome | Detalhes |
|---|---|---|
| **Namespace** | `ufology` | Isolamento exclusivo da operação |
| **Deployment** | `ufodb` | 1 réplica, imagem `leogloriainfnet/ufodb:1.0-win` |
| **Service** | `ufodb` | ClusterIP, porta `5432` (acesso interno) |

### Variáveis de Ambiente

| Variável | Valor |
|---|---|
| `POSTGRES_USER` | `postgres` |
| `POSTGRES_PASSWORD` | `devops2025!` |
| `POSTGRES_DB` | `ufology` |

### Arquivo de Manifesto

- [`ufology.yaml`](./ufology.yaml) — Contém namespace, deployment e service

### Comandos Úteis

```bash
# Aplicar os manifestos
kubectl apply -f ufology.yaml

# Verificar todos os recursos no namespace
kubectl get all -n ufology

# Ver logs do pod do banco
kubectl logs -n ufology -l app=ufodb

# Testar conexão ao banco (via port-forward)
kubectl port-forward -n ufology svc/ufodb 5432:5432
```

### Conexão Interna (DNS do Cluster)

Outros serviços dentro do cluster podem se conectar ao banco usando:

```
Host: ufodb.ufology.svc.cluster.local
Porta: 5432
Usuário: postgres
Senha: devops2025!
Database: ufology
```

---

## Missão 2 — Sistema de Cache da Operação

**Objetivo:** Adicionar uma camada de cache Redis em memória para otimizar consultas frequentes ao banco.

### Recursos Criados

| Recurso | Nome | Detalhes |
|---|---|---|
| **Deployment** | `ufo-cache` | 1 réplica, imagem `redis:7-alpine` |
| **Service** | `ufo-cache` | ClusterIP, porta `6379` (acesso interno) |

### Conexão Interna (DNS do Cluster)

```
Host: ufo-cache.ufology.svc.cluster.local
Porta: 6379
```

### Comandos Úteis

```bash
# Ver logs do Redis
kubectl logs -n ufology -l app=ufo-cache

# Testar conexão ao Redis (via port-forward)
kubectl port-forward -n ufology svc/ufo-cache 6379:6379
```

---

## Missão 3 — Dockerização da Aplicação UfoTracker

**Objetivo:** Containerizar a aplicação Spring Boot do repositório GitHub e publicar a imagem no Docker Hub.

### Repositório Fonte

- https://github.com/leoinfnet/devops2026-ufoTracker/

### Dockerfile

Localizado em `devops2026-ufoTracker/Dockerfile` — Multi-stage build:

| Estágio | Base | Função |
|---|---|---|
| **Build** | `eclipse-temurin:21-jdk-alpine` | Compila o projeto com Maven |
| **Runtime** | `eclipse-temurin:21-jre-alpine` | Executa o JAR (imagem leve) |

### Comandos Utilizados

```bash
# Build da imagem
docker build -t mariimariis/ufotracker:1.0 .

# Push para o Docker Hub
docker push mariimariis/ufotracker:1.0
```

### Link Público da Imagem

🔗 https://hub.docker.com/r/mariimariis/ufotracker

---

## Missão 4 — Implantação da Aplicação UfoTracker

**Objetivo:** Implantar a aplicação Spring Boot no cluster, conectando-a ao PostgreSQL e Redis existentes, usando ConfigMap e Secret para dados sensíveis.

### Recursos Criados

| Recurso | Nome | Detalhes |
|---|---|---|
| **ConfigMap** | `app-config` | `DB_NAME=ufology` |
| **Secret** | `db-secret` | `DB_PASSWORD=devops2025!` (base64) |
| **Deployment** | `ufotracker` | 2 réplicas, imagem `mariimariis/ufotracker:1.0`, porta `8080` |
| **Service** | `ufotracker` | ClusterIP, porta `8080` (acesso interno) |

### Variáveis de Ambiente do Deployment

| Variável | Origem | Valor / Referência |
|---|---|---|
| `DB_NAME` | ConfigMap `app-config` | `ufology` |
| `SPRING_DATASOURCE_URL` | Interpolação | `jdbc:postgresql://ufodb:5432/$(DB_NAME)` |
| `POSTGRES_USERNAME` | Literal | `postgres` |
| `POSTGRES_PASSWORD` | Secret `db-secret` | `devops2025!` |
| `SPRING_DATA_REDIS_HOST` | Literal | `ufo-cache` |

### Conexão Interna (DNS do Cluster)

```
Host: ufotracker.ufology.svc.cluster.local
Porta: 8080
```

### Comandos Úteis

```bash
# Aplicar todos os manifestos
kubectl apply -f ufology.yaml

# Verificar todos os recursos no namespace
kubectl get all,configmap,secret -n ufology

# Verificar que ufotracker tem 2 pods em execução
kubectl get pods -n ufology -l app=ufotracker

# Ver logs da aplicação
kubectl logs -n ufology -l app=ufotracker

# Testar conexão à aplicação (via port-forward)
kubectl port-forward -n ufology svc/ufotracker 8080:8080
```

---

## Resumo dos Componentes no Namespace `ufology`

| Componente | Tipo | Réplicas | Porta |
|---|---|---|---|
| PostgreSQL (`ufodb`) | Deployment + Service | 1 | 5432 |
| Redis (`ufo-cache`) | Deployment + Service | 1 | 6379 |
| UfoTracker (`ufotracker`) | Deployment + Service | 2 | 8080 |
| Configuração (`app-config`) | ConfigMap | — | — |
| Credenciais (`db-secret`) | Secret | — | — |

Essa documentação foi criada para atender as exigências do Assessment 