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
| `sonar-scan.yml` | Gate de calidad **SonarQube** Community. Corre `sonarqube-scan-action` con `qualitygate.wait`: el job falla si el Quality Gate del proyecto queda rojo. projectKey = nombre del repo. Se invoca en PR → `develop2`. |
| `codeql.yml` | **SAST** con CodeQL: genera SARIF, lo sube a DefectDojo y corta si hay findings CRITICAL. |
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

**Etapa `feature` (lo que corre antes de integrar):** sobre cada push a una rama
`feature/**` y sobre el PR hacia `develop2` se ejecutan las pruebas de CI
(typecheck, lint, tests). Además, el PR hacia `develop2` dispara el **gate de
calidad SonarQube** (`sonar-scan.yml`): si el Quality Gate del proyecto queda
rojo, el merge se bloquea. Así, el código se valida y se mide su calidad antes
de entrar a la rama de integración. Detalle de cada prueba en §9.

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
    uses: CuevaTech/cuevatech-infra-workflows/.github/workflows/ci.yml@main
    with: { lang: node }

  build:
    needs: ci
    uses: CuevaTech/cuevatech-infra-workflows/.github/workflows/build-image.yml@main
    with: { product: worldbinary, env: dev, microservice: ms-template }
    secrets: inherit                 # pasa DEFECTDOJO_TOKEN

  deploy:
    needs: build
    uses: CuevaTech/cuevatech-infra-workflows/.github/workflows/cd.yml@main
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
    uses: CuevaTech/cuevatech-infra-workflows/.github/workflows/ci.yml@main
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

| Análisis | Herramienta | Etapa | Qué hace |
|---|---|---|---|
| Typecheck | `tsc --noEmit` | feature / CI | valida tipos TypeScript en todo el repo |
| Lint | `eslint` | feature / CI | estilo y errores estáticos de código |
| Tests unitarios | `jest` | feature / CI | corre la suite de pruebas del repo |
| Calidad de código | SonarQube Community | PR → develop2 | bugs, vulnerabilities, code smells, duplicación, cobertura |
| Vulns deps + OS | Trivy (image) | build | **gate**: bloquea si CRITICAL con fix |
| Secrets en imagen | Trivy | build | reportado |
| SAST | CodeQL | build/CI | análisis semántico del código fuente → DefectDojo |

El gate y el reporte son pasos separados: el reporte registra TODO (incluido lo
no-bloqueante) aunque el gate corte el build.

### 9.1 Qué valida cada prueba

- **Typecheck (`tsc --noEmit`)** — corre el compilador de TypeScript en modo
  solo-verificación (no emite JS). Detecta errores de tipos, imports rotos y
  contratos mal usados antes de construir la imagen. Etapa feature.
- **Lint (`eslint`)** — reglas de estilo y patrones peligrosos (variables sin
  usar, `==` vs `===`, promesas sin `await`, etc.). Etapa feature.
- **Tests unitarios (`jest`)** — ejecuta la suite del repo; verifica la lógica
  de cada módulo de forma aislada. Etapa feature.
- **SonarQube** — mide la calidad del código: bugs, vulnerabilidades, code
  smells, duplicación y cobertura. Corre como gate en el PR hacia `develop2`;
  si el Quality Gate del proyecto queda rojo, el merge se bloquea. Un proyecto
  por repo (projectKey = nombre del repo).
- **Trivy** — escanea la imagen Docker: vulnerabilidades de dependencias y del
  SO base, más secrets embebidos. Bloquea el build si hay CRITICAL con fix
  disponible; el reporte completo (todas las severidades) va a DefectDojo.
- **CodeQL** — análisis estático de seguridad (SAST) del código fuente; genera
  SARIF y lo registra en DefectDojo, cortando ante findings CRITICAL.

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

## 11. Tecnologías en uso — qué es y qué falta

Cada área del pipeline está cubierta. Aquí qué hace cada tecnología hoy y qué
quedaría por reforzar.

### GitHub Actions — workflows reutilizables
**Qué es:** el motor de CI/CD. Toda la lógica vive en este repo como reusables
(`workflow_call`); cada microservicio declara un caller delgado.
**Hoy:** CI, build y CD centralizados; agregar un micro = copiar un caller.
**Falta:** runners self-hosted (hoy GitHub-hosted); separar build/scan/push en
jobs independientes.

### SonarQube (Community, self-hosted)
**Qué es:** plataforma de análisis estático de calidad: mide bugs,
vulnerabilidades, code smells, duplicación y cobertura. Corre en EKS
`eks-worldbinary-app-v1` (`https://sonarqube.worldbinary.pro`).
**Hoy:** gate bloqueante en el PR hacia `develop2` (`sonar-scan.yml`); un
proyecto por repo. Si el Quality Gate queda rojo, no se mergea.
**Falta:** alimentar la cobertura (`lcov`) al scanner para que el gate la mida;
branch analysis / PR decoration nativos (sólo en edición de pago — en
evaluación vía AWS Marketplace).

### Trivy — escaneo de imagen
**Qué es:** escáner de vulnerabilidades de contenedores: dependencias, paquetes
del SO base y secrets embebidos.
**Hoy:** corre en `build-image`; bloquea el build ante CRITICAL con fix; el
reporte completo va a DefectDojo.
**Falta:** escaneo de IaC/misconfig, licencias y generación de SBOM.

### CodeQL — SAST
**Qué es:** análisis semántico de seguridad del código fuente (SAST) de GitHub.
**Hoy:** reusable `codeql.yml` disponible; genera SARIF a DefectDojo con gate
CRITICAL.
**Falta:** cablearlo en los callers de `develop` (aún no activo por repo).

### DefectDojo — panel de hallazgos
**Qué es:** plataforma que centraliza los hallazgos de seguridad de todos los
escáneres en un solo panel, con deduplicación y tendencia.
**Hoy:** recibe los reportes de Trivy (y CodeQL); un Product por microservicio,
un Engagement por ambiente.
**Falta:** sumar todos los escáneres y definir SLAs/triage de hallazgos.

### ECR — registro de imágenes
**Qué es:** registro privado de imágenes Docker en AWS.
**Hoy:** una imagen por build, tag inmutable `SHA-TIMESTAMP`, nunca `:latest`.
**Falta:** política de lifecycle (retención por ambiente, limpieza de untagged).

### OIDC — autenticación a AWS
**Qué es:** federación OIDC entre GitHub Actions y AWS; el pipeline asume un rol
sin credenciales de larga vida.
**Hoy:** cada cuenta tiene el provider + role `github-actions-deployer`, con
trust por prefijo de repo.
**Falta:** permisos least-privilege por servicio/cuenta (hoy el rol es amplio).

### ArgoCD — deploy Kubernetes (GitOps)
**Qué es:** controlador GitOps que reconcilia el estado del cluster contra el
repo de manifests.
**Hoy:** `cd-argocd` parcha la imagen en el repo de manifests y pushea; ArgoCD
sincroniza por pull.
**Falta:** gate de health/sync post-deploy y rollback automático.

### Deploy Lambda
**Qué es:** despliegue de funciones Lambda basadas en imagen.
**Hoy:** `cd-lambda` hace `update-function-code --image-uri`.
**Falta:** alias canary y rollback a la versión anterior.

---

## 12. Estado actual

- ✅ Reusables CI + build + CD (kubernetes/lambda).
- ✅ OIDC en las cuentas; `accounts.json` con región por ambiente.
- ✅ DefectDojo desplegado + integrado en `build-image`.
- ✅ Gate SonarQube en PR → `develop2` (self-hosted, `sonar-scan.yml`).
- ⏳ Callers por ambiente en cada app, manifests de apps en dev/qa, promoción
  prod, gobernanza (branch protection), cobertura → Sonar y CodeQL por repo
  (ver detalle §11).
