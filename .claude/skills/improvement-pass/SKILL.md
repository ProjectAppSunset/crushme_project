---
name: improvement-pass
description: "Orquesta una ronda de mejora de UN proyecto en seis frentes —seguridad, mantenibilidad, observabilidad, rendimiento, responsividad y QA—, con un subagente por frente, un crítico de cobertura y un verificador independiente por candidato. Usar para una ronda de mejora transversal. Default aplica sólo lo que supera el umbral, con rama y PR por frente; --check sólo analiza; --refresh registra; --review3 revisa anteriores. No hacer nada es válido."
argument-hint: "[proyecto] [--apply|--check|--refresh|--review3] [--fronts=security,maintainability,observability,performance,responsive,qa] [--module=<alcance>]"
allowed-tools: Bash, Read, Edit, Write, Grep, Glob, Agent, SendMessage, AskUserQuestion, EnterWorktree, ExitWorktree
---

# Improvement pass — seis frentes, crítico de cobertura y validación independiente

Sos el ORQUESTADOR de una ronda de mejora de UN proyecto: creás subagentes, repartís
el trabajo, resolvés conflictos e integrás; no implementás ningún frente. La política y
el motor son canónicos:
`$HOME/webapps/vps-ops-toolkit/workflows/improvement/IMPROVEMENT_STANDARDS.md` (el
contrato) y `scripts/improvement/improvement-ledger.sh`. No reimplementar selección,
huellas de contexto, validación ni persistencia.

## Regla cero

Nunca implementar el trabajo de un frente. Sin subagentes, **detenerse y avisar al
operador** (contrato, «Sin subagentes, sin verificador o sin crítico»), sin recorrer
los frentes en secuencia como sustituto. Son del orquestador, y no cuentan como
trabajo de un frente: Git y los registros, `compartido` (§3) y la conducción de
[[qa]], cuyas pruebas escriben sus roles.

## Cómo invocar este skill

Sin menú por diseño (§4): el alcance y el modo se resuelven de la invocación y del contexto; la delegación conserva la selección del conductor.

Sin picker por diseño: sin flags se **diagnostica, valida, registra, aplica lo que
supera el umbral y se valida con QA** en el proyecto del cwd/contexto; `--apply` es ese
mismo modo explícito. `--check` diagnostica y valida sin escribir nada (ni worktrees,
reportes, ledger ni memoria). `--refresh` diagnostica, valida y registra, sin aplicar.
`--review3` revalida hasta tres candidatos ya trabajados **por frente**, sin cambios
funcionales salvo `--apply`. `--check`, `--refresh` y `--review3` son excluyentes;
`--apply` sólo combina con `--review3`. La lista de frentes del `argument-hint`
restringe la ronda (default: los seis) y `--module` delimita el alcance; con cualquiera
la ronda es **parcial** y se declara. No hay modo fleet ni deploy.

Qué NO se pregunta: los frentes, el umbral, la cantidad de subagentes, las ramas ni
los menús de cada pasada. Un proyecto no identificable o una decisión de negocio
imprescindible sí exige aclaración, mientras el análisis sigue. Un aviso de parada no
pide permiso para seguir gastando trabajo.

## Roles y mecanismo de subagentes

| Rol | Quién | Escribe | Cuántos |
|---|---|---|---|
| Orquestador | esta sesión | Git, worktrees, ledger, registro, reportes | 1 |
| Diagnosticador | subagente fresco por frente | nada (devuelve JSON) | 1 por frente presente |
| Crítico de cobertura | subagente fresco | nada (devuelve JSON de huecos) | 1 por ronda |
| Verificador | subagente fresco que no encontró el candidato | nada en el proyecto | según el tope del contrato |
| Revisor fresco | otro subagente fresco | nada | sólo modalidad `standard` |
| Implementador | un subagente nuevo por frente aprobado, lanzado desde su worktree | sus archivos aprobados, sin commit | 1 por frente aprobado |
| Roles de [[qa]] | los despacha el orquestador al conducir [[qa]] | tests, según [[qa]] | por rama |

- **Herramientas** (contrato, «Herramientas por host»): en Claude Code, `Agent` (tipo
  `general-purpose`) con un mensaje **autosuficiente**, continuado con `SendMessage`; en
  Codex, `spawn_agent`/`send_input` con los parámetros medidos; en un VPS del fleet,
  sólo `Agent`. Un subagente no lanza subagentes. Nunca `isolation: "worktree"`.
- **Concurrencia:** diagnosticadores en un solo mensaje; a lo sumo **un agente
  ejecutando pruebas a la vez**, bajo el candado común del contrato.
- **Frente N/A** demostrado por el preflight (sin frontend ⇒ responsividad): no se
  lanza su diagnosticador y se reporta N/A con su evidencia, nunca como MADURO.

## 0. Contexto, ronda, modo y árbol de lectura

Resolver proyecto/codebase, coordenada y fecha del host, nunca de memoria. Pre-entry,
desde el clon principal (sólo lectura):

```bash
# pre-entry: corre en el clon principal, antes de EnterWorktree
bash ~/webapps/vps-ops-toolkit/scripts/maintenance/resolve-work-coordinate.sh --check <proyecto>
```
```bash
# pre-entry: corre en el clon principal, antes de EnterWorktree
bash ~/webapps/vps-ops-toolkit/scripts/qa/qa-agent.sh --preflight <proyecto>
```

`host_status=wrong-host`, coordenada ambigua o falta de permiso real bloquean la
aplicación; el diagnóstico de `--check` puede seguir, declarado. `BASE` = la release
con `pr_state=single`; si no, `main`/`master`. No tocar ni stashar el clon principal.

**Id de ronda** `imp<ddmm>r<n>`: `n` es 1 más el mayor `r<n>` del día entre los
worktrees (`session-worktree.sh list --porcelain --offline`), las ramas locales
(`git branch --list`) y las remotas (`git ls-remote --heads origin`) que calzan
`improve/<DDMMYYYY>-imp<ddmm>r*`. Se lee desde el clon principal antes de entrar y se
escribe literal. Slugs: `imp<ddmm>r<n>-compartido` e `imp<ddmm>r<n>-<frente>`, con el
frente en español (`seguridad`, `mantenibilidad`, `observabilidad`, `rendimiento`,
`responsividad`, `qa`).

**Árbol de lectura** (contenido exacto de `origin/<BASE>`):

- Default/`--apply`: crear el worktree del orquestador y entrar en él (Claude
  `EnterWorktree` con `path`; Codex `cd`); es el único árbol donde un verificador puede
  ejecutar algo, y adentro `git rev-parse HEAD` da el **SHA base de la ronda**.

  ```bash
  # pre-entry: corre en el clon principal, antes de EnterWorktree
  bash ~/webapps/vps-ops-toolkit/scripts/maintenance/session-worktree.sh create improve imp<ddmm>r<n>-compartido
  ```
- `--check`, `--refresh` y `--review3` sin `--apply`: sin worktree; el clon principal
  se lee sin ejecutar nada, se declara su `HEAD` contra `origin/<BASE>` y los
  verificadores trabajan sólo con la fuente.

**Modo de ramas.** Default: **por frente, secuencial** (una rama, un worktree y un PR
por frente aprobado), sólo si (1) el `CLAUDE.md` (en Codex, el `AGENTS.md`) de
`compartido` contiene la marca literal `<!-- improvement-pass: ramas-por-frente -->`
(`grep -cF` ≥ 1) y (2) pasa la sonda de arranque: `ExitWorktree` con `action: "keep"`,
`EnterWorktree` con `path` de vuelta a `compartido` y la escritura del primer archivo
de despacho. Si falta una, la ronda corre en el **modo de una rama** y lo declara:
`compartido` es la única rama, con hasta tres candidatos en total, un commit por
candidato, un PR y un cierre de QA (ronda del motor `<ronda>-compartido`). El modo
paralelo no está habilitado.

**Toolkit fijado.** En un host con el motor de integridad, el ledger y los reportes van
al clon privado del toolkit (contrato, «Registros, secretos y dónde se escriben»),
actualizado por fast-forward antes de leer el ledger: `IMPROVEMENT_LEDGER_DIR` apunta a
su `config/improvement-ledger` y el `--report-dir` de `qa-agent.sh`, a su `docs/audits`.

Leer `improvement-ledger.sh --show <proyecto> --projdir=<árbol literal>` y el registro
de la ronda anterior del codebase: candidatos previos por frente y huellas
`rejected`, `needs_validation` y `deferred` vigentes. Con eso, armar:

- **CONTEXTO DEL PROYECTO**, igual para todos: stack y convenciones (bloque específico
  de su `CLAUDE.md`/`AGENTS.md`), capas, motor de DB y app alcanzable (del preflight),
  comandos literales de prueba y las guardas: nunca `manage.py migrate` (el `.env`
  enlazado apunta a producción), nada contra servicios desplegados ni datos reales,
  ninguna instalación de dependencias, ningún secreto ni contenido de clientes.
- **IMPROVEMENT_CONTEXT** por frente, según el contrato común.

Antes de despachar, el orquestador corre en `compartido` (con el toolkit fijado, más
`--report-dir=<docs/audits del clon privado>`) y pasa la salida literal al frente QA;
sin worktree, corre sobre el clon principal con `--no-report` en lugar de `--projdir`:

```bash
bash ~/webapps/vps-ops-toolkit/scripts/qa/qa-agent.sh --check <proyecto> --projdir=<worktree compartido literal> --run-id=<ronda>-qa
```

## 1. Diagnóstico — un subagente por frente, sólo lectura

La tabla «Frentes y efecto mínimo» del contrato fija skill, estándar y vara de cada
frente. Cada diagnosticador sigue su skill en modo delegado: [[security-pass]] (con
[[vuln-audit]] en modo auditoría), [[maintainability-pass]], [[observability-pass]],
[[perf-pass]], [[responsive-pass]] y [[qa]] en su modo «frente QA».

- **Frente QA:** diagnostica la deuda de tests **preexistente**, con una sola lista:
  flows `junk-only`, flows P1/P2 faltantes, clases negativas faltantes y duplicados.
  Los drafts sin validar no son candidatos, y un test inestable sólo lo es con
  reproducción local (si no, `needs_validation`). Parte de la salida de `qa-agent` que
  le pasa el orquestador y no despacha roles; si hay que escribir el flow-map o el
  Memory Bank, lo hace un rol subagente.
- **Seguridad:** el orquestador copia literales en el despacho SECURITY_REFERENCE §1 y
  los bloques `S-*` seleccionados.
- No usar `perf-scout`: su permiso de escritura rompería el writer único.

En `--review3`, cada subagente reexamina contra la fuente actual los candidatos de
`improvement-ledger.sh --review3` de su frente (`mode: review`); sus veredictos pasan
por §2 como los nuevos.

Cada mensaje se guarda antes de despachar («Despacho y PR de cada frente» del
contrato) y lleva, en orden: el `IMPROVEMENT_CONTEXT` del frente (`mode: discover`, o
`check`); el CONTEXTO DEL PROYECTO; el frente, su skill y su vara; los candidatos
previos del frente (ID, `root_cause`, estado, decisión) y las huellas `rejected` que no
se reproponen sin evidencia nueva; el principio rector y la instrucción de
diagnóstico, textuales del contrato; la adaptación del fleet; y el contrato de salida.

Adaptación del fleet (en el mismo mensaje):

- Llevá **al menos tu causa más fuerte** hasta un veredicto con evidencia, aunque
  concluyas MADURO o NO VALE LA PENA; atacarla no obliga a cambiar código.
- Dos estados más, para no forzar un falso MADURO: BLOQUEADO y FALTA EVIDENCIA. MADURO
  y NO VALE LA PENA exigen haber revisado el alcance completo.
- Escala: ALTO = `material` (u obligación), MEDIO = `minor`, BAJO = `marginal` y una
  línea de descarte.
- Sólo fuente y evidencia local acotada: no escribas ni ejecutes nada en el clon
  principal ni contactes servicios desplegados o datos reales.
- Lo de otro frente va en `out_of_front`; una frontera de tu frente que no revisaste es
  una unidad tuya `deferred` con `pending: frontera encontrada sin revisar`.
- Registrá la cobertura: qué unidades examinaste y cuáles no.

Contrato de salida: **un solo objeto JSON, sin prosa**, con la forma del contrato
(«Flujo», paso 1): `front`, `state`, `candidates` (≤6, con `proposed_verdict`;
`propose: true` en tres como máximo), `discards`, `coverage` y `out_of_front`. Un
retorno malformado recibe un único pedido de corrección; si vuelve a fallar, el frente
se reporta como fallido y su trabajo no se absorbe.

## 1b. Crítico de cobertura

Consolidado el diagnóstico y antes de verificar, lanzar el crítico según «Crítico de
cobertura» del contrato (nunca recibe la conclusión de un verificador). Sus huecos
dejan unidades `deferred` y el frente en FALTA EVIDENCIA; sus objeciones a un descarte
lo pasan al conjunto a verificar. Sin segunda ola: la ronda siguiente los trabaja
primero.

## 2. Validación independiente

Seguir el «Protocolo de validación de candidatos» del contrato. Acá sólo va la
mecánica del orquestador:

1. Consolidar por huella y después por causa raíz. Una causa compartida tiene un frente
   dueño; los demás quedan como efectos a verificar y no ocupan otro cupo.
2. Llenar el conjunto a verificar en el orden del tope del contrato, declarado en el
   plan de despacho; lo recortado va al ledger como `needs-evidence` «sin calificar: sin
   verificar por tope» y al reporte.
3. Un verificador fresco por candidato; `quick`, y `standard` automática para
   `mandatory` y `bloqueante`.
4. Ejecutar sólo dentro de «Límites de la evidencia y de la ejecución» del contrato: en
   DEV, chequeos acotados en `compartido` bajo el candado común; en un VPS con motor,
   sólo lectura salvo el OK explícito del operador para esa ronda; sin worktree, nada
   (lo que exige ejecución queda `needs_validation` con su plan local exacto).
5. El orquestador es el único writer del registro y lo valida con
   `improvement-ledger.sh --validate-record` antes de escribirlo. Sin veredicto válido,
   un candidato no se aplica ni deja declarar suficiente su frente.

Sólo `confirmed` pasa a la evaluación de valor; `needs_validation` entra con
`benefit: unknown` y sin calificación, con su plan en el reporte; `rejected` queda en el
registro. La verificación final de lo aplicado es la del contrato.

## 3. Consolidar y decidir

Registrar con `--refresh` los candidatos validados y sólo las scopes que el ledger de
cobertura permite marcar `complete: true` (en `--check`, con `--check`). Seleccionar
**por frente**, hasta tres (una ronda del motor por rama):

```bash
bash ~/webapps/vps-ops-toolkit/scripts/improvement/improvement-ledger.sh --select <proyecto> --projdir=<árbol literal> --fronts=<front>
```

Revisar cada propuesta con escepticismo: el orquestador puede rechazar, con la razón
registrada, un beneficio mal demostrado aunque subagente y verificador lo aprueben.
Nunca aprueba sin `confirmed` ni bajo el umbral, y una obligación no se rechaza: si no
puede hacerse, queda `blocked` con su siguiente acción.

Si ningún frente queda VALE LA PENA: avisar con el resumen, cerrar los subagentes,
registrar la ronda (§6), salir con `ExitWorktree` y retirar `compartido` si no tiene
commits; sin ramas ni PRs. `--check` y `--refresh` terminan acá siempre.

```bash
bash ~/webapps/vps-ops-toolkit/scripts/maintenance/session-worktree.sh remove imp<ddmm>r<n>-compartido
```

Cerrar los subagentes de los frentes que no quedaron VALE LA PENA. Después:

- **Un dueño por archivo.** Si dos frentes piden el mismo, decide el orquestador: el
  frente de cuya causa se trata; el otro lo pide como solicitud.
- **`compartido`** recibe sólo cuatro categorías, todas del orquestador: dependencias
  (únicamente por [[vuln-audit]] con `VULN_CANDIDATE`), lockfiles, migraciones
  aditivas (`makemigrations` y `sqlmigrate`, nunca `migrate`) y configuración global.
  Un cambio compartido que necesita un solo frente va en la rama de ese frente, en
  commit propio; si lo necesitan varios, va a `compartido` y los dependientes quedan
  `blocked` con `depends_on` hasta que aterrice. No se apilan PRs de frente sobre
  `compartido`.
- **Un worktree por frente aprobado**, creado desde `compartido` con un comando simple:

  ```bash
  bash ~/webapps/vps-ops-toolkit/scripts/maintenance/session-worktree.sh create improve imp<ddmm>r<n>-<frente>
  ```

  Si el slug ya existe, ese frente se detiene y se reporta: nunca se reutiliza un
  worktree ajeno.

## 4. Autorizar la implementación

Por frente aprobado: `ExitWorktree` con `action: "keep"` → `EnterWorktree` con `path`
hacia su worktree → ahí, un **implementador nuevo** con un mensaje autosuficiente, que
hereda ese cwd. Nunca el cambio directo entre worktrees `.wt` ni `action: "remove"`. Si
adentro `git rev-parse HEAD` difiere del SHA base de la ronda, repetir `--select` y
revalidar los afectados antes de autorizar.

El mensaje lleva las mejoras aprobadas (ID, huella y plan), la lista exacta de
archivos, su `IMPROVEMENT_CONTEXT` con `mode: apply`, `round_id: <ronda>-<frente>`,
`candidate_ids` y `allowed_paths`, y la instrucción de implementación textual del
contrato.

## 5. Supervisión e integración por rama

Desbloquear con `SendMessage`/`send_input`. Un subagente que falla dos veces se
reporta al operador y su trabajo no se absorbe; un candidato bloqueado no detiene a los
demás ni se rellena el cupo. Por rama (`compartido` primero si otros dependen de él;
luego en el orden de la tabla de frentes), parado en su worktree:

1. **Alcance.** `git status --porcelain --untracked-files=all` debe quedar contenido en
   los archivos aprobados y los tests del cierre. Si un frente se salió de su alcance,
   queda en pausa, sin commit ni PR y con el diff intacto, y se avisa al operador.
2. **QA de cierre**, uno por rama, con los guiones de los frentes que contiene: [[qa]]
   delegado con el contexto de la rama (`round_id: <ronda>-<frente>`, sus
   `candidate_ids` y `allowed_paths` extendidos a los tests que justifique su
   Architect); autoría y correcciones acá, verificación final en el paso 4. Con el
   motor presente, cada corrida lleva el `TMPDIR` literal bajo `~/.cache` y el candado
   común. Un rojo habilita el fix loop acotado de QA, nunca bajar requisitos, agrandar
   el baseline de basura ni bajar severidad.
3. **Commit** del orquestador, selectivo (archivos aprobados y tests del cierre):
   Conventional Commits en inglés con el frente como scope, en lo posible uno por
   candidato, y en el cuerpo qué cambió, por qué, cómo se verificó, la calificación y
   el registro. Identidad git repo-local si `git var GIT_COMMITTER_IDENT` falla.
4. **Registro y verificación.** `--record` de los candidatos y de la ronda
   `<ronda>-<frente>` (uno a tres candidatos, SHA40 real); el Verifier de QA ejecuta
   sobre ese SHA y el manifest va a `--record-qa` (con el interino del contrato si el
   motor del host no admite el frente). Un arreglo posterior cambia el SHA: actualizar
   la ronda y repetir lo afectado. E2E sin app que sirva ese SHA deja `qa-pending`.
5. **PR y CI** por [[pr-green]] en ese worktree: abre el PR con
   `Intención: improvement-pass <ronda> — frente <frente>`, espera el CI, arregla sólo
   lo atribuible y PARA en verde. Nunca merge.

`compartido`: el orquestador aplica las «Solicitudes al orquestador» que superan el
mismo umbral, después de validarlas como cualquier candidato. Si [[vuln-audit]] exige
árbol limpio, antes se commitea lo propio ya revisado. Si el cambio altera
comportamiento, lleva su propio cierre de QA.

## 6. Registro y cierre

- **Reporte de ronda**:
  `docs/audits/<YYYY-MM-DD>-<proyecto>-improvement-pass-<alcance>-<ronda>.md`, con el
  registro validado al lado en un `.json` del mismo nombre: tabla por frente, cambios y
  PRs, `needs_validation` con su plan, descartes, huecos del crítico, solicitudes,
  cobertura y un anexo con los mensajes. En `--check`, todo va en la respuesta.
- Los registros del toolkit van por su flujo trunk, en un commit por ronda, nunca
  mezclados con el proyecto; con el toolkit fijado, en el clon privado, con un push sin
  propagar y sin commits sin publicar al cerrar (`ahead=0`).
- Cerrar los subagentes. Los worktrees con PR quedan hasta el merge (los retira
  [[all-in-base]]); sólo `compartido` sin commits se retira ahora.
- Cierre: [[all-in-base]] `--check-only`, una unidad por worktree de frente. La sesión
  no mergea: drenar los PRs con `/merge-queue` es del operador.

## Output final

Usar [[_output-protocol]] con una fila por frente y las filas Crítico, Validación, QA
de cierre, Entrega y Registro. Cada fila de frente lleva su estado (MADURO, NO VALE LA
PENA, VALE LA PENA, BLOQUEADO, FALTA EVIDENCIA o N/A), qué se hizo y el PR si hubo
cambios. Después de la tabla: las **descartadas por bajo retorno**, una línea cada una
(huella · título · impacto/esfuerzo/riesgo · motivo); las **solicitudes aplicadas en
`compartido`**; y los **riesgos reales abiertos** (`needs_validation` con su plan,
huecos del crítico y obligaciones `blocked`).

Distinguir «no conviene seguir en este alcance» de «no se revisó». En `--check`, no
presentar commits, reportes ni PRs como hechos ni prometer una recepción ProjectApp
que no se ejecutó. Next steps: `(operador) /merge-queue` (sólo como texto) y
[[all-in-base]] `--check-only`.

Veredictos: 🟢 cada frente con estado, cobertura, crítico y validación cerrados y PRs
en verde, o ningún frente vale la pena y la evidencia lo muestra · 🟡 FALTA EVIDENCIA o
BLOQUEADO, QA en `qa-pending`, ronda `incomplete`, modo de una rama o ronda parcial ·
🔴 frente fuera de alcance, CI rojo sin arreglo atribuible o registro rechazado por el
helper · 🚫 aplicación en `wrong-host` · ⏸️ sin subagentes o subagente que falló dos
veces. El bloque de avance va antes de la última línea:
`🟢/🟡/🔴/🚫/⏸️ improvement-pass — <resultado>`.

## Referencias externas

- https://github.com/cloudflare/security-audit-skill · commit c1c8a8c1471069fb0e188eeaff69b8e8db6564a8 · revisado 2026-10-10 · Adaptado de cloudflare/security-audit-skill (MIT) · orquestación con verificador independiente, crítico de cobertura, registro con tres veredictos y huella estable
