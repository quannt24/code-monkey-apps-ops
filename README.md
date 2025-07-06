# apps-ops

Ops repo for managing list of ArgoCD applications
Based on: https://github.com/argoproj/argo-helm/tree/main/charts/argocd-apps

## Add a new ArgoCD application to the application list

- Prepare secrets, see [Secrets](#secrets).
- Add the new app to `values.yaml`, for example:

  ```yaml
  monkey-app:
    namespace: argocd
    finalizers:
      - resources-finalizer.argocd.argoproj.io
    project: default
    sources:
      - repoURL: git@github.com/monkey-ops.git
        targetRevision: main
        path: helm-chart
        helm:
          releaseName: monkey
          # valueFiles:
          # - values.yaml
          # - {env}-values.yaml
    destination:
      server: https://kubernetes.default.svc
      # namespace: monkey
    syncPolicy:
      automated:
        prune: false
        selfHeal: false
      syncOptions:
        - CreateNamespace=true
  ```

- Add the new app to `{env}-values.yaml` to setup usage of environment specific values, for example:

  ```yaml
  monkey-app:
    sources:
      - repoURL: git@github.com/monkey-ops.git
        targetRevision: trunk
        path: helm-chart
        helm:
          valueFiles:
            - values.yaml
            - {env}.values.yaml
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
  ```

## Secrets

Managed secrets:

- Ops repo secrets: For pulling ops Git repos
- Helm repo secrets: For pulling helm charts
- Image pull secrets: For pulling images in pods

### Create docker config secret for pulling images - Informative

```bash
## Login with your username and password
docker login ghcr.io
## Docker config file will be created/updated at '$HOME/.docker/config.json'

## NOTE:
## - Can repeat this step for multiple registries
## - Double check the config.json, remove unnecessary registries

kubectl create secret generic imagepullsecret \
    --namespace=monkey \
    --from-file=.dockerconfigjson=$HOME/.docker/config.json \
    --type=kubernetes.io/dockerconfigjson \
    --dry-run=client \
    -o yaml > image-pull-secret.yaml

kubectl apply -f image-pull-secret.yaml
```

### Encrypt secret values files

```bash
sops --config ./{env}.sops.yaml -e secret/{env}-secret.values.dec.yaml > helm-chart/{env}-secret.values.yaml
## Test templating
cd helm-chart
helm secrets template monkey-apps . -f values.yaml -f {env}.values.yaml -f {env}-secret.values.yaml
```

Note: Encrypted files should not be edited manually.

## Install apps-app

```bash
kubectl apply -f {env}-apps-app.yaml
```

## Add new ArgoCD users

- Update `argocd-rbac-cm.yml` for required roles and groups
- Update `argocd-cm.yml` (argo-cd chart) to add `accounts.<username>` entries
- Apply the changes
- Set user password:

  ```bash
  argocd login <server_ip>:<port>
  # Example
  # argocd login argocd.local:443

  argocd account list
  # if you are managing users as the admin user,
  # <current-user-password> should be the current admin password.
  argocd account update-password \
    --account <name> \
    --current-password <current-user-password> \
    --new-password <new-user-password>
  ```
