# CI/CD

## Objetivo de aprendizaje

CI/CD no es "configurar GitHub Actions". Es **automatizar el camino desde un commit hasta producción** de forma repetible, rápida y segura. Aquí se estudian los **bloques** que componen ese camino: integración continua, build de imágenes, tests, deploy, versionado y rollback.

La meta es que el estudiante entienda que **un pipeline bien diseñado es un producto en sí mismo**: rápido, observable y que falla con información útil.

---

## Mapa general

| Componente       | Función                                              |
| ---------------- | ---------------------------------------------------- |
| CI (Integración) | Cada commit compila, testa y se valida              |
| Build            | Empaqueta la app (imagen Docker, binario, bundle)   |
| Testing          | Unit, integration, e2e automatizados                |
| Deploy           | Llevar el build a un entorno (staging, producción)  |
| Versionado       | Identificar de forma única cada build                |
| Rollback         | Volver a una versión anterior cuando algo falla     |

---

## 1. GitHub Actions

### Idea central

Plataforma de **automatización** integrada en GitHub. Defines workflows en YAML que corren en **eventos** (push, PR, schedule) en **runners** (máquinas que ejecutan tus jobs).

### Anatomía de un workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"

      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test -- --coverage

      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/
```

### Eventos comunes

| Evento           | Cuándo corre                          |
| ---------------- | ------------------------------------- |
| `push`           | Push a una rama                       |
| `pull_request`   | PR abierto o sincronizado             |
| `workflow_dispatch` | Manual desde la UI o API            |
| `schedule`       | Cron (`cron: "0 6 * * *"`)           |
| `release`        | Cuando se publica un release          |
| `push tags`      | Push de un tag (`v*`)                |

### Estrategias para velocidad

- **Cache** de dependencias: `actions/cache` o nativo (`cache: "npm"`).
- **Matrix builds**: paraleliza por versión de Node, OS, etc.
- **Sharding** de tests: divide E2E en N grupos paralelos.
- **Conditional steps**: solo corre lo necesario.
- **Self-hosted runners**: para builds pesados o con caché persistente.

### Ejemplo: matrix + cache

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [18, 20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: "npm"
      - run: npm ci
      - run: npm test
```

### Reutilización con composite actions

```yaml
# .github/actions/setup-node-project/action.yml
name: Setup Node Project
description: Setup Node, cache, install
runs:
  using: composite
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: 20
        cache: npm
    - run: npm ci
      shell: bash
```

```yaml
# workflow que lo usa
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-node-project
  - run: npm test
```

### Secretos

```yaml
- name: Deploy
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  run: ./deploy.sh
```

Secretos en **Settings → Secrets and variables → Actions**. Nunca en código.

### Buenas prácticas

- **Pinear versiones de actions**: `@v4` no `@main`.
- **Permisos mínimos**: `permissions: read-all` por default, sobreescribe lo necesario.
- **Timeouts** explícitos: `timeout-minutes: 15`.
- **Concurrencia**: cancela runs obsoletos en el mismo branch.

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

---

## 2. Docker Build en CI

### Idea central

Construir una **imagen inmutable** por cada commit, etiquetada de forma única, lista para desplegar en cualquier entorno.

### Buildx y multi-platform

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx@v3

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      ghcr.io/mi-org/api:${{ github.sha }}
      ghcr.io/mi-org/api:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
    platforms: linux/amd64,linux/arm64
```

### Cache layers en CI

Reconstruir toda la imagen cada vez es lento. Usa **cache de layers**:

- `type=gha`: GitHub Actions cache (gratis, rápido).
- `type=registry`: cache en registry (Docker Hub, GHCR).
- `type=s3`: cache en S3 (self-hosted).

### Tags útiles

- `${{ github.sha }}` — único por commit.
- `${{ github.ref_name }}` — nombre de rama.
- `v1.2.3` — desde un release.
- `latest` — solo para main, con cuidado.

### Build solo cuando cambia el código

```yaml
- name: Detect changes
  id: changes
  uses: dorny/paths-filter@v3
  with:
    filters:
      api: ['apps/api/**']
      web: ['apps/web/**']
```

Así, un cambio en `apps/web` no reconstruye la imagen de la API.

### Optimizaciones de Dockerfile para CI

- **Ordena capas** de menos a más cambiantes.
- **Multi-stage** para reducir la imagen final.
- **`.dockerignore`** agresivo.
- **Build secrets** sin que queden en la imagen final.

```dockerfile
# secret durante build, no queda en la imagen
RUN --mount=type=secret,id=npmrc \
    cp /run/secrets/npmrc ~/.npmrc && \
    npm ci
```

---

## 3. Testing en CI

### Pirámide aplicada a pipelines

```
PR abierto
  ├── lint + typecheck (10s)
  ├── unit + integration (1-2 min)
  └── build (30s)

Merge a main
  ├── todo lo anterior
  ├── e2e (5-10 min, paralelizado)
  └── deploy a staging

Manual approval
  └── deploy a producción
```

### Servicios en CI

Para integration y e2e necesitas servicios. Opciones:

- **GitHub Services**: `postgres`, `redis`, `mysql` como servicios del job.
- **Docker compose**: `docker compose up -d` en el runner.
- **Testcontainers**: levanta contenedores desde el test.

```yaml
services:
  postgres:
    image: postgres:16
    env:
      POSTGRES_PASSWORD: test
    ports: [5432:5432]
    options: >-
      --health-cmd "pg_isready -U postgres"
      --health-interval 5s
```

### Sharding de E2E

Playwright puede partir los tests en N shards que corren en paralelo:

```yaml
jobs:
  e2e:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - run: npx playwright test --shard=${{ matrix.shard }}/4
```

```bash
# correr todos los shards en un solo job (local)
npx playwright test --shard=1/4
npx playwright test --shard=2/4
...
```

### Reporte de resultados

```yaml
- uses: actions/upload-artifact@v4
  if: always()
  with:
    name: playwright-report-${{ matrix.shard }}
    path: playwright-report/
```

Luego un **job final** descarga todos los reports y los fusiona con `merge-reports`.

### Quality gates

```yaml
- name: Coverage gate
  run: |
    COVERAGE=$(jq '.total.lines.pct' coverage/coverage-summary.json)
    if (( $(echo "$COVERAGE < 80" | bc -l) )); then
      echo "Coverage $COVERAGE% below 80%"
      exit 1
    fi
```

---

## 4. Deploy

### Estrategias de despliegue

| Estrategia         | Cómo funciona                                  | Downtime | Riesgo de rollback |
| ------------------ | ---------------------------------------------- | -------- | ------------------ |
| **Recreate**       | Para el viejo, levanta el nuevo                | Sí       | Bajo               |
| **Rolling**        | Sustituye instancias gradualmente              | No       | Medio              |
| **Blue/Green**     | Dos entornos, switch atómico del LB           | No       | Muy bajo (inmediato) |
| **Canary**         | % pequeño de tráfico al nuevo, luego todo     | No       | Bajo               |

### Rolling update con Docker Compose

```bash
docker compose up -d --no-deps --scale api=3 --scale api_old=0
docker compose rm -f api_old
```

### Blue/Green con Nginx

```nginx
upstream backend {
  server api-blue:3000;
  server api-green:3000 backup;  # se activa al fallar blue
}

# cambiar el "primary" actualizando la config
```

### Canary con Kubernetes

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: { duration: 10m }
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 100
```

### Deploy environments

```
dev (cada PR) → staging (cada merge a main) → production (manual approval)
```

Cada environment tiene sus **secretos, variables y configuración** distintos.

### Feature flags

Permiten deployar código **sin activarlo**:

```ts
if (flags.isEnabled("new-checkout", { userId })) {
  return <NewCheckout />;
}
return <OldCheckout />;
```

Proveedores: LaunchDarkly, Unleash, Flagsmith, PostHog, o casero.

Beneficio: deploy y release son **dos acciones separadas**. Deploy técnico ≠ feature al usuario.

### GitOps

El estado deseado del cluster está **en un repo Git**. Argo CD / Flux lo sincroniza.

```
Git repo (manifests) → Argo CD → Kubernetes cluster
```

- Audit trail automático.
- Rollback = `git revert`.
- Diff visual de qué cambió.

---

## 5. Versionado

### Idea central

Cada build debe tener un **identificador único e inmutable** que permita rastrear qué código, qué configuración y qué artefacto se desplegó.

### Semantic Versioning (SemVer)

```
MAJOR.MINOR.PATCH
  ↑       ↑      ↑
  ↑       ↑      └─ bugfixes compatibles
  ↑       └──────── nuevas features compatibles
  └──────────────── cambios incompatibles
```

- `1.0.0` → primera release.
- `1.1.0` → nueva feature.
- `1.1.1` → bugfix.
- `2.0.0` → breaking change.

### Pre-releases

```
1.0.0-alpha.1
1.0.0-beta.2
1.0.0-rc.1
1.0.0  ← estable
```

### Tags de Git

```bash
git tag -a v1.4.2 -m "Release 1.4.2"
git push origin v1.4.2
```

En GitHub Actions, un push de tag puede triggear un deploy:

```yaml
on:
  push:
    tags: ['v*']
```

### Conventional Commits

Mensajes de commit que **generan changelogs automáticamente**:

```
feat: añadir endpoint de cancelación de pedidos
fix: corregir cálculo de total con descuentos
feat!: cambiar API de autenticación a OAuth2
docs: actualizar README
chore: actualizar dependencias
```

Con `standard-version` o `release-please`, el CHANGELOG y la versión se generan del historial.

### Release Please (automatización)

```yaml
# .github/workflows/release.yml
on:
  push:
    branches: [main]

jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```

Abre PRs con bump de versión y CHANGELOG. Al mergear, crea el release.

### CalVer (alternativa)

Útil cuando el ritmo es rápido y la compatibilidad no es el foco:

```
2026.03.15     (fecha)
2026.03.15.1   (segundo release del día)
```

Usado por empresas como Ubuntu (`24.04`).

---

## 6. Rollback

### Idea central

Poder **volver a la versión anterior** rápido y sin error humano cuando algo sale mal. Es la **red de seguridad** de cualquier deploy.

### Tipos de rollback

| Tipo                | Cuándo                                  | Velocidad   |
| ------------------- | --------------------------------------- | ----------- |
| **Re-deploy**        | Volver a desplegar la versión anterior  | Minutos     |
| **Switch de tráfico**| Blue/Green: volver al entorno anterior  | Segundos    |
| **Revert de Git**    | GitOps: revertir el commit              | Segundos    |
| **Feature flag off** | Apagar la feature sin re-deploy         | Instantáneo |

### Estrategia por entorno

- **Producción**: blue/green o canary + rollback automático en health check failure.
- **Staging**: redeploy es suficiente.
- **Desarrollo**: cualquier cosa vale.

### Health checks como red de seguridad

```
Deploy nueva versión (canary 5%)
  ↓
Health checks OK durante 5 min?
  ├─ Sí → subir a 25% → ... → 100%
  └─ No → rollback automático
```

### Rollback en Kubernetes

```bash
kubectl rollout undo deployment/api
kubectl rollout undo deployment/api --to-revision=3
```

### Rollback en Docker Compose

```bash
# Tag de la versión anterior, desplegar
export IMAGE_TAG=1.4.1
docker compose pull
docker compose up -d
```

### Lo que un buen rollback debe tener

- **Trigger claro**: alerta → decisión.
- **Tiempo objetivo**: "rollback en menos de 5 minutos".
- **Documentado**: runbook con los pasos exactos.
- **Probado**: drills periódicos de rollback.
- **Sin pérdida de datos**: si la nueva versión escribió datos incompatibles, hay un problema mayor.

---

## Pipeline completo de referencia

```
PR abierto a main
  │
  ├── Job 1: lint + typecheck         (1 min)
  ├── Job 2: unit + integration       (3 min)
  └── Job 3: build imagen (sin push)  (1 min)

  → checks required: 3 jobs
  → no se puede mergear si alguno falla

Merge a main
  │
  ├── Job 1-3: igual que en PR
  ├── Job 4: e2e sharded (4 jobs en paralelo)  (5 min)
  ├── Job 5: build + push imagen      (2 min)
  ├── Job 6: deploy a staging         (2 min)
  └── Job 7: smoke tests en staging   (2 min)

Tag v* pusheado
  │
  └── Job 8: deploy a producción (manual approval)
      ├── canary 5% durante 10 min
      ├── canary 25% durante 10 min
      └── 100% con health checks OK

Si falla cualquier health check → rollback automático
```

### Ejemplo: workflow de deploy

```yaml
# .github/workflows/deploy-prod.yml
name: Deploy Production

on:
  push:
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/mi-org/api:${{ github.ref_name }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      url: https://api.example.com
    steps:
      - name: Deploy
        run: ./scripts/deploy.sh ${{ github.ref_name }}
      - name: Smoke test
        run: ./scripts/smoke-test.sh
```

---

## Buenas prácticas transversales

- **Idempotencia**: el mismo input produce el mismo output. Re-correr el deploy no rompe nada.
- **Pipelines rápidos**: cada minuto de CI cuesta dinero y tiempo de developer. Apunta a < 10 min para el flujo principal.
- **Fail fast**: jobs baratos primero (lint, typecheck). Si fallan, no gastes en tests lentos.
- **Observabilidad del pipeline**: status checks, badges en README, notificaciones a Slack.
- **Trunk-based development**: ramas cortas, integrate a main seguido.
- **Inmutabilidad**: la imagen taggeada con `${{ github.sha }}` nunca cambia. Deploy = cambiar a qué tag apunta el cluster.
- **Separación deploy / release**: feature flags para que el deploy técnico no signifique feature al usuario.
- **No commitear secretos**: ni en código, ni en `.env`, ni en secrets de GitHub Actions que no sean por environment.
- **Tests del propio pipeline**: GitHub Actions puede fallar. Ten un fallback (notificación + runbook).

---

## Seguridad del pipeline

### Amenazas comunes

| Amenaza                     | Mitigación                                |
| --------------------------- | ----------------------------------------- |
| PR malicioso con secrets    | `gitleaks` o `trufflehog` en CI            |
| Dependencia comprometida     | Dependabot, Renovate, `npm audit`        |
| Action comprometida          | Pinear versiones, revisar provenance     |
| Secret leaks en logs         | Usar `secrets.*` enmascarados            |
| Permisos excesivos           | `permissions: read-all` por default      |

### Ejemplo: gitleaks

```yaml
- name: Detect secrets
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Ejemplo: pin de action con SHA

```yaml
# en vez de @v4
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
```

---

## Ejercicios

### Ejercicio 1 — Pipeline mínimo

Configura un workflow que en cada PR:

1. Corra lint.
2. Corra typecheck.
3. Corra unit tests.
4. Falle el PR si algo falla.

### Ejercicio 2 — Build y push de imagen

Configura un workflow que:

1. Buildee la imagen Docker.
2. La pushee a GHCR.
3. La taggee con `${{ github.sha }}`.

### Ejercicio 3 — Tests en matrix

Configura unit tests que corran en:

- Node 18, 20, 22.
- Ubuntu, macOS, Windows.

Paraleliza y mide tiempo total vs secuencial.

### Ejercicio 4 — E2E sharded

Toma una suite de Playwright con 50 tests. Configúralos en 4 shards paralelos. Mide la diferencia.

### Ejercicio 5 — Deploy con approval

Configura un workflow que:

1. Buildee la imagen en main.
2. Despliegue a staging automáticamente.
3. Requiera **manual approval** para producción.
4. Use el environment `production` con secrets separados.

### Ejercicio 6 — Release Please

Configura Release Please en tu repo:

1. Conventional commits en tu historial.
2. PRs automáticos de bump de versión.
3. CHANGELOG generado.
4. Tag al mergear.

### Ejercicio 7 — Rollback drill

Simula un rollback en producción:

1. Despliega v1.
2. Despliega v2 con un bug.
3. Detecta el bug (smoke test fallido).
4. Rollback a v1 en menos de 5 minutos.
5. Documenta el tiempo y los pasos.

### Ejercicio 8 — Secrets scanning

Integra `gitleaks` o `trufflehog` en CI. Añade un secret de prueba, verifica que falla. Limpia el secret.

---

## Problema propuesto

### Pipeline completo para un SaaS

Debes diseñar e implementar el **CI/CD completo** de un SaaS multi-tenant con frontend (Next.js), API (Node o Python), y worker (Node o Python).

### Requisitos funcionales

1. **CI en cada PR**: lint, typecheck, unit, integration, build de imagen (sin push).
2. **Merge a main**: corre todo lo anterior + e2e sharded + push de imagen + deploy a staging + smoke tests.
3. **Tag `v*`**: deploy a producción con manual approval y estrategia canary.
4. **Rollback automático** si los smoke tests fallan en producción.
5. **Release Please** con conventional commits, generando CHANGELOG y tags.
6. **Secret scanning** en cada PR.
7. **Dependency scanning** (Dependabot o equivalente) semanal.
8. **Pipeline total < 12 minutos** para el flujo principal.

### Requisitos técnicos

Tu solución debe:

1. Usar **GitHub Actions** (o equivalente documentado).
2. Tener **workflows separados** por responsabilidad:
   ```
   .github/workflows/
     ci.yml
     e2e.yml
     release.yml
     deploy-staging.yml
     deploy-prod.yml
     security.yml
   ```

3. **Cache** de dependencias e imágenes Docker.
4. **Matrix builds** donde tenga sentido.
5. **Sharding** de E2E con al menos 4 shards.
6. **Environments** de GitHub con secrets separados para staging y producción.
7. **Manual approval** para producción.
8. **Health checks** post-deploy que disparen rollback.

### Entregables

- Workflows completos.
- Configuración de environments.
- Pipeline ejecutándose de punta a punta (puede ser en un repo de demo).
- Capturas o logs de:
  - CI rápido en PR.
  - E2E sharded en paralelo.
  - Release Please abriendo un PR.
  - Deploy a staging con smoke tests.
  - Tag → deploy a producción con approval.
  - Rollback automático disparado.
- Métricas: tiempo total del pipeline, tamaño de imágenes, cache hit rate.
- Documento de 1 página justificando:
  - Por qué esa estructura de workflows.
  - Estrategia de versionado y tags.
  - Estrategia de rollback y tiempo objetivo.
  - Qué harías si el pipeline creciera 5x (más servicios, más tests).
  - Cómo manejas secretos por environment.
  - Qué falta para producción real (drift detection, GitOps, etc.).

### Criterios de evaluación

| Criterio                                        | Peso |
| ----------------------------------------------- | ---- |
| CI rápido y completo en PR                      | 20%  |
| Build + push de imagen con cache                | 10%  |
| E2E sharded en paralelo                         | 10%  |
| Deploy staging automático                       | 10%  |
| Deploy producción con approval + canary         | 15%  |
| Rollback funcional y probado                    | 15%  |
| Release Please + conventional commits           | 10%  |
| Secret scanning + seguridad básica              |  5%  |
| Documentación y métricas del pipeline           |  5%  |
