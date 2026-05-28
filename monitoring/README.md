# Monitoring — Prometheus + Grafana

Stack de observabilidade via [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack), instalada com GitOps (ArgoCD).

## Estrutura

```
monitoring/
  kube-prometheus-stack/
    values.yaml              # Helm values (chart externo)
  manifests/
    servicemonitor-desafio-alice.yaml   # scrape /metrics (opcional)
deploy/applications/
  monitoring.yaml            # Helm chart via multi-source
  monitoring-manifests.yaml  # CRs pós-instalação (sync-wave 1)
```

## Deploy (GitOps)

Após o bootstrap do `platform-apps`, o ArgoCD cria:

1. **monitoring** (wave 0) — namespace `monitoring` com Prometheus, Grafana, Alertmanager
2. **monitoring-manifests** (wave 1) — ServiceMonitor da app (requer CRDs do operator)

## Acesso ao Grafana

```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
```

Abra `http://localhost:3000` — usuário/senha padrão definidos em `values.yaml` (`admin` / `changeme`).

> Em produção, use `grafana.admin.existingSecret` e troque a senha.

## Dashboards úteis

No Grafana, busque por:

- **Kubernetes / Compute Resources / Namespace (Pods)** — filtre `desafio-alice`
- **Kubernetes / Compute Resources / Pod** — CPU/memória dos pods da app

## Alertas da aplicação

Regras em `values.yaml` → `additionalPrometheusRulesMap.desafio-alice`:

| Alerta | Condição |
|--------|----------|
| `DesafioAliceUnavailable` | < 1 réplica disponível por 2 min |
| `DesafioAliceHighRestartRate` | > 3 restarts em 15 min |
| `DesafioAliceHPAMaxedOut` | HPA no maxReplicas por 10 min |

## ServiceMonitor (opcional)

O arquivo `manifests/servicemonitor-desafio-alice.yaml` aponta para `/metrics`. A app Flask atual **não expõe** esse endpoint — remova o Application `monitoring-manifests` ou adicione um exporter Prometheus na app para ativar scrape de métricas de aplicação.

Métricas de infra (CPU, memória, restarts, HPA) já funcionam via kube-state-metrics e cAdvisor sem ServiceMonitor.

## Requisitos

- ArgoCD **2.6+** (multi-source para `$values/...`)
- Cluster GKE com recursos suficientes (~2 Gi RAM livres para a stack)
