# gitops

Argo CD source of truth for Strimzi + Kafka (KRaft) running on AKS.
The cluster is provisioned by the companion **infra-terraform** repo.

## Order
1. Bring up the cluster + Argo CD via **infra-terraform**.
2. Replace `<ORG>` in `bootstrap/root-app.yaml` and `apps/kafka-cluster.yaml`,
   commit, and push.
3. `kubectl apply -f bootstrap/root-app.yaml`
4. Watch: `kubectl -n kafka get pods,kafka,kafkanodepool`

## Argo CD UI
`kubectl -n argocd port-forward svc/argocd-server 8080:443`
Initial admin password:
`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`

## Note
Strimzi chart version and the Kafka/metadataVersion fields are marked TODO —
verify before syncing. See CLAUDE.md.
