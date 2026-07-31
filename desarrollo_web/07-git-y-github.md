# Módulo 7: Git y GitHub

## Objetivos de aprendizaje

Al finalizar este módulo, el estudiante será capaz de:

- Inicializar y gestionar repositorios Git locales.
- Registrar cambios con commits significativos.
- Trabajar con ramas, fusiones y resolución de conflictos.
- Publicar proyectos en GitHub y abrir Pull Requests.
- Aplicar el flujo de trabajo Git Flow.
- Desplegar sitios estáticos con GitHub Pages.
- Configurar pipelines básicos con GitHub Actions.

---

## 1. Git

**Git** es un sistema de **control de versiones distribuido** que registra los cambios de un proyecto a lo largo del tiempo, permitiendo revertir, comparar y colaborar.

### Características

- **Distribuido**: cada clon es un repositorio completo.
- **Rápido**: la mayoría de operaciones son locales.
- **Snapshot-based**: guarda "fotos" del proyecto, no diferencias de archivos.
- **Integridad**: usa SHA-1 para identificar cada cambio.
- **Tres áreas**: working directory, staging area, repository.

### Instalación

```bash
# macOS
brew install git

# Windows
winget install Git.Git

# Linux
sudo apt install git
```

### Configuración inicial

```bash
git config --global user.name  "Tu Nombre"
git config --global user.email "tu@email.com"
git config --global init.defaultBranch main
git config --global core.editor "code --wait"
git config --global core.autocrlf input  # cuidado en Windows
git config --global alias.s  "status -s"
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.unstage "reset HEAD --"
```

### Áreas de Git

```
 Working Directory         Staging Area        Repository
 (tus archivos)      ──git add──▶   (index)   ──git commit──▶   (.git)
                                                              HEAD
```

### Actividad práctica

- Configura tu identidad en Git y comprueba la configuración con `git config --list`.

---

## 2. Repositorios

Un **repositorio** (repo) es un directorio con seguimiento de versiones.

### Inicializar

```bash
mkdir mi-proyecto
cd mi-proyecto
git init                     # crea .git
git init -b main             # rama por defecto "main"
```

### Clonar

```bash
git clone https://github.com/usuario/repo.git
git clone https://github.com/usuario/repo.git mi-fork
git clone --depth 1 URL      # clon superficial (sin historial)
```

### Estado

```bash
git status
git status -s               # formato corto
```

### Añadir al staging

```bash
git add archivo.txt
git add carpeta/
git add -A                   # todo
git add -p                   # interactivamente (por hunk)
```

### Commit

```bash
git commit -m "feat: añadir login"
git commit -a -m "fix: correcion rapida"
git commit --amend -m "nuevo mensaje"   # modifica el último
```

### `.gitignore`

```gitignore
# dependencias
node_modules/
vendor/

# logs
*.log
npm-debug.log*

# entorno
.env
.env.local

# sistema
.DS_Store
Thumbs.db

# builds
dist/
build/
*.min.js
```

### Ver el historial

```bash
git log
git log --oneline
git log --oneline --graph --all
git log -p                  # con diff
git log --stat              # con estadísticas
git log --author="Ana"
git log --since="2024-01-01"
```

### Ver diferencias

```bash
git diff                          # WD vs Stage
git diff --staged                 # Stage vs Repo
git diff HEAD                     # WD vs Repo
git diff main..feature            # entre ramas
```

### Deshacer cambios

```bash
git restore archivo              # descartar cambios en WD
git restore --staged archivo    # sacar del staging
git reset --soft HEAD~1          # deshace commit, mantiene cambios en stage
git reset --mixed HEAD~1         # default, mantiene cambios sin stage
git reset --hard HEAD~1          # ⚠️ borra todo
git revert <hash>                # commit inverso (seguro)
```

### Actividad práctica

- Crea un repositorio local, haz 3 commits con archivos distintos y revisa `git log --oneline --graph`.

---

## 3. Commits

Un **commit** es una foto inmutable del proyecto. Tiene un **hash SHA-1** único, autor, fecha y mensaje.

### Anatomía

```
commit a1b2c3d4 (HEAD -> main)
Author: Ana Pérez <ana@example.com>
Date:   Wed Jan 15 10:30:00 2025

    feat: añadir formulario de contacto
```

### Buena escritura de mensajes

#### Conventional Commits

```
<tipo>(<scope>): <descripción corta>

[cuerpo]

[footer(s)]
```

Tipos:
- `feat`: nueva funcionalidad.
- `fix`: corrección de bug.
- `docs`: solo documentación.
- `style`: formato (espacios, comas), sin cambio lógico.
- `refactor`: cambio sin nueva funcionalidad ni fix.
- `test`: añadir o corregir tests.
- `chore`: tareas, build, deps.

Ejemplos:

```bash
git commit -m "feat(auth): añadir login con Google"
git commit -m "fix: evitar doble submit en formulario"
git commit -m "docs: actualizar README con ejemplos"
git commit -m "refactor(api): extraer validaciones a función"
```

#### Regla 50/72

- Línea de resumen: máximo **50 caracteres**.
- Cuerpo: envuelto a **72 caracteres** por línea.

### Commits atómicos

Cada commit debe ser una unidad lógica de cambio. Evita commits tipo:

```
"cambios varios"
wip
arreglos
```

### Cambiar el último commit

```bash
git commit --amend                  # mensaje o cambios
git commit --amend --no-edit        # añade sin cambiar mensaje
git add olvidos && git commit --amend
```

### Firmar commits (GPG)

```bash
git config --global commit.gpgsign true
git config --global user.signingkey ABCDEF123...
```

### Actividad práctica

- Haz una serie de commits siguiendo Conventional Commits para una pequeña web estática.

---

## 4. Branches (Ramas)

Las ramas permiten **desarrollar funcionalidades en paralelo** sin afectar la línea principal.

### Concepto

Una rama es simplemente un **puntero movable** a un commit. Crear una rama es O(1).

### Listar y crear

```bash
git branch               # locales
git branch -a            # locales + remotas
git branch -r            # solo remotas
git branch feature-x     # crear
git checkout -b feature-x # crear y moverse
git switch -c feature-x  # ES2020+, equivalente
```

### Moverse

```bash
git switch main
git switch feature-x
git checkout main       # legacy
```

### Renombrar y borrar

```bash
git branch -m viejo nuevo            # renombrar
git branch -d feature-x              # borrar (seguro)
git branch -D feature-x              # forzar
git push origin --delete feature-x   # borrar en remoto
```

### Remotos

```bash
git remote -v
git remote add origin git@github.com:user/repo.git
git remote set-url origin https://...
```

### Push y pull

```bash
git push -u origin main               # primer push
git push                              # siguientes
git push --set-upstream origin fix/bug
git push --force-with-lease           # push forzado seguro
git pull
git pull --rebase                    # reaplica tus commits encima del remoto
git fetch origin                     # descarga sin integrar
```

### Tracking

```bash
git branch --set-upstream-to=origin/main main
git branch -vv                       # muestra tracking
```

### Stash (guardar trabajo temporal)

```bash
git stash
git stash push -m "wip login"
git stash list
git stash pop              # aplica y borra
git stash apply stash@{0}  # aplica sin borrar
git stash drop stash@{0}
git stash clear
```

### Cherry-pick

```bash
git cherry-pick <hash>          # copia un commit a la rama actual
git cherry-pick <hash1> <hash2>
```

### Tag (etiquetas)

```bash
git tag v1.0.0                    # ligera
git tag -a v1.0.0 -m "release"    # anotada
git tag -d v1.0.0
git push origin v1.0.0
```

### Actividad práctica

- Crea 2 branches, haz cambios en cada una, vuelve a `main` y comprueba el historial con `git log --oneline --graph --all`.

---

## 5. Merge

Un **merge** combina los cambios de dos ramas.

### Tipos

```
# Fast-forward (lineal, sin commit nuevo)
main:        A --- B
                 \
feature:           C --- D
merge:        A --- B --- C --- D

# 3-way merge (crea commit de merge)
main:        A --- B
                 \       \
feature:           C --- D
merge:        A --- B --- M (merge commit)
                       \   /
                        \ /
                        (mezcla B, D)
```

### Comandos

```bash
git switch main
git merge feature-x               # por defecto (3-way o fast-forward)
git merge --no-ff feature-x       # siempre crea commit de merge
git merge --squash feature-x      # colapsa todos los commits en uno
git merge --abort                 # cancelar en curso
```

### Conflictos

Git pausa el merge y marca los archivos con conflictos:

```diff
<<<<<<< HEAD
console.log("Hola desde main");
=======
console.log("Hola desde feature");
>>>>>>> feature-x
```

Resolver:

```bash
# 1. Edita los archivos
# 2. git add archivo
# 3. git commit (o git merge --continue)
```

Herramientas visuales:

```bash
git mergetool
git difftool
```

Estrategias de merge:

```bash
git merge -X ours   # en conflicto, prioriza HEAD
git merge -X theirs # en conflicto, prioriza la rama entrante
```

### Rebase (alternativa a merge)

Reescribe el historial aplicando tus commits sobre la punta de otra rama.

```bash
git switch feature-x
git rebase main
git switch main
git merge feature-x   # ahora fast-forward
```

> ⚠️ No rebase commits ya publicados en remoto. Dificulta la colaboración.

Rebase interactivo:

```bash
git rebase -i HEAD~4
```

Abre editor con:
```
pick a1b2c3 feat: uno
pick b2c3d4 fix: dos
pick c3d4e5 chore: tres
pick d4e5f6 docs: cuatro
```

Cambios posibles:
- `pick`, `reword`, `edit`, `squash`, `fixup`, `drop`, `exec`.

### Conflictos durante rebase

```bash
git rebase --abort
git add archivo && git rebase --continue
```

### Actividad práctica

- Crea dos branches, edita el mismo archivo en ambas, intenta hacer merge y resuelve el conflicto manualmente.

---

## 6. Pull Request

Un **Pull Request** (PR) es una petición para **fusionar cambios** de una rama a otra. En GitHub es la pieza central de la colaboración.

### Flujo típico

```
1. Fork o clone
2. Crear rama feature
3. Commits
4. Push a la rama
5. Abrir PR hacia main
6. Revisar y discutir
7. Aprobar y mergear
```

### Abrir un PR desde CLI (gh)

```bash
gh pr create --base main --head feature-x \
  --title "feat: añadir login" \
  --body  "Implementa..."
gh pr list
gh pr checkout 42
gh pr merge 42 --squash
gh pr review 42 --approve
gh pr review 42 --request-changes -b "Faltan tests"
```

### Plantilla de PR

```markdown
## Descripción
Resumen de los cambios.

## Cambios
- Item 1
- Item 2

## Cómo probar
1. Paso 1
2. Paso 2

## Checklist
- [ ] Tests añadidos
- [ ] Documentación actualizada
- [ ] Sin warnings de lint
```

### Code review

Comentarios por línea (inline), sugerencias, resoluciones.

### Reglas de protección de rama (en GitHub)

- Requerir X aprobaciones.
- Requerir checks (CI).
- No permitir force push.
- Require signed commits.

### Actividad práctica

- Haz fork de un repo público, corrige un typo, abre un PR y vincúlalo a un Issue.

---

## 7. Git Flow

**Git Flow** es un modelo de ramificación propuesto por Vincent Driessen. Define roles claros para cada rama.

### Ramas principales

- `main`: código en producción. Commits etiquetados con versiones.
- `develop`: rama de integración, base para nuevas features.

### Ramas de soporte

- `feature/*`: nuevas funcionalidades desde `develop`.
- `release/*`: preparación de release.
- `hotfix/*`: correcciones urgentes desde `main`.

### Flujo visual

```
main       ●─────●──────●────────●────►
                \           /
hotfix      ●────●────●────●
                   \       \
develop    ●──●────●───────●──────────►
              \   /         \
feature   ●───●─●          ●──●

release         ●──────●──●
                       (tag v1.1)
```

### Comandos básicos (con git-flow CLI)

```bash
git flow init
git flow feature start login
git flow feature finish login
git flow release start v1.2.0
git flow release finish v1.2.0
git flow hotfix start fix-crash
git flow hotfix finish fix-crash
```

### Variante: GitHub Flow (más simple)

- Solo `main`.
- Cada cambio en una rama.
- PR → revisión → merge → deploy.

```bash
git switch -c feat-login
# commits
git push
gh pr create
# review y merge
```

### Actividad práctica

- Simula un git flow completo: crea feature, release y hotfix con su merge a las ramas correctas.

---

## 8. GitHub

**GitHub** es la plataforma de alojamiento de repositorios Git más usada del mundo. Además de hosting, ofrece:

- Issues, Discussions, Wiki.
- Pull Requests con revisión de código.
- GitHub Actions (CI/CD).
- GitHub Packages (registry).
- GitHub Pages (hosting estático).
- Codespaces (entorno en la nube).
- Copilot (asistente IA).

### Crear un repositorio

```bash
# Opción A: desde el sitio web
gh repo create mi-proyecto --public --clone
cd mi-proyecto
echo "# Mi Proyecto" > README.md
git add . && git commit -m "docs: añadir README" && git push
```

### Configuración SSH

```bash
ssh-keygen -t ed25519 -C "tu@email.com"
cat ~/.ssh/id_ed25519.pub    # pegar en GitHub → Settings → SSH keys

git remote set-url origin git@github.com:usuario/repo.git
```

### Issues

```markdown
Título: Bug: el formulario no envía datos

## Descripción
Pasos:
1. Ir a /contacto
2. Rellenar campos
3. Pulsar Enviar

## Esperado
Enviar el formulario.

## Actual
Pantalla en blanco.

## Capturas
[adjuntar]
```

Etiquetas (`labels`): `bug`, `enhancement`, `documentation`, `good first issue`.

Templates: `.github/ISSUE_TEMPLATE/bug.md`.

### Discussions

Espacio para preguntas, ideas, anuncios. Útil en proyectos OSS.

### Code Owners

```bash
# .github/CODEOWNERS
*                  @usuario
/docs/             @usuario-docs
/src/api/          @usuario-backend
```

### Secretos y seguridad

- **Secret Scanning** automático.
- **Dependabot** para dependencias vulnerables.
- **Code Scanning** con CodeQL.

### Actividad práctica

- Crea un repositorio en GitHub, configura SSH y sube un proyecto local.

---

## 9. GitHub Pages

Servicio de **hosting estático** gratuito directamente desde un repo.

### Activación

1. Settings → Pages.
2. Branch: `main` (o `gh-pages`).
3. Carpeta: `/ (root)` o `/docs`.

URL: `https://usuario.github.io/repo/`

### Tipos de sitios

- **Project site**: `usuario.github.io/repo`.
- **User/Org site**: `usuario.github.io` (repo debe llamarse `<usuario>.github.io`).

### Jekyll básico

GitHub Pages incluye Jekyll automáticamente. `_config.yml`:

```yaml
title: Mi sitio
theme: minima
permalink: /:title/
```

### Deploy desde Actions (recomendado para SPAs)

```yaml
# .github/workflows/pages.yml
name: Deploy GH Pages
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npm run build
      - uses: actions/configure-pages@v4
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./dist
  deploy:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/deploy-pages@v3
```

### Dominio personalizado

```bash
# Crear archivo CNAME en el repo
echo "mi-dominio.com" > CNAME

# Configurar DNS
# CNAME www     -> usuario.github.io.
# A    @        -> 185.199.108.153
# A    @        -> 185.199.109.153
# A    @        -> 185.199.110.153
# A    @        -> 185.199.111.153
```

En Settings → Pages → Custom domain: marcar **Enforce HTTPS**.

### Actividad práctica

- Despliega un sitio estático con `npm run build` usando GitHub Actions en una rama `gh-pages`.

---

## 10. GitHub Actions (Introducción)

**GitHub Actions** permite automatizar workflows (CI/CD) directamente en el repositorio.

### Conceptos

- **Workflow**: archivo YAML en `.github/workflows/`.
- **Job**: unidad de trabajo.
- **Step**: tarea individual.
- **Action**: bloque reusable del marketplace.
- **Runner**: máquina que ejecuta (Ubuntu, macOS, Windows o self-hosted).
- **Eventos**: disparadores (`push`, `pull_request`, `schedule`, etc.).

### Workflow básico: CI de Node.js

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

    strategy:
      matrix:
        node-version: [18, 20, 22]

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: "npm"

      - name: Instalar
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Tests
        run: npm test
        env:
          CI: true

      - name: Build
        run: npm run build

      - name: Subir artefacto
        uses: actions/upload-artifact@v4
        with:
          name: dist-${{ matrix.node-version }}
          path: dist
```

### Secrets

```yaml
- name: Deploy
  env:
    API_KEY: ${{ secrets.API_KEY }}
  run: ./deploy.sh
```

`Settings → Secrets and variables → Actions`.

### Matriz

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macOS-latest]
    node: [18, 20]
```

### Condicionales

```yaml
- if: github.ref == 'refs/heads/main'
  run: echo "Solo main"
- if: success()
- if: failure()
- if: always()
```

### Cron / schedule

```yaml
on:
  schedule:
    - cron: "0 2 * * *"   # cada día a las 2 AM UTC
```

### Disparadores manuales

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Entorno"
        required: true
        default: "staging"
        type: choice
        options: [staging, production]
```

### Cache

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.npm
      node_modules
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
```

### Marketplace de Actions

- `actions/checkout`: clonar.
- `actions/setup-node`, `setup-python`, `setup-go`.
- `actions/upload-artifact`, `download-artifact`.
- `github/codeql-action`: análisis estático.
- `docker/build-push-action`: Docker.

### Actividad práctica

- Crea un workflow que ejecute lint y tests en cada PR. Añade una badge en el README.

---

## Buenas prácticas y comandos útiles

### Alias útiles

```bash
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.st "status -s"
git config --global alias.co "checkout"
git config --global alias.last "log -1 HEAD --stat"
```

### `.gitattributes` (manejo de fines de línea)

```gitattributes
* text=auto
*.js   text eol=lf
*.png  binary
```

### Hooks (locales)

```bash
# .git/hooks/pre-commit (ejecutable)
#!/bin/sh
npm run lint
```

Con `husky` se simplifica:

```bash
npx husky init
npx husky add .husky/pre-commit "npm test"
npx husky add .husky/commit-msg 'npx --no-install commitlint --edit "$1"'
```

### Commitlint (Conventional Commits)

```bash
npm i -D @commitlint/cli @commitlint/config-conventional
echo "module.exports = { extends: ['@commitlint/config-conventional'] }" > commitlint.config.js
```

### `.gitignore` global

```bash
git config --global core.excludesfile ~/.gitignore_global
echo ".DS_Store" > ~/.gitignore_global
```

### Recuperación

```bash
git reflog              # historial de movimientos del puntero HEAD
git reset --hard HEAD@{2}
git fsck --lost-found   # buscar commits colgantes
```

---

## Recursos complementarios

### Lecturas

- [Pro Git book (gratis)](https://git-scm.com/book/es/v2)
- [Conventional Commits](https://www.conventionalcommits.org/es/)
- [GitHub Docs](https://docs.github.com/es)
- [Atlassian - Git Tutorials](https://www.atlassian.com/git/tutorials)

### Práctica interactiva

- [Learn Git Branching](https://learngitbranching.js.org/)
- [Oh My Git](https://ohmygit.org/) (juego)
- [Git Immersion](http://gitimmersion.com/)

### Herramientas

- [GitHub CLI](https://cli.github.com/)
- [GitLens (VS Code)](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens)
- [Sourcetree](https://www.sourcetreeapp.com/)
- [lazygit](https://github.com/jesseduffield/lazygit) (TUI)

---

## Evaluación sugerida

1. **Repositorio local (20%)**: 5 commits convencionales sobre un proyecto simple.
2. **Branches + merge (25%)**: crear feature, hotfix, resolver conflictos.
3. **Pull Request (20%)**: abrir PR real en GitHub con descripción y checklist.
4. **GitHub Pages (15%)**: desplegar sitio estático con dominio personalizado.
5. **GitHub Actions (20%)**: workflow CI con lint + test + build.

---

> **Nota para el estudiante**: Git es una herramienta con la que trabajarás todos los días. Invierte tiempo en entender su modelo interno (objetos, refs, tree, blob) más allá de la interfaz de alto nivel: te evitará muchos dolores de cabeza.
