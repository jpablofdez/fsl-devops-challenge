# FSL DevOps Challenge — Laboratorio de CI con GitHub Actions

[![CI](https://github.com/jpablofdez/fsl-devops-challenge/actions/workflows/ci.yml/badge.svg)](https://github.com/jpablofdez/fsl-devops-challenge/actions/workflows/ci.yml)

Guía práctica del pipeline de Integración Continua (CI) de este repositorio: qué hace, cómo está protegida la rama `main` y una serie de ejercicios para reproducir **un PR exitoso** y **un fallo en cada etapa** del pipeline.

La aplicación es **rdiCidr**, una calculadora CIDR hecha con React (Create React App). Ver [Sobre la aplicación](#sobre-la-aplicación).

---

## Contenido

1. [Arquitectura del pipeline](#1-arquitectura-del-pipeline)
2. [Protección de la rama `main`](#2-protección-de-la-rama-main)
3. [Requisitos](#3-requisitos)
4. [Ejecutar el pipeline en local](#4-ejecutar-el-pipeline-en-local)
5. [Laboratorio](#5-laboratorio)
   - [Lab 1 — PR exitoso](#lab-1--pr-exitoso-)
   - [Lab 2 — Fallo en Lint](#lab-2--fallo-en-lint-)
   - [Lab 3 — Fallo en Test](#lab-3--fallo-en-test-)
   - [Lab 4 — Fallo en Build](#lab-4--fallo-en-build-)
   - [Lab 5 — Actualizar un PR fallido hasta dejarlo en verde](#lab-5--actualizar-un-pr-fallido-hasta-dejarlo-en-verde)
   - [Lab 6 — Intentar saltarse el pipeline (push directo a `main`)](#lab-6--intentar-saltarse-el-pipeline-push-directo-a-main)
6. [PRs de ejemplo ya creados](#6-prs-de-ejemplo-ya-creados)
7. [Comandos útiles](#7-comandos-útiles)
8. [Solución de problemas](#8-solución-de-problemas)
9. [Sobre la aplicación](#sobre-la-aplicación)

---

## 1. Arquitectura del pipeline

Archivo: [`.github/workflows/ci.yml`](.github/workflows/ci.yml)

### Disparadores

| Evento | Cuándo |
|---|---|
| `pull_request` → `opened` | Se crea un PR hacia `main` |
| `pull_request` → `synchronize` | Se suben commits nuevos a un PR (PR actualizado) |
| `pull_request` → `reopened` | Se reabre un PR |
| `push` → `main` | Se hace merge a `main`, para que la rama principal siempre muestre su estado |

### Etapas

```
 Checkout ─► Setup Node (.nvmrc) ─► npm install ─► npm run lint ─► CI=true npm run test ─► npm run build
                                                       │                  │                     │
                                                  ❌ detiene          ❌ detiene            ❌ detiene
```

| # | Step | Comando | Qué valida |
|---|---|---|---|
| 1 | Checkout | `actions/checkout` | Descarga el código del PR |
| 2 | Set up Node.js | `actions/setup-node` con `node-version-file: .nvmrc` | Usa **Node 15.5.1**, la misma versión que exige `package.json` (`engines` + `engine-strict`). Cachea npm |
| 3 | Install dependencies | `npm install` | Que las dependencias se resuelvan e instalen |
| 4 | Lint (ESLint) | `npm run lint` | Errores de estilo y calidad (`eslint ./src/`) |
| 5 | Test (Jest) | `CI=true npm run test` | Tests unitarios. `CI=true` hace que Jest corra una vez y no en modo *watch* |
| 6 | Build | `npm run build` | Que la app compile para producción |

Cada step tiene un nombre propio, así que en la pestaña **Checks** del PR se ve qué paso pasó (✅), cuál falló (❌) y cuáles se saltaron (⏭️). Cuando un step falla, los siguientes no se ejecutan.

Otras decisiones:

- **`permissions: contents: read`**: el token del workflow solo tiene los permisos mínimos.
- **`concurrency` con `cancel-in-progress`**: si se suben commits nuevos al mismo PR, se cancela la ejecución anterior para no gastar minutos en código viejo.
- **`timeout-minutes: 15`**: evita que un job colgado corra hasta el límite de 6 h.
- **Runner fijo (`ubuntu-24.04`)** en lugar de `ubuntu-latest`, para que los builds sean reproducibles.

---

## 2. Protección de la rama `main`

Ruleset **`protect-main`** (*Settings → Rules → Rulesets*):

| Regla | Efecto |
|---|---|
| Require a pull request before merging | No se puede hacer push directo a `main`. Todo cambio entra por PR |
| Require status checks to pass: **`Install, Lint, Test, Build`** | El botón de merge queda bloqueado si el CI no está en verde |
| Require branches to be up to date | El PR debe estar al día con `main`, para que el CI haya validado el código final |
| Require conversation resolution | Todos los comentarios de la revisión deben resolverse |
| Block force pushes / Restrict deletions | No se puede reescribir el historial ni borrar `main` |

> **Aprobaciones requeridas: 0.** Este es un repositorio de una sola persona y GitHub no permite aprobar tus propios PRs. La revisión se hace manualmente: leer el diff, verificar los checks y resolver los comentarios. En un equipo real se configuraría `required_approving_review_count: 1` o más, junto con `CODEOWNERS`.

---

## 3. Requisitos

| Herramienta | Versión | Instalación (macOS) |
|---|---|---|
| Git | cualquiera | incluido en macOS / `brew install git` |
| GitHub CLI (`gh`) | 2.x | `brew install gh` y luego `gh auth login` |
| nvm | 0.40+ | `brew install nvm` |
| Node.js | **15.5.1** (según `.nvmrc`) | `nvm install` (dentro del repo) |
| npm | 7.x (viene con Node 15) | — |

> `.npmrc` tiene `engine-strict=true`: con otra versión de Node, `npm install` falla a propósito.

Clonar el repositorio:

```bash
gh repo clone jpablofdez/fsl-devops-challenge
cd fsl-devops-challenge
nvm install   # instala/usa la versión del .nvmrc
nvm use
node -v       # v15.5.1
```

---

## 4. Ejecutar el pipeline en local

Antes de abrir un PR conviene correr los mismos pasos que el CI:

```bash
npm install
npm run lint
CI=true npm run test
CI=true npm run build   # CI=true para reproducir exactamente el comportamiento del runner
```

> **Importante:** GitHub Actions define `CI=true` en **todos** los steps. Con `CI=true`, `react-scripts build` trata los *warnings* de ESLint como **errores**. Si en local corres `npm run build` sin `CI=true`, un build que en CI falla puede pasar en tu máquina (ver [Lab 4](#lab-4--fallo-en-build-)).

Script de una línea que se detiene en el primer fallo, igual que el pipeline:

```bash
npm install && npm run lint && CI=true npm run test && CI=true npm run build && echo "✅ PIPELINE OK"
```

---

## 5. Laboratorio

Cada ejercicio sigue el mismo flujo:

```
crear rama ─► hacer cambio ─► probar en local ─► push ─► abrir PR ─► observar el CI ─► analizar ─► limpiar
```

Partir siempre de un `main` actualizado:

```bash
git switch main && git pull
```

> Los comandos `sed -i ''` son para macOS. En Linux usa `sed -i` (sin las comillas vacías).

---

### Lab 1 — PR exitoso ✅

**Objetivo:** ver un PR que pasa todas las etapas y se puede hacer merge.

```bash
git switch -c lab/success

# Cambio inocuo: actualizar el texto del footer
sed -i '' 's|you might enjoy reading my blog:|you may enjoy reading my blog:|' src/App.js
git diff

# Probar en local
npm run lint && CI=true npm run test && CI=true npm run build

git commit -am "docs: tweak footer wording"
git push -u origin lab/success
gh pr create --base main --title "[LAB] Successful PR" --body "All CI steps should pass."

# Seguir el CI en vivo
gh pr checks --watch
```

**Resultado esperado**

| Install | Lint | Test | Build | Merge |
|---|---|---|---|---|
| ✅ | ✅ | ✅ | ✅ | **Permitido** |

```bash
gh pr view --json mergeStateStatus -q .mergeStateStatus   # CLEAN
```

**Limpieza:** hacer merge (`gh pr merge --squash --delete-branch`) o cerrar (`gh pr close --delete-branch`).

---

### Lab 2 — Fallo en Lint ❌

**Objetivo:** que ESLint detenga el pipeline antes de correr los tests.

**Cambio:** se extrae el título a una constante con un *typo*: se declara `appTitel` pero se usa `appTitle`. ESLint lo reporta con la regla `no-undef`, que es de nivel **error**.

```bash
git switch main && git switch -c lab/lint-failure

sed -i '' 's|<div>CIDR Calculator</div>|<div>{appTitle}</div>|' src/App.js
sed -i '' 's|^function App() {|const appTitel = "CIDR Calculator";\
\
function App() {|' src/App.js
git diff

# Probar en local: debe fallar
npm run lint; echo "exit code: $?"    # exit code: 1

git commit -am "feat: extract app title into a constant"
git push -u origin lab/lint-failure
gh pr create --base main --title "[LAB] Lint failure" --body "Expected to fail at the Lint step."
gh pr checks --watch
```

**Resultado esperado**

| Install | Lint | Test | Build | Merge |
|---|---|---|---|---|
| ✅ | ❌ | ⏭️ | ⏭️ | **Bloqueado** |

Salida del step **Lint (ESLint)**:

```
src/App.js
   4:7   warning  'appTitel' is assigned a value but never used  no-unused-vars
  10:15  error    'appTitle' is not defined                      no-undef

✖ 2 problems (1 error, 1 warning)
```

**Análisis:** ESLint sale con código `1` porque hay al menos un **error**. Los warnings solos no detienen el lint (ver Lab 4).

> Guarda este PR abierto para el [Lab 5](#lab-5--actualizar-un-pr-fallido-hasta-dejarlo-en-verde).

---

### Lab 3 — Fallo en Test ❌

**Objetivo:** un cambio que compila y pasa el lint, pero rompe la lógica. Solo los tests lo detectan.

**Cambio:** un *off-by-one* en `IPv4.count()`: `31 - mask` en lugar de `32 - mask`.

```bash
git switch main && git switch -c lab/test-failure

sed -i '' 's|parseInt((1 << (32 - this.mask)).toString(2), 2);|parseInt((1 << (31 - this.mask)).toString(2), 2);|' src/lib/ipv4.js
git diff

# Probar en local
npm run lint; echo "lint exit: $?"           # 0  -> el código es "válido"
CI=true npm run test; echo "test exit: $?"   # 1  -> la lógica está mal

git commit -am "refactor: simplify address count calculation"
git push -u origin lab/test-failure
gh pr create --base main --title "[LAB] Test failure" --body "Expected to fail at the Test step."
gh pr checks --watch
```

**Resultado esperado**

| Install | Lint | Test | Build | Merge |
|---|---|---|---|---|
| ✅ | ✅ | ❌ | ⏭️ | **Bloqueado** |

Salida del step **Test (Jest)**:

```
FAIL src/tests/ipv4.test.js
  ✕ count of address in CIDR range is correct
  ✕ usable avaiable address count is calculated correctly
  ✕ usable avaiable address count (AWS) is calculated correctly

Tests:       3 failed, 7 passed, 10 total
```

**Análisis:** para `/20` se esperan `2^(32-20) = 4096` direcciones y el código devuelve `2^(31-20) = 2048`. Los dos tests de "usable count" también fallan porque dependen de `count()`.

**Limpieza:** `gh pr close --delete-branch`

---

### Lab 4 — Fallo en Build ❌

**Objetivo:** el caso más sutil. Lint ✅, tests ✅, pero el **build falla en CI**.

**Cambio:** se agrega una constante que nunca se usa.

```bash
git switch main && git switch -c lab/build-failure

sed -i '' 's|^function App() {|function App() {\
  const appVersion = "0.1.0";|' src/App.js
git diff

# Probar en local
npm run lint; echo "lint exit: $?"                 # 0 (solo 1 warning)
CI=true npm run test; echo "test exit: $?"         # 0
npm run build; echo "build sin CI exit: $?"        # 0  <- en local "funciona"
CI=true npm run build; echo "build CI exit: $?"    # 1  <- en el runner falla

git commit -am "feat: add app version constant"
git push -u origin lab/build-failure
gh pr create --base main --title "[LAB] Build failure" --body "Expected to fail at the Build step."
gh pr checks --watch
```

**Resultado esperado**

| Install | Lint | Test | Build | Merge |
|---|---|---|---|---|
| ✅ | ✅ (1 warning) | ✅ | ❌ | **Bloqueado** |

Salida del step **Build**:

```
Treating warnings as errors because process.env.CI = true.
Most CI servers set it automatically.

Failed to compile.

src/App.js
  Line 5:9:  'appVersion' is assigned a value but never used  no-unused-vars
```

**Análisis:**
- `no-unused-vars` es un **warning** en la configuración `react-app`, por eso `npm run lint` sale con `0`.
- GitHub Actions exporta `CI=true` en todos los steps, y `react-scripts build` convierte los warnings en errores cuando `CI=true`.
- Lección: "en mi máquina compila" no basta. Hay que reproducir el entorno del CI (`CI=true npm run build`).

**Limpieza:** `gh pr close --delete-branch`

---

### Lab 5 — Actualizar un PR fallido hasta dejarlo en verde

**Objetivo:** probar el disparador `synchronize` (PR *actualizado*) y el ciclo de feedback: el CI falla, se corrige y el CI pasa.

Partiendo del PR del **Lab 2** (que sigue abierto):

```bash
git switch lab/lint-failure

# Corregir el typo
sed -i '' 's/appTitel/appTitle/' src/App.js
npm run lint && echo "lint OK"

git commit -am "fix: typo in appTitle constant"
git push                      # dispara el CI automáticamente (evento synchronize)
gh pr checks --watch
```

**Resultado esperado:** el PR pasa de ❌ a ✅. En la pestaña *Checks* se ven las dos ejecuciones: la fallida y la exitosa.

> Si `main` avanzó mientras tanto, GitHub muestra *"This branch is out-of-date"* (por la regla *require up to date*). Hay que pulsar **Update branch** o ejecutar `gh pr update-branch`, y el CI corre de nuevo sobre el código combinado.

**Limpieza:** hacer merge (`gh pr merge --squash --delete-branch`) o cerrar.

---

### Lab 6 — Intentar saltarse el pipeline (push directo a `main`)

**Objetivo:** comprobar que el ruleset impide meter código sin PR ni CI.

```bash
git switch main
git commit --allow-empty -m "test: direct push to main"
git push origin main
```

**Resultado esperado:** el push es **rechazado**:

```
remote: error: GH013: Repository rule violations found for refs/heads/main.
remote: Review all repository rules at https://github.com/jpablofdez/fsl-devops-challenge/rules?ref=refs%2Fheads%2Fmain
remote: - Changes must be made through a pull request.
remote: - Required status check "Install, Lint, Test, Build" is expected.
 ! [remote rejected] main -> main (push declined due to repository rule violations)
```

**Limpieza** (descartar el commit local que no se pudo subir):

```bash
git reset --hard origin/main
```

---

## 6. PRs de ejemplo ya creados

| PR | Escenario | Install | Lint | Test | Build | Merge |
|---|---|---|---|---|---|---|
| [#1](https://github.com/jpablofdez/fsl-devops-challenge/pull/1) | Actualizar acciones a v5 y fijar runner | ✅ | ✅ | ✅ | ✅ | Permitido |
| [#2](https://github.com/jpablofdez/fsl-devops-challenge/pull/2) | Typo → `no-undef` | ✅ | ❌ | ⏭️ | ⏭️ | Bloqueado |
| [#3](https://github.com/jpablofdez/fsl-devops-challenge/pull/3) | Off-by-one en `count()` | ✅ | ✅ | ❌ | ⏭️ | Bloqueado |
| [#4](https://github.com/jpablofdez/fsl-devops-challenge/pull/4) | Warning tratado como error con `CI=true` | ✅ | ✅ | ✅ | ❌ | Bloqueado |

---

## 7. Comandos útiles

```bash
# Estado de los checks del PR de la rama actual (exit 0 = pasa, 1 = falla)
gh pr checks
gh pr checks --watch                 # seguir en vivo

# Listar ejecuciones del workflow
gh run list --workflow ci.yml

# Ver una ejecución: jobs, steps y anotaciones
gh run view <run-id>
gh run view <run-id> --log-failed    # solo el log de los steps que fallaron

# Ver si un PR se puede hacer merge
gh pr view <n> --json mergeStateStatus -q .mergeStateStatus   # CLEAN | BLOCKED | BEHIND

# Re-ejecutar una ejecución fallida
gh run rerun <run-id> --failed

# Ver el ruleset de main
gh api repos/jpablofdez/fsl-devops-challenge/rulesets
```

---

## 8. Solución de problemas

| Síntoma | Causa | Solución |
|---|---|---|
| `npm ERR! code EBADENGINE` / `Unsupported engine` | Versión de Node distinta de 15.x (`engine-strict=true`) | `nvm use` dentro del repo |
| `nvm: command not found` | nvm no está cargado en la shell | Agregar a `~/.zshrc`: `export NVM_DIR="$HOME/.nvm"; . "$(brew --prefix nvm)/nvm.sh"` |
| `npm run test` se queda esperando | Jest en modo *watch* | Usar `CI=true npm run test` |
| El build pasa en local pero falla en CI | El CI define `CI=true` y los warnings pasan a ser errores | Probar en local con `CI=true npm run build` |
| El PR dice *"Required status check is expected"* | El workflow no se ha ejecutado aún o el nombre del check cambió | Esperar al CI. Si se renombró el job, actualizar el ruleset |
| *"This branch is out-of-date with the base branch"* | Regla *require branches to be up to date* | **Update branch** o `gh pr update-branch` |
| `node-sass` falla al instalar | Binario no disponible para tu SO/arquitectura con Node 15 | Verificar `node -v` = 15.5.1. En Apple Silicon, usar Node x64 (Rosetta) |

---

## Sobre la aplicación

### rdiCidr

Ok, let's get this out of the way. Why did I choose that name? I chose the name because palindromes are fun. The app's name stands for RDerik's Interactive  CIDR (RDICIDR). I had to add the Interactive to make the name work. I hope you like it.

RDICIDR is a react-app, but if you want to focus on the subnetting code, check the file:

```
src/lib/ipv4.js
```

You should find there how each property was calculated.

This application is not a final product. I just built it because I wanted to refresh my subnetting knowledge. Enjoy!

#### Live version

You can find a live version at the following url:

[https://rdicidr.rderik.com](https://rdicidr.rderik.com)

If you find this tool useful, you might enjoy reading my blog: [https://rderik.com](https://rderik.com)
