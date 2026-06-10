# Repo: gitops

## What this repo is
The GitOps source of truth for everything that runs INSIDE the AKS cluster:
the Strimzi operator and an Apache Kafka (KRaft) cluster. Argo CD watches this
repo and syncs it into the cluster. Test/sandbox only, NOT production.

## Companion repo
The cluster itself (AKS, network, Argo CD install) is provisioned by a SEPARATE
repo: `infra-terraform`. Nothing here creates Azure resources. Bring the cluster
up there first; Argo CD is already installed by the time this repo is used.

## How it flows
1. `infra-terraform` provisions AKS and installs Argo CD.
2. `kubectl apply -f bootstrap/root-app.yaml` registers the app-of-apps.
3. Argo CD syncs `apps/` -> Strimzi operator (wave 0), then Kafka cluster (wave 1).

## Key decisions / constraints
- Kafka runs in **KRaft mode only**. ZooKeeper was removed in Kafka 4.0 and
  Strimzi dropped ZooKeeper support in 0.46. Use `Kafka` + `KafkaNodePool` CRs.
- Default topology = 1 dual-role (broker+controller) node, replication factors 1
  (minimal test). Scale replicas to 3 and raise replication factors for realism.
- Public Strimzi images from quay.io. No private registry.

## Conventions
- One Argo CD Application per concern under `apps/`.
- Raw Strimzi/K8s manifests under `manifests/`.
- Use sync waves to order CRDs before custom resources.
- No secrets committed. Use SealedSecrets / external secret stores if needed.

## Repo layout
- `bootstrap/root-app.yaml`   app-of-apps root Application (apply once)
- `apps/`                     Argo CD Applications (Strimzi operator, Kafka cluster)
- `manifests/kafka/`          KafkaNodePool + Kafka custom resources

## TODO for Claude Code
- [x] Replace <ORG> in bootstrap/root-app.yaml and apps/kafka-cluster.yaml. (Franta89)
- [ ] Confirm latest Strimzi chart version in apps/strimzi-operator.yaml.
- [ ] Confirm the Kafka `version` and `metadataVersion` in manifests/kafka/kafka.yaml.
