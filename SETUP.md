```bash

helm repo add containeroo https://charts.containeroo.ch
helm upgrade --install local-path-provisioner containeroo/local-path-provisioner --version 0.0.26 --create-namespace --namespace extra -f helm/values-local-path-provisioner.yaml

helm repo add stevehipwell https://stevehipwell.github.io/helm-charts/
helm upgrade --install nexus-v3 stevehipwell/nexus3 --create-namespace --namespace nexus --version 4.42.1 -f helm/values-nexus.yaml

```
