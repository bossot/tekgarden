# Plantilles FluxCD

Components GitOps reutilitzables per Kubernetes amb FluxCD. Tots els exemples estan anonimitzats per compartir.

## Índex

- [HelmRelease amb health checks](#helmrelease-amb-health-checks)
- [Kustomization](#kustomization)
- [ExternalSecret amb 1Password](#externalsecret-amb-1password)
- [Estàndards](#estàndards)

## HelmRelease amb health checks

Exemple de HelmRelease d'un servei web amb sourceRef, health checks, dependsOn i values personalitzats.

```yaml
# helmrelease.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: [APP_NAME]
  namespace: [NAMESPACE]
spec:
  interval: 15m
  timeout: 5m
  releaseName: [APP_NAME]

  # Origen del chart
  chartRef:
    kind: OCIRepository
    name: [APP_NAME]-chart
    namespace: flux-system

  # Dependències: no desplegar fins que aquests recursos estiguin ready
  dependsOn:
    - name: redis
      namespace: [NAMESPACE]
    - name: postgres
      namespace: [NAMESPACE]
    - name: cert-manager
      namespace: cert-manager
    - name: ingress-nginx
      namespace: ingress-nginx

  # Valors específics de l'aplicació
  values:
    image:
      repository: [REGISTRY_URL]/[IMAGE_NAME]
      tag: [IMAGE_TAG]

    replicaCount: 2

    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi

    ingress:
      enabled: true
      className: nginx
      hosts:
        - host: [APP_NAME].[DOMAIN]
          paths:
            - path: /
              pathType: Prefix

    env:
      - name: DB_HOST
        value: "[APP_NAME]-postgres"
      - name: DB_PORT
        value: "5432"
      - name: REDIS_HOST
        value: "[APP_NAME]-redis"
      - name: LOG_LEVEL
        value: "info"

    # Health checks del chart
    probes:
      liveness:
        path: /health
        initialDelaySeconds: 30
        periodSeconds: 10
      readiness:
        path: /ready
        initialDelaySeconds: 5
        periodSeconds: 5
      startup:
        path: /startup
        initialDelaySeconds: 0
        periodSeconds: 10
        failureThreshold: 30
```

## Kustomization

Exemple de Kustomization que referència un HelmRelease amb prune i valors de configuració externa.

```yaml
# kustomization.yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: [APP_NAME]
  namespace: flux-system
spec:
  interval: 10m
  timeout: 5m
  retryInterval: 2m
  path: ./apps/[NAMESPACE]/[APP_NAME]/overlays/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system

  # Dependències a nivell de Kustomization
  dependsOn:
    - name: infrastructure
      namespace: flux-system
    - name: monitoring
      namespace: flux-system

  # Aplicar validacions abans de desplegar
  validation: client

  # Post-build per substituir placeholders
  postBuild:
    substitute:
      APP_NAME: [APP_NAME]
      NAMESPACE: [NAMESPACE]
      DOMAIN: [DOMAIN]
      IMAGE_TAG: [IMAGE_TAG]
    substituteFrom:
      - kind: ConfigMap
        name: cluster-config
      - kind: Secret
        name: cluster-secrets

  # Notificacions de reconcilació
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2
      kind: HelmRelease
      name: [APP_NAME]
      namespace: [NAMESPACE]
```

## ExternalSecret amb 1Password

Exemple d'ExternalSecret que obté secrets d'1Password Connect i els exposa com a Secret de Kubernetes.

```yaml
# externalsecret.yaml
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: [APP_NAME]-secrets
  namespace: [NAMESPACE]
spec:
  refreshInterval: 1h

  # Backend: 1Password Connect
  secretStoreRef:
    kind: ClusterSecretStore
    name: onepassword-connect

  target:
    name: [APP_NAME]-secrets
    creationPolicy: Owner
    deletionPolicy: Retain

  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: [ONEPASSWORD_VAULT]/[ONEPASSWORD_ITEM]
        property: password

    - secretKey: REDIS_PASSWORD
      remoteRef:
        key: [ONEPASSWORD_VAULT]/[ONEPASSWORD_ITEM]
        property: redis_password

    - secretKey: SECRET_KEY
      remoteRef:
        key: [ONEPASSWORD_VAULT]/[ONEPASSWORD_ITEM]
        property: secret_key

    - secretKey: API_TOKEN
      remoteRef:
        key: [ONEPASSWORD_VAULT]/[ONEPASSWORD_ITEM]
        property: api_token
```

## Estàndards

- **Namespace aïllat**: Cada aplicació al seu propi namespace
- **Labels FluxCD**: Tots els recursos porten els labels estàndard de FluxCD (`app.kubernetes.io/name`, `app.kubernetes.io/instance`)
- **Dependencies**: Sempre explícites amb `dependsOn`, mai confiar en l'ordre d'aplicació
- **Health checks**: Probes de liveness, readiness i startup per tots els serveis
- **Timeouts**: Configurats per evitar reconciliacions infinites (5m per defecte)
- **Pull Request**: Mai desplegar directament a main — tot passa per PR
- **Secrets**: Mai en valors en clar al repositori, sempre via ExternalSecrets amb 1Password
- **Retry**: `retryInterval: 2m` per reintents automàtics en cas d'error
