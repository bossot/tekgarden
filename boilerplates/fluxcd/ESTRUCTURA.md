# Estructura FluxCD — Pipeline GitOps i exemples avançats

## Índex

- [Pipeline GitOps: diagrama de flux](#pipeline-gitops-diagrama-de-flux)
- [Recursos bàsics: HelmRepository, HelmRelease, Kustomization](#recursos-bàsics-helmrepository-helmrelease-kustomization)
- [Relacions entre recursos](#relacions-entre-recursos)
- [Exemples avançats](#exemples-avançats)
  - [Kustomization amb postBuild.substitute per ConfigMaps dinàmics](#1-kustomization-amb-postbuildsubstitute-per-configmaps-dinàmics)
  - [ExternalSecret amb 1Password Connect](#2-externalsecret-amb-1password-connect)
  - [Kustomization que genera un Secret des d'un fitxer .env](#3-kustomization-que-genera-un-secret-des-dun-fitxer-env)
  - [Patch strategic merge a un Deployment](#4-patch-strategic-merge-a-un-deployment)
  - [ImageUpdateAutomation per auto-update d'imatges](#5-imageupdateautomation-per-auto-update-dimatges)
- [Estructura plana per app (sense overlays)](#estructura-plana-per-app-sense-overlays)

---

## Pipeline GitOps: diagrama de flux

```
┌─────────────────────────────────────────────────────────────────────┐
│                     REPOSITORI GIT (fluxcd)                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  clusters/vega/                                              │  │
│  │  ├── flux-system/        ← config del propi FluxCD          │  │
│  │  ├── infrastructure/     ← dependències base (ingress,      │  │
│  │  │                         cert-manager, CSI drivers)        │  │
│  │  └── apps/               ← aplicacions desplegades          │  │
│  │      └── [NAMESPACE]/                                       │  │
│  │          └── [APP_NAME]/                                     │  │
│  │              ├── kustomization.yaml   ← Kustomization        │  │
│  │              ├── helmrelease.yaml     ← o HelmRelease        │  │
│  │              ├── externalsecret.yaml  ← secrets (opcional)   │  │
│  │              └── configmap.yaml       ← config (opcional)    │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   │ git push
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    FluxCD (al cluster)                              │
│                                                                     │
│  1. GitRepository source     ← poll / webhook per detectar canvis  │
│  2. Kustomization controller ← aplica els manifests llegits        │
│  3. Helm controller          ← processa HelmRelease si n'hi ha     │
│  4. Image automation         ← opcional: auto-update tags          │
│                                                                     │
│  Flux llegeix el path configurat a la Kustomization arrel          │
│  i reconcilia recursivament tot el que troba.                      │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   │ reconcile
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    CLUSTER DESTÍ                                   │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                   │
│  │ Namespace  │  │ HelmRelease│  │  Secrets   │                   │
│  │ [NAMESPACE]│  │ [APP_NAME] │  │ [APP_NAME]-secrets             │
│  └────────────┘  └────────────┘  └────────────┘                   │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                   │
│  │ ConfigMap  │  │ Deployment │  │  Service   │                   │
│  │ [APP_NAME] │  │ [APP_NAME] │  │ [APP_NAME] │                   │
│  └────────────┘  └────────────┘  └────────────┘                   │
└─────────────────────────────────────────────────────────────────────┘
```

### Flux pas a pas

1. **Git push** al repositori fluxcd (branch main)
2. FluxCD detecta el canvi via `GitRepository` source (poll cada 3m o webhook)
3. El `Kustomization` controller llegeix el directori i resol els manifests
4. Per cada `HelmRelease`, el Helm controller gestiona el desplegament del chart
5. Per recursos simples (Deployment, Service), `kustomize` els aplica directament
6. Flux monitoritza l'estat i exposa mètriques d'èxit/error

---

## Recursos bàsics: HelmRepository, HelmRelease, Kustomization

### HelmRepository (o OCIRepository)

Defineix d'on baixar els charts Helm. Pot ser un repositori HTTP o un registre OCI.

```yaml
# helmrepository.yaml — Repositori Helm tradicional
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: [CHART_REPO_NAME]
  namespace: flux-system
spec:
  interval: 30m
  url: https://charts.[DOMAIN]
```

```yaml
# ocirepository.yaml — Chart en registre OCI
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: [APP_NAME]-chart
  namespace: flux-system
spec:
  interval: 30m
  url: oci://[REGISTRY_URL]/charts/[APP_NAME]
  ref:
    tag: [CHART_VERSION]
```

### HelmRelease

Desplega un chart Helm al cluster. És el recurs que fa de *pont* entre la font del chart i els valors concrets.

```yaml
# helmrelease.yaml — Veure README.md per l'exemple complet
```

### Kustomization

Recurs que aplica manifests (YAML directes o renders de kustomize) des d'un path del repositori.

```yaml
# kustomization.yaml — Veure README.md per l'exemple complet
```

---

## Relacions entre recursos

```
HelmRepository / OCIRepository
        │
        │ sourceRef
        ▼
    HelmRelease ─── values ───→ Chart (Deployment, Service, Ingress...)
        │
        │ dependsOn
        ▼
    Altres HelmRelease / Kustomization

GitRepository
        │
        │ sourceRef
        ▼
    Kustomization ─── path ───→ manifests YAML al repositori
        │
        │ dependsOn
        ▼
    Altres Kustomization / HelmRelease
```

**Punts clau:**

- `HelmRelease` **no** conté manifests — delega al chart. Els values són configuració.
- `Kustomization` aplica YAML directes o processa amb kustomize (patches, configMapGenerator, secretGenerator).
- `dependsOn` estableix ordre: espera que [X] estigui ready abans de començar.
- Un HelmRelease pot tenir un `Kustomization` com a pare (si el path conté un HelmRelease YAML), i viceversa.

---

## Exemples avançats

### 1. Kustomization amb `postBuild.substitute` per ConfigMaps dinàmics

Permet substituir placeholders als manifests abans d'aplicar-los. Ideal per ConfigMaps on el contingut depèn de l'entorn.

```yaml
# kustomization.yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: [APP_NAME]-config
  namespace: flux-system
spec:
  interval: 10m
  path: ./apps/[NAMESPACE]/[APP_NAME]/config
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  postBuild:
    substitute:
      APP_NAME: [APP_NAME]
      NAMESPACE: [NAMESPACE]
      DOMAIN: [DOMAIN]
      LOG_LEVEL: info
      DB_HOST: [APP_NAME]-postgres
      DB_PORT: "5432"
      REDIS_HOST: [APP_NAME]-redis
      CACHE_TTL: "3600"
      MAX_UPLOAD_SIZE: "50MB"
      FEATURE_FLAG_NEW_UI: "true"
      SENTRY_DSN: https://[SENTRY_KEY]@sentry.[DOMAIN]/[SENTRY_PROJECT_ID]
    substituteFrom:
      - kind: ConfigMap
        name: cluster-config
      - kind: Secret
        name: [APP_NAME]-secrets
```

```yaml
# config/configmap.yaml — Plantilla amb placeholders
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: [APP_NAME]-config
  namespace: [NAMESPACE]
data:
  app.env: |
    APP_NAME=[APP_NAME]
    LOG_LEVEL=[LOG_LEVEL]
    DB_HOST=[DB_HOST]
    DB_PORT=[DB_PORT]
    REDIS_HOST=[REDIS_HOST]
    CACHE_TTL=[CACHE_TTL]
    MAX_UPLOAD_SIZE=[MAX_UPLOAD_SIZE]
    FEATURE_FLAG_NEW_UI=[FEATURE_FLAG_NEW_UI]
    SENTRY_DSN=[SENTRY_DSN]
```

> **Nota:** `postBuild.substitute` substitueix abans d'aplicar al cluster. Els placeholders que no es trobin a `substitute` ni a `substituteFrom` causaran error.

---

### 2. ExternalSecret amb 1Password Connect

Recupera secrets des d'1Password Connect i els exposa com a Secret de Kubernetes.

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

  secretStoreRef:
    kind: ClusterSecretStore
    name: onepassword-connect

  target:
    name: [APP_NAME]-secrets
    creationPolicy: Owner
    deletionPolicy: Retain
    template:
      type: kubernetes.io/basic-auth  # opcional: Opaque per defecte
      metadata:
        labels:
          app.kubernetes.io/name: [APP_NAME]

  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: [OP_VAULT_ID]/[OP_ITEM_ID]
        property: password

    - secretKey: REDIS_PASSWORD
      remoteRef:
        key: [OP_VAULT_ID]/[OP_ITEM_ID]
        property: redis_password

    - secretKey: SECRET_KEY
      remoteRef:
        key: [OP_VAULT_ID]/[OP_ITEM_ID]
        property: secret_key

    - secretKey: API_TOKEN
      remoteRef:
        key: [OP_VAULT_ID]/[OP_ITEM_ID]
        property: api_token

    - secretKey: SMTP_PASSWORD
      remoteRef:
        key: [OP_VAULT_ID]/[OP_ITEM_ID]
        property: smtp_password

  dataFrom:
    - extract:
        key: [OP_VAULT_ID]/[OP_ITEM_BUNDLE]
```

**Requisits al cluster:**

- `external-secrets` operator desplegat (namespace: external-secrets)
- `ClusterSecretStore` configurat apuntant a 1Password Connect:

```yaml
# clusters/[CLUSTER_NAME]/infrastructure/external-secrets/onepassword-store.yaml
---
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: onepassword-connect
spec:
  provider:
    onepassword:
      connectHost: http://[OP_CONNECT_HOST]:[OP_CONNECT_PORT]
      vaults:
        [OP_VAULT_NAME]: [OP_VAULT_ID]
      auth:
        secretRef:
          connectTokenSecretRef:
            name: onepassword-connect-token
            key: token
            namespace: external-secrets
```

---

### 3. Kustomization que genera un Secret des d'un fitxer .env

Kustomize pot generar Secrets automàticament a partir d'un fitxer `.env`. Ideal per credencials locals o config sensible que no està a 1Password.

```yaml
# kustomization.yaml — Genera un Secret des d'un .env
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: [APP_NAME]-secrets
  namespace: flux-system
spec:
  interval: 10m
  path: ./apps/[NAMESPACE]/[APP_NAME]/secrets
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-gpg
```

```yaml
# secrets/kustomization.yaml — Configuració kustomize local
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: [NAMESPACE]

secretGenerator:
  - name: [APP_NAME]-app-secrets
    envs:
      - app.env
    type: Opaque
    options:
      labels:
        app.kubernetes.io/name: [APP_NAME]
      annotations:
        generated-by: "kustomize-secret-generator"

generatorOptions:
  disableNameSuffixHash: false
```

```bash
# secrets/app.env — Fitxer .env (xifrat amb SOPS o en git-crypt)
DB_PASSWORD=[DB_PASSWORD]
REDIS_PASSWORD=[REDIS_PASSWORD]
API_KEY=[API_KEY]
JWT_SECRET=[JWT_SECRET]
SMTP_PASSWORD=[SMTP_PASSWORD]
OAUTH_CLIENT_SECRET=[OAUTH_CLIENT_SECRET]
```

> **Important:** Mai pujar `.env` en clar al repositori. Usar SOPS (Mozilla sops) per xifrar-lo, o desar-lo via ExternalSecret. El `decryption.provider: sops` al Kustomization fa que Flux el desxifri automàticament.

---

### 4. Patch strategic merge a un Deployment

Kustomize permet modificar un Deployment existent des d'un HelmRelease o YAML base sense tocar-lo. Exemple: afegir un sidecar per logging i modificar resources.

```yaml
# patches/increase-resources.yaml — Patch per resources i sidecar
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: [APP_NAME]
  namespace: [NAMESPACE]
spec:
  replicas: 3
  template:
    spec:
      containers:
        # Sidecar de logging
        - name: fluent-bit
          image: [REGISTRY_URL]/fluent-bit:[FLUENT_BIT_VERSION]
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: config
              mountPath: /fluent-bit/etc
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi
        # Modificar el contenidor principal
        - name: [APP_NAME]
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1
              memory: 1Gi
          env:
            - name: LOG_FORMAT
              value: json
            - name: TRACE_ENABLED
              value: "true"
      volumes:
        - name: varlog
          emptyDir: {}
        - name: config
          configMap:
            name: [APP_NAME]-fluent-config
```

```yaml
# kustomization.yaml — Aplica el patch
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: [NAMESPACE]

resources:
  - ../base  # HelmRelease o Deployment base

patches:
  - target:
      kind: Deployment
      name: [APP_NAME]
    path: patches/increase-resources.yaml
```

**Comandes útils per debug:**

```bash
# Veure el resultat del patch sense aplicar
kustomize build apps/[NAMESPACE]/[APP_NAME]/

# Verificar que el patch és vàlid
kustomize build apps/[NAMESPACE]/[APP_NAME]/ | kubectl apply --dry-run=client -f -
```

---

### 5. ImageUpdateAutomation per auto-update d'imatges

Flux pot actualitzar automàticament les imatges dels manifests quan es publica una nova versió al registre.

**Requisits:** `flux install --components-extra=image-reflector-controller,image-automation-controller`

```yaml
# imagepolicy.yaml — Política d'actualització per imatge
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: [APP_NAME]
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: [APP_NAME]
  policy:
    semver:
      range: ">=1.0.0 <2.0.0"  # Rang de versions acceptades
```

```yaml
# imagerepository.yaml — Escaneja el registre per tags
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: [APP_NAME]
  namespace: flux-system
spec:
  image: [REGISTRY_URL]/[IMAGE_NAME]
  interval: 5m
  secretRef:
    name: registry-credentials  # opcional: si el registre requereix auth
```

```yaml
# imageupdateautomation.yaml — Auto-commit dels canvis al repositori
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 30m
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        name: FluxCD Automation
        email: flux@[DOMAIN]
      messageTemplate: 'chore(auto): update image {{ .Updated.Image }}'
    push:
      branch: main
  update:
    path: ./clusters/[CLUSTER_NAME]
    strategy: Setters
```

**Funcionament:**

1. `ImageRepository` escaneja el registre OCI/Docker cada 5 min
2. `ImagePolicy` filtra per rang semver (ex: >=1.0.0 <2.0.0)
3. `ImageUpdateAutomation` fa commit automàtic al repositori quan detecta una nova versió
4. Flux reconcilia el canvi i desplega la nova imatge

**Marcar un tag com a actualitzable** al manifest:

```yaml
# helmrelease.yaml o deployment.yaml
spec:
  values:
    image:
      tag: "1.0.0"  # Flux overescriurà aquest valor
```

O amb l'anotació `fluxcd.io/automated`:

```yaml
metadata:
  annotations:
    fluxcd.io/automated: "true"
```

---

## Estructura plana per app (sense overlays)

Per clusters petits o apps senzilles, evitar overlays `base/` + `overlays/` i usar estructura plana.

```
apps/
└── [NAMESPACE]/
    └── [APP_NAME]/
        ├── kustomization.yaml     ← Kustomization Flux
        ├── helmrelease.yaml       ← si usa chart Helm
        ├── deployment.yaml        ← si usa YAML directe
        ├── service.yaml
        ├── ingress.yaml
        ├── configmap.yaml         ← config directa
        ├── externalsecret.yaml    ← secrets des d'1Password
        ├── hpa.yaml               ← autoescalat opcional
        └── patches/               ← opcional: patches separats
            ├── resources.yaml
            └── sidecar.yaml
```

**Avantatges:**

- Menys complexitat (sense heretar de bases)
- Més fàcil d'entendre per nous contributors
- Cada app és autocontinguda
- Fàcil de promoure entre clusters (copiar directori)

**Quan usar overlays en comptes d'estructura plana:**

- Necessites variants per entorn (staging vs production) amb canvis significatius
- Comparteixes una base comuna entre múltiples clusters
- Tens policies o patches globals que s'apliquen a totes les apps

**Exemple minimal Kustomization per estructura plana:**

```yaml
# kustomization.yaml — Estructura plana
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: [APP_NAME]
  namespace: flux-system
spec:
  interval: 10m
  path: ./apps/[NAMESPACE]/[APP_NAME]
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: [APP_NAME]
      namespace: [NAMESPACE]
```

---

## Resum de recursos FluxCD

| Recurs | Controller | Funció |
|--------|-----------|--------|
| `GitRepository` | source-controller | Fuente de manifests des de Git |
| `OCIRepository` | source-controller | Fuente de charts des de registre OCI |
| `HelmRepository` | source-controller | Fuente de charts Helm HTTP |
| `HelmRelease` | helm-controller | Desplega un chart Helm |
| `Kustomization` | kustomize-controller | Aplica manifests YAML/kustomize |
| `ImageRepository` | image-reflector-controller | Escaneja registres per tags |
| `ImagePolicy` | image-reflector-controller | Defineix política de versions |
| `ImageUpdateAutomation` | image-automation-controller | Auto-commit de noves imatges |
| `Bucket` | source-controller | Fuente des de bucket S3/GCS |

---

## Referències

- [FluxCD Documentation](https://fluxcd.io/docs/)
- [FluxCD Image Automation](https://fluxcd.io/docs/guides/image-update/)
- [External Secrets Operator](https://external-secrets.io/)
- [Kustomize Patches](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/)
- [SOPS + FluxCD](https://fluxcd.io/docs/guides/mozilla-sops/)
