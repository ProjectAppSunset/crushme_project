---
name: security-pass
description: "Pasada de seguridad de UN proyecto con método: mapa de superficie, catálogo de clases adaptado al stack y cobertura registrada. Diagnostica hasta 3 candidatos con verificación independiente y crítico de cobertura; --apply corrige sólo los elegibles en el worktree de sesión y prepara brief-security para /qa. Dependencias vía /vuln-audit. Sola o delegada por /improvement-pass; sólo código y fixtures locales: sin exploits contra lo desplegado, valores de secretos ni rotaciones."
argument-hint: "[proyecto] [--apply|--check|--refresh|--review3] [--module=<alcance>]"
allowed-tools: Bash, Read, Edit, Write, Grep, Glob, Agent, AskUserQuestion, EnterWorktree
---

# Security pass

Mejorar la seguridad de un proyecto con evidencia del camino explotable y cambios
acotados. Leer dos referencias del toolkit antes de empezar:

- `$HOME/webapps/vps-ops-toolkit/workflows/improvement/IMPROVEMENT_STANDARDS.md`:
  modos, worktree, registro, regla de beneficio, contexto delegado y el «Protocolo de
  validación de candidatos» (tres veredictos, verificador independiente —`quick`, y
  `standard` automática para `mandatory` y `bloqueante`—, crítico de cobertura y
  ledger de cobertura).
- `$HOME/webapps/vps-ops-toolkit/workflows/improvement/SECURITY_REFERENCE.md`:
  lo específico de seguridad —dónde mirar en nuestro stack, huella del candidato,
  prompt del explorador, catálogo de clases `S-*` con la evidencia que cuenta en cada
  una, lista del verificador, severidad, cobertura y reporte—.

Las pruebas las escribe y ejecuta [[qa]], no esta pasada.

**Regla central.** Un candidato de seguridad nombra seis cosas: el principal de
menor confianza y su capacidad de partida, la entrada o acción aceptada, el
control que debería impedirla, la frontera que cruza, el principal o recurso
afectado y el resultado concreto. Si falta alguna, no es candidato: una buena
práctica ausente es una nota de hardening (impacto BAJO, una línea de descarte), una
conjetura sobre el despliegue es `needs_validation` y el efecto sobre uno mismo no
cruza ninguna frontera.

## Cómo invocar este skill

Sin menú por diseño (§4): el alcance y el modo se resuelven de la invocación y del contexto; la delegación conserva la selección del conductor.

Sin picker por diseño: sin flags diagnostica hasta tres candidatos nuevos y
registra metadatos y cobertura; el proyecto sale del cwd/contexto. No modifica
la aplicación. `--apply` autoriza los cambios elegibles y el handoff; `--check`
no escribe nada —ni reportes, ledger ni cobertura— y entrega mapa, candidatos y
cobertura en la respuesta; `--refresh` sólo descubre/registra; `--review3`
revisa hasta tres candidatos anteriores. Los modos son excluyentes.
`--module=<alcance>` acota mapa y cobertura: lo que queda fuera es
`out_of_scope`, nunca `covered`. Con contexto de [[improvement-pass]], hereda
selección, modo y worktree sin menú propio.

Qué NO se pregunta: permisos ya concedidos para el alcance, coordenadas
resolubles, qué clases del catálogo revisar (las decide el mapa), si corresponde
el verificador independiente (es parte del método) ni rotación de secretos como
consecuencia automática de una alerta. Una decisión real de negocio sobre quién
puede acceder sí se plantea con la evidencia; el resto de la pasada continúa
donde sea independiente.

## Límites de ejecución (todos los modos)

- La pasada y sus verificadores leen código y configuración versionada: no ejecutan
  código del proyecto ni le envían entradas. Lo conductual es de [[qa]], con datos
  ficticios, servicios externos simulados y su entorno de tests aislado; por eso lo
  confirmado por la fuente queda `confidence: inferred` hasta que [[qa]] ejecute el
  test.
- Nada contra lo desplegado: ni tráfico a producción o staging, ni llamadas a
  Stripe, Square, SOVOS, correo o modelos LLM —tampoco a sus entornos de prueba
  ni gastando cuota—, ni consultas a la base. El `backend/.env` de un worktree
  enlaza al de PRODUCCIÓN: ni `manage.py` con él ni migraciones.
- Ningún valor de secreto, cookie, token, cuerpo de request ni dato de clientes en
  briefs, reportes, despachos, chat o registros: se informan tipo, ruta y canal y, si
  hay que probar que una copia coincide, el sha256 del archivo (regla del fleet, R34).
  Las credenciales canónicas bajo `config/credentials/` del toolkit son diseño
  autorizado, y una referencia a un secreto no es una exposición.
- Ninguna mutación del host, deploy, rotación, revocación ni secret scanning
  remoto como efecto de la pasada.
- Un hecho decisivo que el código no muestra (nginx desplegado, valores del
  `.env`, panel del proveedor, firewall, versión servida) no se prueba en vivo:
  `needs_validation` con el hecho exacto y una verificación del dueño que lee
  configuración o estado, nunca tráfico de auditoría.

## 1. Preflight y mapa de superficie

Resolver la coordenada y leer políticas del proyecto antes de escribir. En
`--apply`, reutilizar el worktree propio; si falta, crearlo con
`session-worktree.sh create fix security-<slug>` y entrar (Claude:
`EnterWorktree`; Codex: `cd`, todos los comandos con ese workdir). Anotar el SHA
revisado (`base_commit` del registro) y si el árbol estaba sucio.

Leer el registro y la cobertura anteriores antes de buscar, y aplicar la herencia de
SECURITY_REFERENCE.md §7 junto con «Rondas anteriores» del contrato: con contexto
idéntico se conservan cobertura y decisiones; un `confirmed` previo sin aplicar vuelve
a pasar por un verificador; un `rejected` suprime sólo su afirmación exacta;
pendientes y fuente cambiada son trabajo actual; una pasada parcial no certifica el
resto.

Construir el mapa (en el reporte, ≤ ~500 palabras más tablas; dónde mirar, en
SECURITY_REFERENCE.md §2):

1. Actores: roles, propietarios/tenants, staff/admin, anónimo, emisores
   externos (webhooks, correo entrante) y una página de otro origen.
2. Entradas: rutas Django/DRF —incluidas `@action`, admin, API navegable y
   esquema—, webhooks, tareas Huey/Celery que dispara un usuario, uploads,
   fuentes del navegador, rutas y middleware de Next.js, entradas de un LLM.
3. Límites de confianza con el control más fuerte visible en cada uno.
4. Flujos hasta sumideros: SQL crudo, shell, HTML/plantillas, rutas de archivo,
   redirects, peticiones salientes, deserialización, correo, estado de pagos o
   facturas, salida de un modelo.
5. Controles que dependen del despliegue y quién puede verificarlos.
6. Clases `S-*` seleccionadas y excluidas, cada exclusión con su hecho de
   fuente: una clase entra porque el mapa encontró su frontera, no porque
   aparezca el nombre de una dependencia.

No calificar como vulnerable un endpoint únicamente porque carece de login si su
contrato autoriza acceso público. Sola y con subagentes, el mapa puede hacerlo un
explorador de lectura que devuelve hechos con `file:line` y no escribe; delegada, lo
hace el propio diagnosticador.

## 2. Descubrimiento por clase (una ola, perfil «quick»)

Sembrar las unidades de cobertura —superficie × frontera × clase
seleccionada— con los campos y la prioridad de SECURITY_REFERENCE.md §4 y §7.
Método por unidad:

1. **Invariante concreto.** Principal de menor confianza → valor o acción
   aceptada → control que debería rechazar, ligar o limitar → camino exacto
   después de esa decisión → efecto mínimo (registro ajeno ficticio, valor
   indebido, estado alterado) → cambio de fuente y caso de regresión.
2. **Lectura en profundidad.** Seguir la entrada por parseo, identidad,
   autorización, normalización, estado, copias derivadas y sumidero; leer las
   rutas hermanas del mismo efecto (API, admin, export/import, bulk, tareas,
   retry, error, legacy) y comparar sus controles por equivalencia, no sólo por
   presencia.
3. **Caminos tristes**, sólo donde la interfaz los acepta: ausente, vacío,
   cero, negativo, máximo, duplicado, codificación mezclada, revocado,
   reordenado, concurrente, dependencia caída.
4. **Variantes.** Una causa grave se busca en sus variantes dentro de las
   unidades asignadas: una causa con varias entradas es un candidato; controles
   ausentes independientes son candidatos distintos.
5. **Límite de profundidad.** Parar en cuanto el invariante queda resuelto en
   un sentido u otro y registrar la disposición; otra frontera que aparezca es una
   unidad propia `deferred` con `pending: frontera encontrada sin revisar`, nunca un
   `out_of_front`.

Dependencias: la clase `S-SUPPLY` reutiliza [[vuln-audit]] en modo auditoría
para las superficies presentes; no reimplementar sus scanners ni tratar todo
paquete outdated como una vulnerabilidad. Un advisory cuenta si afecta a la
versión instalada en alcance runtime; dev-only o una función no alcanzada es
nota. Scanner ausente = evidencia incompleta, no seguridad confirmada. El
scanner no se instala ni cambia el venv del servicio para poder auditar.

Quién recorre las unidades:

- **Sola:** hasta cuatro exploradores de lectura simultáneos, con el prompt de
  SECURITY_REFERENCE.md §4 (los bloques `S-*` copiados literales, nunca por nombre);
  sólo la pasada escribe. Sin subagentes, la pasada recorre las unidades ella misma,
  y la verificación y el crítico siguen «Sin subagentes, sin verificador o sin
  crítico» del contrato. Herramientas y condición de host, las de «Herramientas por
  host» del contrato: en un VPS del fleet, sólo `Agent`, en Claude Code.
- **Delegada por [[improvement-pass]]:** la pasada corre dentro de un subagente y no
  lanza otros: ni exploradores, ni crítico, ni verificador. El despacho trae
  SECURITY_REFERENCE.md §1 y los bloques `S-*` literales; el propio diagnosticador
  hace el mapa, recorre sus unidades y devuelve el contrato de retorno de la ronda.
  El crítico y los verificadores los lanza el orquestador.

Compuerta del candidato, antes de proponerlo:

- Traza completa en el repo, de la entrada al sumidero, con el control visible
  más fuerte del camino.
- Resultado concreto sobre otro principal o recurso; sin tercero afectado, es
  hardening.
- Efecto no fortalecido: un crash no es ejecución de código, trabajo ordinario
  no es caída del servicio compartido, una acción del mismo principal no es
  escalada.
- Hecho decisivo fuera del código → `needs_validation`, sin calificar. Refutado
  por la fuente → `rejected`; no queda en espera.
- Una conclusión sólo en prosa, sin unidad, traza ni huella, no se acepta.

Crítico de cobertura: sola, al cerrar la ola, la pasada lanza el crítico del contrato
(«Crítico de cobertura»), un subagente fresco de lectura, con la lista de
SECURITY_REFERENCE.md §7 como extensión de seguridad; delegada, lo lanza el
orquestador de la ronda. Cada hueco deja su unidad `deferred` con `pending: hueco del
crítico` y no abre otra ola; un descarte objetado pasa al conjunto a verificar. Sin
crítico, el frente no puede declararse suficiente («cobertura sin crítico»).

## 3. Verificación independiente, severidad y selección

Aplicar el «Protocolo de validación de candidatos» del contrato. Se verifica **antes
de seleccionar**, con el conjunto que fija el Protocolo: todo candidato propuesto, la
causa más fuerte del frente aunque concluya que no hay nada que cambiar, todo
`mandatory` o `bloqueante`, los registros previos cuya fuente cambió y los descartes
que objete el crítico. Modalidad: `quick`, y `standard` automática para `mandatory` y
`bloqueante`. Lo específico de seguridad es la lista de SECURITY_REFERENCE.md §6:
entrada real → sumidero, controles visibles en el camino, actor y recurso afectados,
probabilidad e impacto por separado, y remediación que hace cumplir el invariante sin
mover la confianza. El verificador intenta refutar y no recibe la conclusión de otro
verificador. Sola, la pasada lanza un verificador fresco por candidato; delegada, los
lanza el orquestador. Sin verificador independiente rige «Sin subagentes, sin
verificador o sin crítico» del contrato: todo hallazgo queda `needs_validation` con la
razón «sin verificador independiente».

Veredictos, con los nombres del Protocolo (los del helper sólo al hablar con él):

- `confirmed` → sigue como candidato, con `confidence: inferred` hasta que [[qa]]
  ejecute el test.
- `needs_validation` → bloqueador exacto y plan (`local`: test de caracterización que
  [[qa]] pueda correr con fixtures; `owner`: verificación del dueño); sin calificar y
  sin aplicar. Ante el helper es `needs-evidence`.
- `rejected` → con su razón; no se repropone mientras la fuente no cambie. En el
  ledger puede quedar `discarded`.
- Promoción o cambio material de causa, traza, impacto o severidad → otro
  verificador fresco; sin él, queda `needs_validation`. Un candidato sin
  verificar nunca se presenta como vulnerabilidad confirmada.

Severidad e impacto son ejes distintos. La **severidad** (el daño) sólo la lleva un
candidato confirmado y nunca supera el impacto demostrado; la probabilidad puede
bajarla, no subirla. Anclas y mapeo a `bloqueante|mayor|menor` en
SECURITY_REFERENCE.md §6; si no se puede nombrar el daño concreto, la severidad es
menor de lo que parece. El **impacto** (el beneficio de corregir) es ALTO =
`material`, MEDIO = `minor` y BAJO = `marginal`: un hardening sin frontera cruzada es
BAJO, una línea de descarte, nunca `minor` ni `mandatory`.

Construir candidatos del frente `security`, deduplicados por causa, con el formato
común y los significados de SECURITY_REFERENCE.md §3. `root_cause` lleva la huella
`<clase>:<ruta>::<símbolo>` del control o sumidero, sin línea, normalizada por lista
blanca a la clase del motor y validada con su regex antes de escribir; así el ID
estable por causa no fusiona controles independientes. Autorización eludida,
exposición comprobada de secretos o datos y vulnerabilidad aplicable que incumple un
requisito obligatorio llevan `mandatory: true` una vez confirmadas; la incertidumbre
se documenta y diagnostica, no se convierte en una afirmación de explotación. Con los
veredictos registrados, seleccionar hasta tres conforme al helper (`--select`).

`--review3` pasa cada candidato anterior por la misma lista sobre la fuente actual: un
`confirmed` sin aplicar vuelve a un verificador aunque la fuente no haya cambiado; la
fuente cambiada exige revalidar desde cero; un arreglo revertido es `regressed`.

## 4. Aplicación, dependencias y guion para QA

Aplicar sólo el camino y los archivos del candidato aprobado. La corrección hace
cumplir el invariante en el último punto de decisión confiable del servidor:
autorización en la vista, el queryset o el permiso del backend; validación
estructural en el límite de entrada; parametrización o codificación en el
sumidero según su contexto. No mover la confianza a guardas del frontend, a
campos o encabezados que controla el cliente, a listas negras de payloads ni a
ocultar la interfaz. Conservar los casos permitidos y la respuesta de rechazo
del contrato (403 o 404). Evitar cambios globales de auth, nuevos roles o
defaults más restrictivos sin el contrato de producto correspondiente. Si el
arreglo exige tocar nginx o el host, el candidato queda `blocked` con el cambio
exacto para el toolkit. Si aparecen archivos/contratos fuera del alcance, parar
ese candidato y explicarlo; no restaurar archivos ajenos.

Antes de cualquier dependencia mutante, delegar a [[vuln-audit]] sólo el
candidato seleccionado en el worktree actual y con el contexto común. Añadir
el descriptor textual `VULN_CANDIDATE` con `surface`,
`packages:[{name,current,target}]`, `advisories` y `allowed_paths`; la delegada
actualiza sólo ese set, no un batch global de outdated. Su `--apply` conserva
límites patch/minor, pins, Node/Python y venv aislado; una
actualización major o feature release de framework no se autoriza aquí. Un
riesgo que necesita major queda `blocked` con la acción explícita de
modernización, no `defer-low-value`. El contexto evita commits/PRs de una segunda
rama y no concede permiso para tocar otras dependencias.

Si `security-pass` se ejecuta independiente, crear ese contexto de delegación
con `conductor: security-pass`, `owns_git: conductor` y la selección propia;
`vuln-audit` deja el diff y esta pasada conserva la responsabilidad de su rama
y entrega. Si ya existe conductor `improvement-pass`, preservar el contexto
externo: no reemplazar su ownership ni abrir otra ronda.

Preparar un bloque `brief-security`, que [[qa]] transforma en los briefs de su
Architect. Cada ítem incluye:

- ID y huella, clase `S-*`, comportamiento/requisito y `file:line` real.
- Actor/rol/tenant de menor confianza, recurso afectado y entrada concreta;
  caso permitido y caso rechazado.
- Valor esperado de respuesta/estado y ausencia de efecto persistente en el
  caso rechazado, acompañada por una aserción positiva sobre el estado conservado.
- Capa y test existente a extender; bug que atraparía —en seguridad, la
  vulnerabilidad confirmada: con el arreglo revertido, el caso rechazado
  falla—; entorno de tests aislado, datos ficticios y servicios externos
  simulados.

Ejemplo de expectativa: propietario cambia su objeto y recibe el valor nuevo;
otro tenant recibe 403/404 conforme al contrato y el objeto conserva el valor
original. Webhook con firma inválida o repetido: rechazo o un único efecto, y el
pedido conserva su estado. No basta con comprobar que se invocó un permiso ni
con un mock. Sin pruebas de carga: un límite se valida con una entrada mínima
por encima del tope.

## 5. Registro, cobertura y cierre

Guardar reporte saneado, veredictos y cobertura mediante el helper común y el
Protocolo; el registro se valida con `improvement-ledger.sh --validate-record` antes
de escribirlo. Cada unidad lleva los campos de SECURITY_REFERENCE.md §7; si el helper
del host todavía no admite el registro (interinos del contrato), la tabla va en el
reporte narrativo y una scope sólo se marca `complete: true` cuando cumple la regla
de suficiencia de esa sección. Reporte con las secciones de SECURITY_REFERENCE.md §8:
un resultado limpio puede tener cero confirmados; no inventar hallazgos menores ni
incluir pasos contra lo desplegado. `--check` entrega todo en la respuesta. Cambio
aplicado = `qa-pending`; no marcar `verified` porque desapareció una advertencia
estática o porque un scanner quedó verde.

El frente es suficiente sólo con todas las unidades del alcance cerradas según esa
regla, el crítico corrido y ningún `needs_validation` ni obligatorio bloqueado
abierto. Haber corrido una ola no basta por sí solo, y tampoco lo demuestran un
módulo, el cupo de tres ni un scanner ausente.

Independiente: entregar el guion y siguiente paso [[qa]]; no ejecutarlo dentro
de esta skill. Delegada: devolver al conductor IDs, diff, guion, cobertura,
riesgos y pendientes; sin commit/push/PR propios. No ejecutar deploy, rotación,
revocación, secret scanning remoto ni cambios del host como efecto implícito.

## Output final

Usar [[_output-protocol]]: Superficie · Auth · Permisos · Inputs · Secrets ·
Integraciones · Dependencias · Cobertura · Valor · QA. `Cobertura` resume
unidades por estado, clases excluidas y los huecos del crítico. Cada frente
suficiente lleva alcance y razón de parada conforme al contrato común. Un
candidato obligatorio bloqueado mantiene la advertencia y el paso necesario; no
declarar el frente suficiente. Cerrar con
`🟢/🟡/🔴 security-pass — <resultado>` según la evidencia, con el bloque de avance
antes de esa última línea en la respuesta al operador; una ejecución delegada
devuelve su resultado al conductor sin ese bloque.

## Referencias externas

- https://github.com/cloudflare/security-audit-skill · commit c1c8a8c1471069fb0e188eeaff69b8e8db6564a8 · revisado 2026-10-10 · Adaptado de cloudflare/security-audit-skill (MIT) · el método de seguridad, reescrito con palabras propias para nuestro stack (detalle y atribución en SECURITY_REFERENCE.md)
