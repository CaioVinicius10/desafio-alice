# desafio-alice

API Flask com CI/CD via GitHub Actions, scan de segurança com Trivy e deploy GitOps no GKE via ArgoCD.

## Estrutura GitOps

```
deploy/
  bootstrap/
    platform-apps.yaml       # Application pai (bootstrap único)
  applications/
    desafio-alice.yaml         # Application filha (gerenciada pelo Git)
k8s/
  namespace.yaml
  deployment.yaml
  service.yaml
```

## CI/CD

### Fluxo

1. **Todo push** — testes (`pytest`), build da imagem Docker, scan Trivy (reporta HIGH/CRITICAL), push para Artifact Registry
2. **Branch `main`** — atualiza a tag da imagem em `k8s/deployment.yaml`; ArgoCD sincroniza no GKE

### Configuração no GitHub

Crie em **Settings → Secrets and variables → Actions → Secrets**:

| Secret | O que colocar | Exemplo |
|--------|---------------|---------|
| `GCP_PROJECT_ID` | ID do projeto GCP | `meu-projeto-gcp` |
| `GCP_REGION` | Região do Artifact Registry | `southamerica-east1` |
| `GCP_ARTIFACT_REGISTRY` | Nome do repositório Docker no AR | `desafio-alice` |
| `GCP_SA_KEY` | JSON completo da service account | conteúdo do `.json` |

A service account precisa da role `roles/artifactregistry.writer` no repositório de imagens.

#### Como gerar o `GCP_SA_KEY`

```bash
# 1. Criar a service account (se ainda não existir)
gcloud iam service-accounts create github-actions \
  --display-name="GitHub Actions CI/CD"

# 2. Conceder permissão de push no Artifact Registry
gcloud projects add-iam-policy-binding SEU_PROJECT_ID \
  --member="serviceAccount:github-actions@SEU_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

# 3. Gerar a chave JSON
gcloud iam service-accounts keys create github-actions-key.json \
  --iam-account=github-actions@SEU_PROJECT_ID.iam.gserviceaccount.com
```

Copie **todo o conteúdo** do arquivo `github-actions-key.json` e cole no secret `GCP_SA_KEY` do GitHub.

> **Atenção:** chaves JSON são credenciais de longa duração. Rotacione periodicamente e nunca commite o arquivo no repositório.

### Trivy

O pipeline **reporta** vulnerabilidades **HIGH** e **CRITICAL** no log, mas **não bloqueia** o deploy:

```bash
trivy image --severity HIGH,CRITICAL --exit-code 0 --ignore-unfixed <imagem>
```

Para voltar a bloquear o pipeline, altere `exit-code` para `'1'` em `.github/workflows/ci-cd.yaml`.

## GitOps (ArgoCD — App of Apps)

O deploy usa o padrão **App of Apps**: um Application pai gerencia Applications filhas a partir do Git.

```
platform-apps (bootstrap 1x)
  └── deploy/applications/desafio-alice.yaml
        └── k8s/ (Deployment, Service, Namespace)
```

### Bootstrap (uma única vez por cluster)

```bash
kubectl apply -f deploy/bootstrap/platform-apps.yaml
```

Após isso, o ArgoCD passa a:

1. Sincronizar `deploy/applications/` e criar/atualizar a Application `desafio-alice`
2. Sincronizar `k8s/` e aplicar os manifests no GKE

### Adicionar novas aplicações

Crie um novo arquivo em `deploy/applications/` apontando para a pasta de manifests da app. O `platform-apps` detecta e aplica automaticamente — sem `kubectl` adicional.

### Placeholders da imagem

Antes do primeiro pipeline na `main`, substitua em `k8s/deployment.yaml`:

- `REGION` → região do Artifact Registry
- `PROJECT_ID` → ID do projeto GCP
- `ARTIFACT_REGISTRY` → nome do repositório

Depois disso, o CI atualiza a imagem automaticamente a cada merge na `main`.

## Local

```bash
docker build -t desafio-alice .
docker run -p 8080:8080 desafio-alice
```

Aplicação disponível em `http://localhost:8080/api/ping`.
