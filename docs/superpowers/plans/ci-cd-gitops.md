# CI/CD GitOps Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan step-by-step. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Pipeline GitHub Actions com Trivy (bloqueio HIGH/CRITICAL), push para Artifact Registry e deploy GitOps via ArgoCD no GKE.

**Architecture:** Monorepo com manifests em `k8s/`. CI builda imagem, escaneia com Trivy, publica no AR e, na `main`, atualiza tag no Deployment para o ArgoCD sincronizar.

**Tech Stack:** GitHub Actions, Trivy, GCP Artifact Registry, ArgoCD, GKE, Kubernetes

---

### Task 1: Manifests Kubernetes e App of Apps

**Files:**
- Create: `k8s/namespace.yaml`
- Create: `k8s/deployment.yaml`
- Create: `k8s/service.yaml`
- Create: `deploy/bootstrap/platform-apps.yaml`
- Create: `deploy/applications/desafio-alice.yaml`

- [ ] Criar manifests K8s e Applications ArgoCD (App of Apps)

### Task 2: Pipeline GitHub Actions

**Files:**
- Create: `.github/workflows/ci-cd.yaml`

- [ ] Job `test`: pipenv + pytest
- [ ] Job `build-scan-push`: auth WIF, build, Trivy HIGH/CRITICAL, push AR
- [ ] Job `gitops`: só `main`, atualiza imagem em `k8s/deployment.yaml`, commit `[skip ci]`

### Task 3: Documentação

**Files:**
- Modify: `README.md`

- [ ] Documentar secrets/variables GitHub, bootstrap ArgoCD e fluxo do pipeline
