# CI/CD com Trivy e GitOps (ArgoCD + GKE)

**Data:** 2026-05-28  
**Status:** Aprovado

## Contexto

API Flask simples (`desafio-alice`) com Dockerfile existente. Infraestrutura GCP já provisionada: GKE, Artifact Registry e ArgoCD. Repositório no GitHub.

## Objetivo

Automatizar build, scan de segurança e deploy GitOps:

- Todo push: testes, build da imagem, scan Trivy da imagem Docker
- Pipeline falha se Trivy encontrar vulnerabilidades **HIGH** ou **CRITICAL**
- Push da imagem para Artifact Registry após scan aprovado
- Apenas branch `main`: atualiza manifest Kubernetes no monorepo; ArgoCD sincroniza no GKE

## Decisões

| Item | Decisão |
|------|---------|
| Infra | GKE + Artifact Registry + ArgoCD (existente) |
| GitOps | Monorepo, pasta `k8s/` |
| CI | GitHub Actions |
| Trigger | Build + scan em todo push; deploy só na `main` |
| Trivy | Scan somente da imagem Docker final |
| Bloqueio | `--severity HIGH,CRITICAL --exit-code 1` |
| Tag da imagem | `{region}-docker.pkg.dev/{project}/{repo}/desafio-alice:{GITHUB_SHA}` |
| Auth GCP | Service account JSON (`GCP_SA_KEY`) via `docker/login-action` |

## Arquitetura

```
Push → GitHub Actions
  → pytest
  → docker build
  → trivy image (HIGH/CRITICAL → fail)
  → push Artifact Registry
  → [main] commit nova tag em k8s/deployment.yaml
  → ArgoCD: platform-apps → desafio-alice Application → sync k8s/ → GKE
```

## Componentes

### GitHub Actions (`.github/workflows/ci-cd.yaml`)

Jobs:

1. **test** — instala dependências com pipenv e executa pytest
2. **build-scan-push** — autentica no GCP, build, scan Trivy, push da imagem
3. **gitops** — condicional à `main`; atualiza imagem no Deployment e commit com `[skip ci]`

### GitOps ArgoCD (App of Apps)

- `deploy/bootstrap/platform-apps.yaml` — Application pai; bootstrap único via `kubectl apply`
- `deploy/applications/desafio-alice.yaml` — Application filha; gerenciada automaticamente pelo pai
- `k8s/` — manifests da aplicação sincronizados pela Application filha

### Kubernetes (`k8s/`)

- `namespace.yaml` — namespace `desafio-alice`
- `deployment.yaml` — 2 réplicas, probes em `/api/ping`, porta 8080
- `service.yaml` — ClusterIP 8080

### Secrets/Variables GitHub

| Nome | Tipo | Uso |
|------|------|-----|
| `GCP_PROJECT_ID` | Variable | ID do projeto GCP |
| `GCP_REGION` | Variable | Região (ex.: `southamerica-east1`) |
| `GCP_ARTIFACT_REGISTRY` | Variable | Nome do repositório no Artifact Registry |
| `GCP_SA_KEY` | Secret | JSON completo da service account |

## Fora de escopo

- Provisionamento de GKE, Artifact Registry ou ArgoCD
- Scan de filesystem/dependências Python (somente imagem)
- Ambientes múltiplos (dev/staging) — apenas produção via `main`
