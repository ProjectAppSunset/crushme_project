---
name: "repo-cleanup"
description: "Auditoría del repo: archivos obsoletos, huérfanos o innecesarios, artefactos trackeados, docs stale y, en proyectos nacidos de un template base (Vue o React), los restos del template que ya no deberían estar. Usar cuando se pide limpiar o auditar un repo. Read-only hasta que el operador aprueba; en los scaffolds base_* nunca busca restos del template."
---

# Repo Cleanup — Systematic File Audit

## Goal
Scan the repository for files that should be deleted, updated, or added to `.gitignore`. Produce a prioritized action list the user can approve before any destructive operation.

## Cómo invocar este skill

Gating ($output-protocol §4): si el operador pasó el argumento de alcance
(`backend`/`frontend`/`docs`/`template`/`all`) → ejecutar directo, sin menú. Si la
intención es clara por la sesión (p.ej. se venía hablando de basura en docs) →
proponer el comando en una línea y esperar confirmación. Sin argumento y sin
intención clara → UNA sola `AskUserQuestion` (Q1). Nunca preguntar en modo
fleet/headless/cron ni dentro de un barrido.

**Q1 — Alcance** (selección única — los alcances son excluyentes):

| label | description | preview |
|---|---|---|
| Todo el repo (`all`) *(Recommended)* | audita backend + frontend + docs; read-only, sólo reporta | `$repo-cleanup all` |
| Sólo backend | dead code Python, artefactos y configs huérfanas de `backend/` | `$repo-cleanup backend` |
| Sólo frontend | componentes/composables sin referencias, builds trackeados | `$repo-cleanup frontend` |
| Sólo docs | documentación stale, duplicada o vacía | `$repo-cleanup docs` |

**Qué NO se pregunta:** nada más — no hay flags ocultos (el único argumento es
el alcance). `template` (sólo la [categoría 7](#restos-del-template)) se tipea:
el picker no admite una quinta opción (§4), y `all` ya la incluye salvo que su
Paso 0 dé `n/a` (`backend`, `frontend` y `docs`, no). Los lotes de borrado no se
preguntan acá: se aprueban después del reporte, con lista y evidencia visibles
(ver `## Acciones disponibles`).

## What To Search For

### 1. Build & Test Artifacts Tracked in Git
- Coverage reports (`*.json`, `*.xml`, `*.lcov`, `htmlcov/`)
- Test result logs, screenshots, or trace files
- Compiled/transpiled output (`.pyc`, `__pycache__/`, `.nuxt/`, `.output/`, `dist/`)
- Lock files or `node_modules/` committed by mistake
- Database dumps, `.sqlite3` files, or SQL scripts that were one-time use

### 2. Dead Code & Unused Source Files
- Backend: Python modules not imported anywhere (check with `grep -r` for the module/function name)
- Frontend: Vue components, composables, stores, or utilities not imported in any file
- Test files for source code that no longer exists
- Management commands that were one-off scripts

### 3. Backup & Temporary Files
- Files with suffixes: `_old`, `_backup`, `_copy`, `.bak`, `.orig`, `.tmp`
- Files with prefixes: `tmp_`, `temp_`, `debug_`
- Misspelled or duplicate filenames
- Demo/seed data files left in the project root

### 4. Orphaned Configuration
- Config files for tools no longer in the project (e.g., old linter configs, unused Docker files)
- Lock files at the wrong directory level (e.g., root `package-lock.json` when frontend has its own)
- Empty directories tracked by git

### 5. .gitignore Gaps
- Files that are tracked but match patterns that _should_ be in `.gitignore`
- Patterns in `.gitignore` that are too narrow and miss variants (e.g., `tmp_cov*.json` but not `cov*.json`)

### 6. Stale Documentation
- Docs that reference outdated stack names, removed features, or superseded standards
- Duplicate documentation covering the same topic in different files
- Empty or near-empty doc files

<a id="restos-del-template"></a>
### 7. Restos del template base (sólo proyectos derivados)

Los proyectos del fleet nacen de `base_django_vue_feature` (Django + Vue) o de
`base_django_react_next_feature` (Django + Next.js) y suelen arrastrar piezas que el
producto ya no usa. Esta categoría las lista con su justificación. **No borra nada
sin la aprobación del operador.**

**Paso 0 — ¿aplica?** Una sola regla: si falla la condición 1 o la 2, la fila del
reporte queda `ℹ️ n/a` con el motivo y la categoría no busca nada; si sólo falla la
3, busca únicamente el bloque 7.g, que no depende del linaje.

1. **No es un template.** El nombre del repo (el directorio del clon principal, o el
   `repo=` de `session-worktree.sh status` en un worktree) no empieza con `base_`, y
   su `origin` no es el `git_repo:` de un `base_*` en
   `~/webapps/vps-ops-toolkit/projects.yml`. Las dos URLs se comparan como
   `owner/nombre`: `git@github.com:` y `https://github.com/` valen igual y el `.git`
   final no cuenta (la regla de `norm_slug` en
   `~/webapps/vps-ops-toolkit/scripts/maintenance/resolve-work-coordinate.sh`). En un
   `base_*` no corre nunca, ni con argumento explícito.
2. **Es un proyecto vigente:** `status: active` o `dev_only`, sin `lifecycle: legacy`;
   un repo fuera del registro cuenta como vigente. Un `scaffold` sin prefijo `base_`
   es un onboarding en curso ($integrate-new-project, Paso 1d): se espera a que
   cierre.
3. **Desciende del template.** Basta una señal: el commit que trajo el código
   (`git log --diff-filter=A --format='%h %ad %s' --date=short -- backend/manage.py`)
   dice «based on base_django_…»; existe `backend/base_feature_project/` o
   `backend/base_feature_app/`; o su `django_settings_module` empieza con
   `base_feature_project.`.

Ese commit es el **commit de copia**: la foto del template tal como llegó, y la
referencia de lo que sigue. Con `next.config.*` la variante es React y el inventario
de $pre-staging-cleanup sirve de guía; con `vite.config.*` y `.vue` es Vue, sin
inventario en el toolkit: manda la copia. $pre-staging-cleanup y
$new-project-setup son local-skills: si el repo no las tiene instaladas, se leen
en `~/webapps/vps-ops-toolkit/workflows/.local-skills/.claude/<skill>.md`.

| Bloque | Qué incluye | Acción por defecto |
|---|---|---|
| 7.a Textos | «Base Django React Next Feature», «Base Django Vue Feature»; `metadata` y `<title>`; el `name` de `package.json`; i18n y correos con la marca del template; el Client ID de OAuth del template ($new-project-setup, fase S6) | ACTUALIZAR |
| 7.b Nombres en textos y config | jobs de CI, units y rutas `base_django_*` o `/home/ryzepeck/webapps/base_*`, `BACKUP_STORAGE_PATH` del template en `.env.example`, menciones en docs | ACTUALIZAR |
| 7.c Rutas y pantallas demo | catálogo, productos, ventas, blog, carrito, checkout, manual, backoffice, admin-login, dashboard demo, con sus urls, vistas, stores, componentes y links | BORRAR si no se adoptaron |
| 7.d Datos demo | generadores de modelos demo (`create_blogs`, `create_products`, `create_sales`), fixtures e imágenes de ejemplo | BORRAR con su feature |
| 7.e README y docs | secciones «Reference Projects» y «Customization» del README; `backend/CLAUDE.md`, `frontend/CLAUDE.md` y guías sin tocar desde la copia; el Memory Bank que describe el template ($methodology-setup) | ACTUALIZAR o BORRAR |
| 7.f Configuración de ejemplo | `scripts/systemd/` y `scripts/nginx/` del repo cuando la real vive en el toolkit (`config/systemd/`, `config/nginx/`) | BORRAR o dejar un puntero |
| 7.g Boilerplate del generador | assets de create-next-app (`next.svg`, `vercel.svg`, `file.svg`, `globe.svg`, `window.svg`); README de create-vue o create-next-app; un módulo sombreado por un paquete homónimo (`urls.py` junto a `urls/`, `tests.py` junto a `tests/`) | BORRAR |
| 7.h Tests del template | tests y specs de lo demo | se van con su código, nunca antes |

**Nunca es resto, aunque venga de la copia:**
- lo que el fleet sincroniza: `.claude/`, `.agents/`, `.codex/`, los bloques
  `fleet-base` de `CLAUDE.md` (`AGENTS.md` es un render: se corrige en el bloque
  `project-specific` de `CLAUDE.md`) y el core de calidad (`scripts/quality/`,
  `scripts/test_quality_gate.py` y sus pares, `frontend/scripts/ast-parser.cjs`, los
  `docs/*_STANDARDS.md`, `.testquality.yml`, `.junk-baseline.json`);
- el contrato de runtime y de onboarding: `/api/health/`, `create_fake_data` y
  `delete_fake_data` ($fake-data-refresh), `silk_garbage_collect`, el Staging
  Phase Banner y `.env.example` (siembra el `.env` canónico; sus valores del template
  se ACTUALIZAN);
- `django_attachments/`, auth, `User`, `PasswordCode` y el resto de la lista
  PRESERVAR de $pre-staging-cleanup;
- las migraciones aplicadas y los `__init__.py`.

**Siempre PROPUESTA APARTE, nunca un borrado** (se listan con sus puntos de
contacto, fuera de todo lote):
- el paquete `base_feature_project` y el app `base_feature_app`: el paquete está
  sellado como `django_settings_module` en `projects.yml`, en las units del toolkit y
  en la generación de Integrity; el label del app vive en `django_migrations`.
  Renombrar es un cambio coordinado de repo, toolkit, Integrity (con password del
  operador) y deploy;
- el nombre de Huey fijo en el código: cambiarlo deja huérfanas las tareas
  encoladas en Redis con el nombre viejo;
- un modelo demo con migración aplicada: quitarlo borra tablas con datos reales y va
  con backup y deploy transaccional.

**Evidencia de cada elemento** (se mide, no se supone):
- **origen**: vino en la copia, llegó después desde el template o es del proyecto;
- **cambios**: `git log --format='%h %ad %s' --date=short -- <path>`. Un cambio
  posterior no prueba adaptación: si sólo lo tocaron pasadas transversales (vuln,
  perf, QA, lint, sync), sigue siendo candidato;
- **referencias**: `git grep -l -F '<símbolo o ruta>' -- . ':!<path>' ':!*.env*'`,
  fuera de sus tests; se listan archivos, nunca líneas;
- **contrato o datos**: si es contrato del fleet o tiene migración aplicada.

**Quién arma la lista:** un subagente de sólo lectura por capa (backend, frontend,
docs y config) con esta consigna; el conductor consolida y numera T1…Tn. En Codex,
la misma consigna en serie.

> «Proyecto `<repo>`, variante `<vue|react>`, commit de copia `<sha>`. Recorré
> `<capa>` y devolvé, por cada elemento del template presente: path, bloque, origen,
> commits con su asunto, referencias (`git grep -l`), riesgo y una línea de por qué
> no debería estar o por qué se queda. No edites, no borres, no corras `manage.py` y
> no imprimas valores de `.env`.»

**La lista**, una por repo. Riesgo bajo = asset, doc o config sin referencias; medio
= código o tests sin referencias, o con arrastre; alto = modelos, API, settings o
nombres sellados, que van al final y fuera de todo lote.

| ID | Path(s) | Bloque | Evidencia | Por qué no debería estar | Riesgo | Acción | Arrastre |
|---|---|---|---|---|---|---|---|

`Acción`: BORRAR, ACTUALIZAR, DECIDIR (el operador sabe si se adoptó), CONSERVAR o
PROPUESTA APARTE. `Arrastre`: las ediciones que exige el borrado (imports, `include()`
de urls, admin, links, el flow en `flow-definitions.json`, `.junk-baseline.json`, que
se regenera y sólo achica, y el CI que nombre un test borrado).

**Aprobación obligatoria.** Nada se borra sin que el operador lo apruebe en la
conversación, por ID (`apruebo T1 T3`) o por un grupo de IDs que la propia lista
nombró. Ningún lote del menú (HIGH, MEDIUM o «TODO») incluye esta categoría. No
cuentan el silencio, un «ok» anterior ni la aprobación de otro lote o de otra
sesión. En headless, cron o barrido, la categoría sólo reporta.

**Aprobado, se borra en el worktree de sesión y con PR:**
1. `bash ~/webapps/vps-ops-toolkit/scripts/maintenance/session-worktree.sh create chore template-cleanup`
   y `EnterWorktree path=$HOME/webapps/.wt/<repo>/template-cleanup`; nunca en el clon
   principal.
2. Un commit por bloque aprobado: `git rm` de lo aprobado más su arrastre, nada más
   (`chore(cleanup): remove template leftovers (T1, T3)`).
3. Se verifica el subset afectado (lint, typecheck, tests de lo tocado). Nunca
   `manage.py migrate` desde el worktree: su `backend/.env` apunta a la base real.
4. `git push -u origin <rama>` y `gh pr create --base <base>` con los `branch=` y
   `base=` de `session-worktree.sh status`, literales; el body lleva `Sesión:`,
   `Intención:` y la tabla aprobada. $pr-green lo lleva a CI verde y la sesión
   para: no mergea. Cierre con $all-in-base `--check-only`.
5. Producción cambia sólo con el deploy normal ($deploy-and-check, del operador).

Un defecto del template mismo se anota en Next steps como «arreglar en el template»;
esta corrida no toca el scaffold.

## Workflow

### Phase 1 — Inventory
1. List the full project tree (directories only, then key file types).
2. Run `git ls-files` to get the canonical list of tracked files.
3. Cross-reference tracked files against `.gitignore` patterns to find mismatches.
4. Identify files by age: `git log --diff-filter=A --format='%ai' -- <file>` for files that haven't changed since creation.

### Phase 2 — Dead Code Detection
5. For each composable, store, utility, and component in `frontend/`:
   - `grep -r "<name>" frontend/` excluding the file itself and its test.
   - If zero external references, flag as potentially unused.
6. For each non-standard Python module in `backend/` (exclude models, views, serializers, migrations, admin, urls, apps, conftest, tests):
   - `grep -r "<module_name>" backend/` excluding the file itself.
   - If zero external references, flag as potentially unused.

### Phase 3 — Artifact Detection
7. Search for large files that look like generated output:
   ```bash
   git ls-files -z | xargs -0 ls -la --sort=size 2>/dev/null | tail -30
   ```
8. Search for files matching artifact patterns:
   ```bash
   git ls-files | grep -iE '\.(bak|orig|tmp|old|backup|copy|log)$'
   git ls-files | grep -iE '(debug_|tmp_|temp_|_backup|_old|_copy)\.'
   ```
9. Look for coverage/test artifacts in unexpected locations:
   ```bash
   git ls-files | grep -iE '(coverage|cov_|test-results|test-reports)\.'
   ```

### Phase 3b — Restos del template (sólo con `all` o `template`)
- Correr el Paso 0 de la [categoría 7](#restos-del-template) y sumar al reporte su lista T (completa, o sólo 7.g si falla únicamente la condición 3).

### Phase 4 — Report
10. Classify every finding into one of these categories:

| Priority | Category | Action |
|----------|----------|--------|
| HIGH | Tracked artifacts (coverage, builds, dumps) | `git rm` + update `.gitignore` |
| HIGH | Files containing secrets or credentials | Immediate removal |
| MEDIUM | Unused source code (dead imports) | Delete file + its test |
| MEDIUM | Orphaned configs or root-level leftovers | Delete |
| LOW | Stale documentation | Flag for review |
| LOW | Empty directories | Remove |
| TEMPLATE | Restos del template (cat. 7) | Lista T con evidencia → aprobación por ID → `git rm` en el worktree de sesión + PR; fuera de los lotes del menú |
| INFO | Propuestas aparte de la cat. 7 | Plan separado; nunca en un lote |

11. Present findings as a table with: file path, size, last modified, category, recommended action.

## Rules
- **Read-only until user approves.** Do not delete, edit, or move any file during the audit. Present the report and wait for confirmation.
- Do not flag files in `.agents/`, `.claude/`, or `.codex/` as obsolete — these are intentional compatibility layers. `.windsurf/` is retired fleet-wide and must be reported as obsolete if present.
- Do not flag Django migrations as obsolete unless they are clearly broken merge migrations.
- Do not flag `__init__.py` files as empty/unnecessary — they mark Python packages.
- Verify dead code claims: a file is only "unused" if it has zero imports AND is not referenced in URL configs, management commands, or template tags.
- When recommending `.gitignore` updates, show the exact lines to add.
- Categoría 7: nunca en un `base_*`; nada se borra sin aprobación por ID; ningún lote del menú la incluye; las propuestas aparte (paquete y app del template, nombre de Huey, modelos demo con migración aplicada) nunca son un `git rm`.

## Acciones disponibles

Tras el reporte, si la sesión es interactiva y NO hubo flags explícitos
(reglas de gating de $output-protocol §4), ofrecer vía AskUserQuestion:

| Opción (label) | description (costo/efecto) | preview (comando exacto) |
|---|---|---|
| Aplicar lote HIGH (con la lista y evidencia mostradas) | ejecuta el `git rm` del lote HIGH tal cual quedó listado arriba, nada más | `git rm <paths del lote HIGH mostrado>` |
| Aplicar patch .gitignore | agrega al `.gitignore` las líneas exactas mostradas en el reporte | `cat >> .gitignore  # líneas del patch mostrado` |
| Aplicar TODO lo propuesto (HIGH+MEDIUM, re-mostrando evidencia) | borra ambos lotes, nunca la categoría 7; re-muestra lista + evidencia de MEDIUM antes de tocar | `git rm <lotes HIGH+MEDIUM>` + patch `.gitignore` |
| Nada — sólo el reporte | cierra sin ejecutar nada | — |

Los lotes MEDIUM/LOW jamás se aplican sin re-mostrar la evidencia; respeta los
carve-outs de la skill (migrations, `__init__.py`, systemd/cron activos).

Los restos del template (categoría 7, prioridad `TEMPLATE`) no tienen fila —el menú
ya tiene las cuatro de §4— y ningún lote del menú los incluye: se aprueban por ID
(`apruebo T1 T3` o `ninguno`) y se borran con el flujo de worktree + PR de esa
categoría.

## Output final

Reportar siguiendo $output-protocol. Plantilla específica de esta skill
(auditoría read-only; ✅ limpio · ⚠️ hallazgos a revisar · ❌ HIGH: secretos o
artefactos trackeados):

```markdown
🟢 repo-cleanup OK — <N> archivos auditados, 0 hallazgos
✨ Todo en orden — no hay acciones pendientes.

| Dimensión | Estado | Detalle |
|---|---|---|
| Artefactos build/test trackeados | ✅ | coverage/dumps/__pycache__/dist no versionados |
| Dead code (backend + frontend) | ✅ | 0 módulos/componentes sin referencias |
| Backup / temp files | ✅ | sin *.bak/_old/_copy/tmp_ |
| Config huérfana | ✅ | sin lockfiles mal ubicados ni dirs vacíos |
| Gaps en .gitignore | ✅ | patrones cubren variantes |
| Docs stale | ✅ | sin refs a stack removido ni duplicados |
| Restos del template | ✅ | sin restos · `ℹ️ n/a` en `base_*` o no vigente · sin linaje: sólo 7.g |
```

Cuerpo bajo la tabla: **action items** por prioridad HIGH→LOW (path · razón ·
comando `git rm`), **patch `.gitignore`** exacto, y tamaño reclamable total.
Si hay hallazgos, reemplazar el ✅ de la fila por ⚠️ o ❌ (HIGH = secretos o
artefactos trackeados) y omitir la línea ✨. Con >15 hallazgos, anteponer
`### Resumen ejecutivo` (conteo) + `### Top 3 acciones prioritarias`.
**Read-only hasta aprobación del operador**: la skill nunca borra.

## Next steps
- (operador) aprobar el action list antes de cualquier `git rm` / delete
- aplicar el patch de `.gitignore` mostrado arriba
- (operador) aprobar por ID los restos del template; las propuestas aparte van por separado
