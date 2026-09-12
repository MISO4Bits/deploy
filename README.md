# deploy

Repositorio GitOps: manifiestos Kustomize que Argo CD sincroniza para todos los servicios de Solventa (DI-007). Sin prefijo `svc-`/`bff-`/`iac-`/etc. a propósito — no es un componente de dominio, es el repo compartido que consumen todos.

Un Argo CD por cluster, sin hub central (5.4): cada repo de infraestructura (`iac-gcp-dev`, `iac-gcp-prod`) crea su propio Argo CD y un único `Application` raíz que apunta a la carpeta `argocd/` de este repo. Desde ahí, el `ApplicationSet` de esa carpeta genera automáticamente una `Application` por cada carpeta bajo `apps/`.

## Estructura

```
deploy/
├── argocd/
│   └── applicationset.yaml   # genera una Application por carpeta de apps/*
└── apps/
    ├── bff-web/
    │   ├── base/              # Namespace, ServiceAccount, SecretProviderClass,
    │   │                      # Deployment, Service, Gateway/HTTPRoute
    │   └── overlays/dev/      # hoy solo pasa el base sin parches
    └── svc-core/
        ├── base/              # Namespace, Deployment, Service (sin Gateway — solo interno)
        └── overlays/dev/
```

Convención al agregar un servicio nuevo: una carpeta `apps/<servicio>/base` + `apps/<servicio>/overlays/{dev,prod}` — el `ApplicationSet` la recoge sola, sin tocar `argocd/applicationset.yaml`.

## Notas

- **Exposición interna**: cada servicio expuesto públicamente trae su propio `Gateway`/`HTTPRoute` (Gateway API de GKE, `gke-l7-global-external-managed`) en su `base/`. Esa dirección es la que consume `iac-gcp-dev/modules/ingress` como backend del API Gateway (`x-google-backend` en el `openapi.yaml` del servicio correspondiente). La dirección se asigna en cuanto el `Gateway` se sincroniza, sin importar si el `Deployment` ya tiene una imagen real corriendo detrás.
- **Tag de imagen — nunca en `base/deployment.yaml`**: `base/` es compartido entre `overlays/dev` y `overlays/prod`, así que el `Deployment` ahí solo trae un tag placeholder (`:latest`, nunca corre así). El tag real de cada ambiente va en el bloque `images:` del `kustomization.yaml` del overlay correspondiente — eso es justo lo que edita `kustomize edit set image` (DI-007, 5.2).
- **`bff-web` y `svc-core` hoy**: `overlays/dev` fija el tag `develop` (DI-006, 4.5), ya publicado en Artifact Registry (ambos repos ya tienen pipeline de CI real — `.github/workflows/ci.yml`, jobs `gitleaks`→`quality`→`sonar`→`image`) — `imagePullPolicy: Always` porque es un tag flotante. `bff-web` corre en `BFF_ADAPTERS=http` (identidad y CoreTransaccional reales, ver `app/adapters/factory.py` en ese repo — no hay modo mixto) contra `svc-core` real y contra Identity Platform real (no el emulador local de Firebase Auth).
- **Write-back de CI (5.2) — implementado (2026-09-12)**: `bff-web` y `svc-core` ya tienen un job `deploy` en su `ci.yml` (después de `image`, solo en push a `develop`) que corre `kustomize edit set image` sobre `apps/<servicio>/overlays/dev` fijando el tag inmutable `sha-<corto>` de esa build (no el tag flotante `develop`), y commitea/pushea el cambio a `deploy`. Con eso, cada build a `develop` produce un cambio de texto real en este repo — que es lo que Argo CD necesita ver para disparar el rollout (auto-sync + selfHeal, DI-007 5.6). `overlays/prod` no existe todavía; cuando exista, ese camino va por PR + 2 revisiones (DI-007, 5.7), no por commit directo como el de `develop`.

## Configuración manual — una sola vez para todo el proyecto

**GitHub App para el write-back (DI-007, 5.3)**: el job `deploy` de cada servicio necesita `vars.DEPLOY_GITHUB_APP_ID` + `secrets.DEPLOY_GITHUB_APP_PRIVATE_KEY` para generar un token de instalación de 1 hora (`actions/create-github-app-token`), en vez de un token de acceso personal (PAT) de una persona. Pasos, una sola vez:
1. Crear la GitHub App en la organización `MISO4Bits` (Settings → Developer settings → GitHub Apps → New GitHub App). Permiso: **Contents: Read and write**, solo sobre repositorios seleccionados.
2. Instalarla **únicamente en el repo `deploy`** — ningún otro repo necesita que esta App esté instalada ahí, porque el token que genera ya apunta a `deploy` sin importar desde qué repo se pida.
3. Generar una clave privada de la App y guardar su `App ID`.
4. En la organización `MISO4Bits` (Settings → Secrets and variables → Actions), a nivel **organización** (no repo por repo, para que todo servicio nuevo lo herede sin configurar nada): variable `DEPLOY_GITHUB_APP_ID` = el App ID, secreto `DEPLOY_GITHUB_APP_PRIVATE_KEY` = el contenido del archivo `.pem` de la clave privada.

Sin esto, el job `deploy` de `bff-web`/`svc-core` falla en el primer paso (generar el token) — el resto del pipeline (`gitleaks`, `quality`, `sonar`, `image`) no se ve afectado.
- **Secretos — decisión final 2026-09-12**: Secret Manager + el *add-on* nativo de Secret Manager para Kubernetes Engine (interfaz de almacenamiento de contenedores, CSI) — no External Secrets Operator (DI-007, 5.8 descartado), no un HashiCorp Vault propio. Este repo solo tiene `ServiceAccount` (anotada para Workload Identity) + `SecretProviderClass` (referencias, el nombre del secreto en Secret Manager) — nunca un valor. El contenedor del secreto y el acceso los crea `iac-gcp-dev/modules/secrets`; **el valor se sube a mano después de aplicar** — ver "Pasos manuales" en el README de `iac-gcp-dev`. `bff-web` lo consume vía `secrets_dir` de `pydantic-settings` (`app/config.py`), como archivo montado en `/var/secrets`, sin variables de entorno intermedias.
- **Rollback (5.9)**: `git revert` en este repo.
