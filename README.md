# gitops

Argo CD source of truth for Strimzi + Kafka (KRaft) running on AKS.
The cluster is provisioned by the companion **infra-terraform** repo.

## Order
1. Bring up the cluster + Argo CD via **infra-terraform**.
2. `kubectl apply -f bootstrap/root-app.yaml`
3. Watch: `kubectl -n kafka get pods,kafka,kafkanodepool`

## Argo CD UI
`kubectl -n argocd port-forward svc/argocd-server 8080:443`
Initial admin password:
`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`

## Daily Dose of Tech (news app)

Public site: <https://dailydoseoftech.org>. A daily AI-curated news digest across
five areas (Cloud Computing, AI Development, IT Security, Financial Markets, World
News). Deployed in sync wave 5 under `manifests/news-digest/`.

News is gathered in real time from **authoritative RSS/Atom feeds** (keyless, $0,
no third-party API) and summarised by Azure AI Foundry (GPT-4.1-mini) via Azure
Workload Identity. The daily CronJob runs at 06:30 CET.

> Migrated off NewsAPI (June 2026): the free tier was 24h-delayed and its terms
> forbade production/staging use. RSS is real-time, keyless, and compliant.

Operational details — RSS source lists, look-back windows, AI selection logic,
manual reruns, and troubleshooting — are in **[docs/runbook.md](docs/runbook.md)**.

## Note
Strimzi chart version and the Kafka/metadataVersion fields are marked TODO —
verify before syncing. See CLAUDE.md.
