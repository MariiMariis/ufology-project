# 🛸 Operação Ufology — Documentação do Projeto

[![Maven CI](https://github.com/MariiMariis/ufology-project/actions/workflows/gradle-ci.yml/badge.svg)](https://github.com/MariiMariis/ufology-project/actions/workflows/gradle-ci.yml)

Infraestrutura Kubernetes para a divisão **Ufology Investigation Unit**.

---

## O Papel do Git no Ciclo DevOps e na Entrega Contínua

O **Git** é a base para qualquer pipeline DevOps moderno. Ele atua como o **sistema de controle de versão distribuído** que permite rastrear todas as mudanças no código-fonte de forma precisa e auditável.

No contexto de **Integração Contínua (CI)** e **Entrega Contínua (CD)**, o Git desempenha os seguintes papéis fundamentais:

- **Controle de versão**: Cada commit registra o estado exato do código, permitindo reverter falhas, comparar versões e manter um histórico completo de mudanças.
- **Rastreamento de mudanças**: Através do `git log`, `git diff` e `git blame`, é possível auditar quem fez o quê, quando e por quê.
- **Gatilho de automação**: Eventos do Git (push, pull request, tags) disparam automaticamente pipelines de CI/CD via GitHub Actions, eliminando processos manuais.
- **Colaboração segura**: Através de branches e pull requests, equipes podem trabalhar em paralelo sem conflitos, com revisão de código antes de integrar ao código principal.

O Git não é apenas uma ferramenta de versionamento — é o **elo central** que conecta desenvolvimento, testes automatizados e deploy, garantindo que cada mudança passe por validação antes de chegar à produção.

---

## Importância de Branches e Tags em CI/CD

### Branches

As **branches** permitem o desenvolvimento isolado de funcionalidades sem impactar o código principal. No fluxo de CI/CD:

- **`main`**: Branch de produção — código estável e pronto para deploy. Pushes nesta branch disparam pipelines de build e deploy.
- **`ci/setup`**: Branch dedicada para configuração da infraestrutura de CI/CD, isolando mudanças até estarem prontas.
- **Feature branches** (ex: `test-pr`): Branches temporárias para desenvolver funcionalidades, que são integradas via Pull Request com revisão e testes automáticos.

### Tags

As **tags** marcam pontos específicos no histórico do repositório, funcionando como "snapshots" imutáveis:

- **Versionamento semântico** (ex: `v1.0.0`): Identifica releases oficiais com número de versão.
- **Rastreabilidade**: Permite saber exatamente qual versão do código está em cada ambiente (staging, produção).
- **Automação de releases**: Tags podem disparar workflows específicos para empacotamento e publicação de artefatos.

Branches e tags juntos formam a estrutura que permite **integração contínua** (merge frequente de código testado) e **entrega contínua** (deploy automatizado de versões validadas).

---

## Workflows do GitHub Actions no Processo de CI/CD

Os **workflows** são arquivos YAML que definem pipelines automatizados no GitHub Actions. Eles são o mecanismo que transforma eventos do repositório em ações automatizadas.

### Estrutura de um Workflow

- **Events (`on`)**: Definem quando o workflow é acionado (push, pull_request, tags, manual).
- **Jobs**: Unidades de trabalho que rodam em um runner (máquina virtual). Podem ser executados em paralelo ou em sequência com `needs`.
- **Steps**: Passos individuais dentro de um job — podem executar comandos shell ou usar actions reutilizáveis.

### Workflows neste Projeto

| Workflow | Evento | Função |
|----------|--------|--------|
| `hello.yml` | push | Validação básica — exibe "Hello CI/CD" |
| `tests.yml` | pull_request | Executa testes em PRs antes do merge |
| `gradle-ci.yml` | push (main) | Build Maven + upload do artefato JAR |
| `env-demo.yml` | push | Demonstra uso de variáveis de ambiente |
| `secret-demo.yml` | push | Demonstra uso seguro de secrets |
| `run-monitor.yml` | push | Monitoramento com variáveis multi-nível, GITHUB_TOKEN e diagnósticos |
| `deploy.yml` | push (main) | Pipeline completo: testes → validação → deploy com aprovação manual |

Os workflows garantem que todo código passe por **validação automatizada** antes de ser integrado ou implantado, reduzindo erros humanos e acelerando o ciclo de entregas.

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