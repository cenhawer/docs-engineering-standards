# Templates — Git

> Templates de referencia para [Convenciones de Git](git-conventions.md).
> Reemplazar `{proyecto}`, `{tipo}`, `{ticket}`, `{scope}`, `{descripcion}` y similares con los valores del proyecto.
> Para el contexto y decisiones detrás de cada template, consultar la guía principal.

**Versión:** 1.0

---

## Contenido

- [A.1 — Estructura inicial del repositorio](#a1-estructura-inicial-del-repositorio)
- [A.2 — .gitattributes](#a2-gitattributes)
- [A.3 — .githooks/commit-msg](#a3-githookscommit-msg)
- [A.4 — .githooks/pre-push](#a4-githookspre-push)
- [A.5 — Instalación de hooks](#a5-instalación-de-hooks)
- [A.6 — PULL_REQUEST_TEMPLATE.md](#a6-pull_request_templatemd)
- [A.7 — CODEOWNERS](#a7-codeowners)
- [A.8 — Flujo: Backend / Frontend — Feature](#a8-flujo-backend--frontend--feature)
- [A.9 — Flujo: Backend / Frontend — Hotfix](#a9-flujo-backend--frontend--hotfix)
- [A.10 — Flujo: Backend / Frontend — Release](#a10-flujo-backend--frontend--release)
- [A.11 — Flujo: Base de Datos — Migración](#a11-flujo-base-de-datos--migración)
- [A.12 — Flujo: Librería — Release con tag](#a12-flujo-librería--release-con-tag)
- [A.13 — Flujo: Documentación](#a13-flujo-documentación)

---

## A.1 Estructura inicial del repositorio

> Contexto: [git-setup.md 1.2 Estructura Inicial](git-setup.md#12-estructura-inicial)

```
{repositorio}/
├── .gitattributes
├── .gitignore
├── .githooks/
│   ├── commit-msg
│   └── pre-push
├── .github/
│   ├── PULL_REQUEST_TEMPLATE.md
│   └── CODEOWNERS               ← opcional, recomendado en repos de BD y librerías
└── README.md
```

---

## A.2 .gitattributes

> Contexto: [git-setup.md 1.2 Estructura Inicial](git-setup.md#12-estructura-inicial)

```
# Normalizar todos los archivos de texto a LF
* text=auto eol=lf

# Scripts de shell — LF obligatorio
*.sh text eol=lf

# Archivos de Windows — CRLF
*.bat text eol=crlf
*.cmd text eol=crlf
*.ps1 text eol=crlf

# Binarios — sin conversión
*.png binary
*.jpg binary
*.jpeg binary
*.gif binary
*.ico binary
*.pdf binary
*.zip binary
*.tar binary
*.gz binary
*.woff binary
*.woff2 binary
*.ttf binary
*.eot binary
```

---

## A.3 .githooks/commit-msg

> Contexto: [git-setup.md 4.1](git-setup.md#41-commit-msg)

```bash
#!/bin/sh
# .githooks/commit-msg
# Valida que el mensaje siga Conventional Commits antes de completar el commit.

COMMIT_MSG_FILE=$1
COMMIT_MSG=$(cat "$COMMIT_MSG_FILE")

# Ignorar commits de merge y revert generados automáticamente
case "$COMMIT_MSG" in
  Merge*|Revert*) exit 0 ;;
esac

TYPES="feat|fix|hotfix|refactor|test|docs|chore|style|perf|ci|build|migration|revert"
PATTERN="^($TYPES)(\([a-z0-9_-]+\))?(!)?: .{1,72}"

if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
  echo ""
  echo "ERROR: El mensaje de commit no sigue Conventional Commits."
  echo ""
  echo "  Formato: <tipo>(<scope>): <descripción>"
  echo "  Ejemplo: feat(auth): agregar login con Google"
  echo ""
  echo "  Tipos válidos: feat, fix, hotfix, refactor, test, docs,"
  echo "                 chore, style, perf, ci, build, migration, revert"
  echo ""
  exit 1
fi
```

---

## A.4 .githooks/pre-push

> Contexto: [git-setup.md 4.2](git-setup.md#42-pre-push)

```bash
#!/bin/sh
# .githooks/pre-push
# Previene push directo a ramas protegidas.

PROTECTED_BRANCHES="main develop"
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)

for BRANCH in $PROTECTED_BRANCHES; do
  if [ "$CURRENT_BRANCH" = "$BRANCH" ]; then
    echo ""
    echo "ERROR: Push directo a '$BRANCH' está prohibido."
    echo "Crea una rama de trabajo y abre un Pull Request."
    echo ""
    exit 1
  fi
done
```

---

## A.5 Instalación de hooks

> Contexto: [git-setup.md 4](git-setup.md#4-hooks)
>
> Ejecutar una vez por clon del repositorio.

```bash
# Configurar el directorio de hooks
git config core.hooksPath .githooks

# Dar permisos de ejecución
chmod +x .githooks/commit-msg
chmod +x .githooks/pre-push
```

Para verificar la instalación:

```bash
git config core.hooksPath
# Resultado esperado: .githooks
```

---

## A.6 PULL_REQUEST_TEMPLATE.md

> Contexto: [git-conventions.md 5.2](git-conventions.md#52-título-del-pr)
>
> Ubicar en `.github/PULL_REQUEST_TEMPLATE.md`

```markdown
## ¿Qué hace este PR?

<!-- Descripción concisa del cambio. 2-3 líneas máximo. -->

## ¿Por qué?

<!-- Qué problema resuelve o qué decisión técnica toma. -->

Ticket: [{PROYECTO}-000](https://tu-tracker.com/{PROYECTO}-000)

## Tipo de cambio

- [ ] `feat` — nueva funcionalidad
- [ ] `fix` — corrección de bug
- [ ] `hotfix` — fix urgente a producción
- [ ] `refactor` — cambio interno sin impacto funcional
- [ ] `migration` — cambio de esquema de base de datos
- [ ] `docs` — documentación
- [ ] `chore` / `ci` / `build` — mantenimiento

## Checklist

- [ ] Los commits siguen Conventional Commits
- [ ] La rama tiene el nombre correcto (`tipo/{PROYECTO}-XXX-descripcion`)
- [ ] Los tests pasan localmente
- [ ] Se agregaron o actualizaron tests para el cambio
- [ ] No hay secrets ni credenciales en el código
- [ ] La documentación fue actualizada (si aplica)
- [ ] Se incluye script de rollback (solo para `migration`)

## Cómo probar

1. ...
2. ...

## Screenshots (si aplica)

<!-- Para cambios visuales de frontend -->
```

---

## A.7 CODEOWNERS

> Contexto: [git-setup.md 7](git-setup.md#7-configuración-por-tipo-de-repositorio)
>
> Ubicar en `.github/CODEOWNERS`

```
# Dueños globales del repositorio
*                   @{equipo-desarrollo}

# Migraciones — revisión obligatoria de DBA/Infra
migrations/         @{equipo-dba} @{equipo-infra}

# Pipelines CI/CD — revisión de Infra
.github/workflows/  @{equipo-infra}

# Configuración de infraestructura
infra/              @{equipo-infra}
```

---

## A.8 Flujo: Backend / Frontend — Feature

> Contexto: [git-conventions.md 4.1 Flujo principal](git-conventions.md#41-backend--microservicio)

```bash
# 1. Actualizar develop
git checkout develop
git pull origin develop

# 2. Crear rama
git checkout -b feature/{PROYECTO}-{numero}-{descripcion}

# 3. Desarrollar — commitear siguiendo Conventional Commits
git add {archivos}
git commit -m "feat({scope}): {descripcion}"

# 4. Sincronizar con develop antes del PR
git fetch origin
git rebase origin/develop
# Si hay conflictos: resolverlos, luego `git rebase --continue`

# 5. Push y abrir PR hacia develop
git push origin feature/{PROYECTO}-{numero}-{descripcion}

# 6. Después del merge — limpiar rama local
git checkout develop
git pull origin develop
git branch -d feature/{PROYECTO}-{numero}-{descripcion}
```

---

## A.9 Flujo: Backend / Frontend — Hotfix

> Contexto: [git-conventions.md 4.1 Flujo de hotfix](git-conventions.md#41-backend--microservicio)

```bash
# 1. Crear desde main
git checkout main
git pull origin main
git checkout -b hotfix/{PROYECTO}-{numero}-{descripcion}

# 2. Corregir y commitear
git add {archivos}
git commit -m "hotfix({scope}): {descripcion}"

# 3. Abrir PR hacia main → merge

# 4. Crear tag en main
git checkout main
git pull origin main
git tag -a v{MAJOR}.{MINOR}.{PATCH} -m "hotfix: {descripcion}"
git push origin v{MAJOR}.{MINOR}.{PATCH}

# 5. Backmerge a develop (OBLIGATORIO)
git checkout develop
git pull origin develop
git merge --no-ff hotfix/{PROYECTO}-{numero}-{descripcion}
git push origin develop

# 6. Limpiar
git branch -d hotfix/{PROYECTO}-{numero}-{descripcion}
git push origin --delete hotfix/{PROYECTO}-{numero}-{descripcion}
```

---

## A.10 Flujo: Backend / Frontend — Release

> Contexto: [git-conventions.md 4.1 Flujo de release](git-conventions.md#41-backend--microservicio)

```bash
# 1. Crear desde develop
git checkout develop
git pull origin develop
git checkout -b release/{MAJOR}.{MINOR}.{PATCH}

# 2. Actualizar versión en el proyecto
# Editar package.json / .csproj / pyproject.toml / build.gradle

git add {archivo-de-version}
git commit -m "chore: bump versión a {MAJOR}.{MINOR}.{PATCH}"

# 3. Solo fixes de estabilización aquí (sin nuevas features)
# Si hay fixes: commitear normalmente

# 4. Abrir PR hacia main → merge

# 5. Crear tag en main
git checkout main
git pull origin main
git tag -a v{MAJOR}.{MINOR}.{PATCH} -m "release: v{MAJOR}.{MINOR}.{PATCH}"
git push origin v{MAJOR}.{MINOR}.{PATCH}

# 6. Backmerge a develop (OBLIGATORIO)
git checkout develop
git merge --no-ff release/{MAJOR}.{MINOR}.{PATCH}
git push origin develop

# 7. Limpiar
git branch -d release/{MAJOR}.{MINOR}.{PATCH}
git push origin --delete release/{MAJOR}.{MINOR}.{PATCH}
```

---

## A.11 Flujo: Base de Datos — Migración

> Contexto: [git-conventions.md 4.3](git-conventions.md#43-base-de-datos)
>
> Los scripts de migración siguen la nomenclatura y estructura de carpetas establecida en [sqlserver-naming-conventions.md 8](../database/sqlserver-naming-conventions.md#8-scripts-ddl-y-dml).

```bash
# 1. Crear desde develop
git checkout develop
git pull origin develop
git checkout -b migration/{PROYECTO}-{numero}-{descripcion}

# 2. Crear scripts en la estructura establecida
# migrations/ddl/{numero}_{accion}_{objeto}.sql
# migrations/rollback/ddl/{numero}_rollback_{accion}_{objeto}.sql

git add migrations/
git commit -m "migration(schema): {descripcion}

{Descripcion ampliada del cambio de esquema.}

Refs: {PROYECTO}-{numero}"

# 3. Probar localmente:
#    a) Ejecutar el script de migración en dev
#    b) Verificar resultado
#    c) Ejecutar el script de rollback
#    d) Verificar que revierte correctamente

# 4. Push y abrir PR hacia develop
git push origin migration/{PROYECTO}-{numero}-{descripcion}
# → El PR requiere aprobación del equipo DBA/Infra (ver CODEOWNERS A.7)

# 5. Después del merge — limpiar
git checkout develop
git pull origin develop
git branch -d migration/{PROYECTO}-{numero}-{descripcion}
```

---

## A.12 Flujo: Librería — Release con tag

> Contexto: [git-conventions.md 4.5](git-conventions.md#45-librería--paquete)

```bash
# 1. Crear rama de release desde develop
git checkout develop
git pull origin develop
git checkout -b release/{MAJOR}.{MINOR}.{PATCH}

# 2. Actualizar versión
# Editar package.json / .csproj / pyproject.toml

git add {archivo-de-version}
git commit -m "chore: bump versión a {MAJOR}.{MINOR}.{PATCH}"

# 3. Abrir PR hacia main → merge

# 4. Crear tag en main — dispara el pipeline de publicación
git checkout main
git pull origin main
git tag -a v{MAJOR}.{MINOR}.{PATCH} -m "release: v{MAJOR}.{MINOR}.{PATCH}

Cambios incluidos:
- {tipo}: {descripcion 1}
- {tipo}: {descripcion 2}"
git push origin v{MAJOR}.{MINOR}.{PATCH}

# El pipeline de CI/CD publica automáticamente al detectar el tag.
# NO publicar desde local.

# 5. Backmerge a develop (OBLIGATORIO)
git checkout develop
git merge --no-ff release/{MAJOR}.{MINOR}.{PATCH}
git push origin develop

# 6. Limpiar
git branch -d release/{MAJOR}.{MINOR}.{PATCH}
git push origin --delete release/{MAJOR}.{MINOR}.{PATCH}
```

---

## A.13 Flujo: Documentación

> Contexto: [git-conventions.md 4.4](git-conventions.md#44-documentación)

```bash
# Repositorio de documentación pura (sin develop)
git checkout main
git pull origin main
git checkout -b docs/{descripcion}

git add {archivos}
git commit -m "docs: {descripcion}"

git push origin docs/{descripcion}
# → Abrir PR hacia main

# Limpiar después del merge
git checkout main
git pull origin main
git branch -d docs/{descripcion}
```

```bash
# Repositorio mixto (código + docs) — usar develop como base
git checkout develop
git pull origin develop
git checkout -b docs/{PROYECTO}-{numero}-{descripcion}

git add {archivos}
git commit -m "docs: {descripcion}

Refs: {PROYECTO}-{numero}"

git push origin docs/{PROYECTO}-{numero}-{descripcion}
# → Abrir PR hacia develop
```
