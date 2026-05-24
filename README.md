# WorldBinary CI/CD — Reusable Workflows

Repositorio único de **workflows reutilizables** de GitHub Actions para todos los
productos de WorldBinary (worldbinary, impulse, cuevatech). Centraliza la lógica
de CI, build y deploy: cada microservicio solo declara un caller delgado.

---

## 1. Idea central

El dev de cada repo **no escribe lógica de pipeline**. Declara 4 datos y el
reusable resuelve todo lo demás (cuenta AWS, región, role OIDC, registry ECR,
nombre de recursos):

| Dato | Ejemplo | Para qué |
|---|---|---|
| `product` | `worldbinary` | mapea a cuenta/región (`accounts.json`) |
| `env` | `dev` / `qa` / `staging` / `prod` | ambiente destino |
| `microservice` | `ms-template` | nombre del repo ECR + carpeta de manifests |
| `type` | `kubernetes` / `lambda` | cómo se despliega |

Regla: **toda la lógica vive una vez aquí**; agregar un micro = copiar un caller.

---

## 2. Arquitectura

```
push → CI (tsc/lint/test) → build (imagen→ECR + scan) → CD (segun type) → ambiente
                                          │
                                          └── reporte Trivy → DefectDojo (panel)
```

- **CI** no construye imagen ni despliega.
- **build** solo en ramas de ambiente (no en feature).
- **CD** rutea por `type`: `kubernetes` → ArgoCD (GitOps), `lambda` → push directo.

---

## 3. Reusables

| Workflow | Rol |
|---|---|
| `ci.yml` | Padre de CI. Hoy `lang: node`; multi-lenguaje a futuro (additivo). |
| `ci-node.yml` | `npm ci` + `tsc --noEmit` + `eslint` + `test`. |
| `build-image.yml` | Build Docker → ECR (OIDC). Tag `SHORT_SHA-TIMESTAMP`, **nunca `:latest`**. Trivy scan (gate CRITICAL) + sube reporte a DefectDojo. |
| `cd.yml` | Dispatcher de deploy. Rutea por `type`. |
| `cd-argocd.yml` | `type=kubernetes`: parcha la imagen en el repo de manifests y pushea → ArgoCD reconcilia (pull). |
| `cd-lambda.yml` | `type=lambda`: `aws lambda update-function-code --image-uri`. No toca env vars. |
| `actions/resolve` | Composite action: lee `accounts.json` y deriva account/region/role_arn/registry. |
| `accounts.json` | Mapa `product × env → { account, region }`. |

---

## 4. accounts.json

```json
{
  "worldbinary": {
    "dev":     { "account": "294337989757", "region": "us-west-2" },
    "qa":      { "account": "294337989757", "region": "us-west-2" },
    "staging": { "account": "683707058612", "region": "us-west-2" },
    "prod":    { "account": "788655294923", "region": "us-west-2" }
  }
}
```

`resolve` deriva (no se listan a mano):
- `role_arn` = `arn:aws:iam::<account>:role/github-actions-deployer`
- `registry` = `<account>.dkr.ecr.<region>.amazonaws.com`

> `dev` y `qa` comparten cuenta/cluster. `staging` = cuentas `*_Staging`.

---

## 5. Branching → ambiente

Regla: **1 rama = 1 ambiente**.

| Rama | Pipeline | Ambiente |
|---|---|---|
| `feature/**` | solo **CI** (sin imagen, sin deploy) | — |
| `develop` | CI + build → deploy | dev |
| `qa` | CI + re-tag (intra-cuenta) → deploy | qa |
| `staging` | CI + build → deploy | staging |
| `main` | CI + promote (cross-account) → deploy | prod |
| `hotfix/**` | desde tag prod → PR main → prod | prod |

> En el repo template las ramas de prueba son provisionales: dev = `ci-cd-develop`,
> staging = `ci-cd-stagging`. Las reales serán `develop`/`qa`/`staging`/`main`.

---

## 6. Convenciones de naming (derivadas)

| Recurso | Regla | Ejemplo |
|---|---|---|
| Repo ECR / imagen | `<product>-<microservice>` | `worldbinary-ms-template` |
| Tag de imagen | `SHORT_SHA-TIMESTAMP` | `262a87f-20260523233841` |
| K8s manifests path | `<product>-dev/wb-<env>/<microservice>/<deployment.yaml>` | `worldbinary-dev/wb-dev/ms-template/deployment.yaml` |
| Lambda function | `<lambda_name>-<envseg>` (dev→`develop`) | `worldbinary-binaries-ms-alerts-develop` |

---

## 7. Cómo declarar un caller

Ejemplo `develop` de un microservicio Kubernetes:

```yaml
name: CI/CD develop
on:
  push:
    branches: [develop]

permissions:
  id-token: write
  contents: read

jobs:
  ci:
    uses: WorldBinary/worldbinary-infra-workflows/.github/workflows/ci.yml@main
    with: { lang: node }

  build:
    needs: ci
    uses: WorldBinary/worldbinary-infra-workflows/.github/workflows/build-image.yml@main
    with: { product: worldbinary, env: dev, microservice: ms-template }
    secrets: inherit                 # pasa DEFECTDOJO_TOKEN

  deploy:
    needs: build
    uses: WorldBinary/worldbinary-infra-workflows/.github/workflows/cd.yml@main
    with:
      product: worldbinary
      env: dev
      microservice: ms-template
      type: kubernetes               # o lambda (+ lambda_name)
      image_uri: ${{ needs.build.outputs.image_uri }}
    secrets:
      MANIFESTS_GH_TOKEN: ${{ secrets.MANIFESTS_GH_TOKEN }}
```

Y el caller `feature/**` (solo CI):

```yaml
name: CI
on:
  push: { branches: ["feature/**"] }
  pull_request: { branches: [develop] }
jobs:
  ci:
    uses: WorldBinary/worldbinary-infra-workflows/.github/workflows/ci.yml@main
    with: { lang: node }
```

---

## 8. Secrets y variables (a nivel organización)

| Nombre | Tipo | Uso |
|---|---|---|
| `MANIFESTS_GH_TOKEN` | secret | PAT `contents:write` sobre el repo de manifests (push GitOps de `cd-argocd`). |
| `DEFECTDOJO_TOKEN` | secret | Token API DefectDojo para subir reportes de scan. |
| `DEFECTDOJO_URL` | variable | `https://reportes-ci-cd.worldbinary.pro`. Si está vacía, el upload se salta. |

OIDC: cada cuenta tiene el provider GitHub + role `github-actions-deployer`
(trust por prefijo de repo `repo:WorldBinary/<product>-*`).

---

## 9. Seguridad / análisis

| Análisis | Herramienta | Comportamiento |
|---|---|---|
| Typecheck | `tsc --noEmit` | falla CI |
| Lint | `eslint` | falla CI (errores; warnings no) |
| Vulns deps + OS | Trivy (image) | **gate**: bloquea push si CRITICAL con fix |
| Secrets en imagen | Trivy | reportado |
| Reporte completo | Trivy JSON (todas severidades) | sube a DefectDojo (no bloquea) |

El gate y el reporte son pasos separados: el reporte registra TODO (incluido lo
no-bloqueante) aunque el gate corte el build.

---

## 10. DefectDojo — panel de hallazgos

Panel unificado de vulnerabilidades/hallazgos de CI/CD.

- **URL**: `https://reportes-ci-cd.worldbinary.pro`
- **Dónde corre**: EKS `eks-worldbinary-app-v1`, ns `monitoring`, nodo infra.
- **Cómo llega la data**: `build-image` → Trivy JSON → `POST /api/v2/reimport-scan/`
  con `auto_create_context`. Cada micro = **Product** `<product>-<microservice>`,
  cada ambiente = **Engagement** `<env>`. Reimport dedupe entre builds (tendencia).
- **Futuro**: Snyk / CodeQL al mismo panel cambiando `scan_type`.

---

## 11. Lo que tenemos vs. lo "pro" (production-grade)

Estado actual del pipeline y hacia dónde debe evolucionar.

| Área | Tenemos hoy | Pro (objetivo) |
|---|---|---|
| CI checks | tsc, eslint, test | + Snyk OSS/Code, CodeQL, SonarQube |
| Scan imagen | Trivy vuln + secret, gate CRITICAL | + Trivy IaC/misconfig, license, SBOM |
| Firma imagen | — | cosign sign + verify en deploy |
| Build | multi-stage alpine + cache gha | + multi-arch (arm64), distroless, non-root |
| Deploy k8s | cd-argocd: patch + push (GitOps) | + gate de health/sync, rollback automático |
| Deploy lambda | update-function-code | + alias canary, rollback a versión N-1 |
| Promoción qa→prod | cada rama buildea su imagen | promover por **digest** (crane, cross-account): imagen bit-exacta dev→prod |
| Gobernanza | — | branch protection + required checks + CODEOWNERS + Environments con reviewers (prod 2 + wait) |
| Secrets runtime | ESO (k8s) / SSM `secrets:` (lambda) | + rotación, cero `.env` |
| Migraciones DB | manual | job pre-deploy / init container versionado |
| Notificaciones | logs de Actions | + Slack ok/fail + métricas en Grafana |
| Tests e2e | — | Playwright contra qa con seed de data |
| Registro de hallazgos | DefectDojo (Trivy) | DefectDojo con todos los scanners + SLAs/triage |
| ECR | tag inmutable SHA-TS | + lifecycle (prod keep 5, dev/qa 30d, untagged 7d) |
| OIDC | role deployer con AdministratorAccess | least-privilege por servicio/cuenta |
| Runners | GitHub-hosted | self-hosted K8s (ARC) en EKS: BuildKit remoto + cache compartido + IRSA + Karpenter spot |
| Granularidad jobs | CI: checks + test (jobs); build: 1 job con steps | build/scan/push como jobs separados (requiere ARC o registro intermedio) |

### Roadmap (orden sugerido)
1. Gobernanza (branch protection + required checks) — barato, alto impacto.
2. Snyk + CodeQL → DefectDojo.
3. Promoción por digest (qa→prod) + cosign.
4. Rollback por target + notifs Slack.
5. ECR lifecycle + OIDC least-privilege.
6. **Runners K8s (ARC)** en EKS: BuildKit remoto (cache compartido) + IRSA + Karpenter spot. Habilita separación total build/scan/push y saca el build de runners GitHub-hosted. Proyecto de infra aparte.

---

## 12. Estado actual

- ✅ Reusables CI + build + CD (kubernetes/lambda).
- ✅ OIDC en las cuentas; `accounts.json` con región por ambiente.
- ✅ DefectDojo desplegado + integrado en `build-image`.
- ⏳ Callers por ambiente en cada app, manifests de apps en dev/qa, promoción
  prod, gobernanza, scans adicionales (ver tabla §11).
