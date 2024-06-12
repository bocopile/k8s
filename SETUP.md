```bash

helm repo add containeroo https://charts.containeroo.ch
helm upgrade --install local-path-provisioner containeroo/local-path-provisioner --version 0.0.26 --create-namespace --namespace extra -f helm/values-local-path-provisioner.yaml


```
