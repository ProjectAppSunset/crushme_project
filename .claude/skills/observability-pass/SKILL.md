---
name: observability-pass
description: "Pasada de observabilidad y tolerancia a fallos de UN proyecto: logs, métricas, errores, correlación/tracing, alertas, diagnóstico, timeouts, retries, idempotencia y concurrencia. Diagnostica hasta 3 candidatos; --apply corrige instrumentación o fallos acotados y prepara /qa. Evalúa beneficio antes de cambiar y registra decisiones. Diseña futuras entregas de eventos a ProjectApp conforme a su API v1 existente; no instala un emisor, activa monitoreo ni genera tokens automáticamente."
argument-hint: "[proyecto] [--apply|--check|--refresh|--review3] [--module=<alcance>]"
allowed-tools: Bash, Read, Edit, Write, Grep, Glob, Agent, AskUserQuestion, EnterWorktree
---

# Observability pass

Poder detectar y explicar un fallo real, y limitar sus efectos cuando una
dependencia externa, retry o carrera interviene. Leer
`$HOME/webapps/vps-ops-toolkit/workflows/improvement/IMPROVEMENT_STANDARDS.md`,
con su «Protocolo de validación de candidatos».
Para integración con ProjectApp, leer también
`workflows/improvement/PROJECTAPP_EVENT_CONTRACT.md` en ese mismo toolkit y
corroborar la versión vigente de la API antes de proponer un payload.

## Cómo invocar este skill

Sin menú por diseño (§4): el alcance y el modo se resuelven de la invocación y del contexto; la delegación conserva la selección del conductor.

Sin picker por diseño: default diagnostica y valida (§1b) hasta tres
candidatos nuevos y registra hallazgos; `--apply` cambia sólo candidatos elegibles; `--check` no
escribe nada; `--refresh` sólo descubre/registra; `--review3` revisa hasta tres
anteriores. Modos excluyentes. Hereda contexto delegado de [[improvement-pass]]
sin preguntar modo ni crear otra rama.

Qué NO se pregunta: instalar telemetría, comprar servicios, emitir tokens o
activar integración como paso rutinario. Esos cambios requieren alcance y
operación propios; no se infieren de esta pasada.

## 1. Preflight e inventario

Resolver proyecto/coordenada. Aplicar en el worktree actual o crear
`session-worktree.sh create fix observability-<slug>`; Claude `EnterWorktree`,
Codex `cd` y comandos con ese workdir. Identificar procesos críticos y sus
interfaces externas; no inspeccionar logs con secretos sin un filtro seguro.

| Frente | Pregunta que guía la evidencia |
|---|---|
| Logs/errores | ¿Se distingue éxito, fallo y degradación con contexto suficiente y saneado? |
| Métricas/alertas | ¿La señal detecta una obligación del proceso y tiene dueño/acción, sin spam? |
| Tracing/correlación | ¿Se puede relacionar una operación con su tarea/llamada sin usar datos privados? |
| Diagnóstico | ¿Un fallo silenciado o mensaje genérico impide encontrar su causa concreta? |
| Timeouts/retries | ¿Hay límites completos y un retry de un efecto produce una repetición peligrosa? |
| Idempotencia/concurrencia | ¿Duplicados o carreras alteran datos, estados o efectos externos? |

Seguir código de inicio, errores, recuperación y cancelación. Usar fallos
simulados/fixtures; no provocar outages, correos o pagos reales. Una señal
ausente sólo se convierte en candidato al nombrar qué fallo quedaría invisible
y qué decisión permitiría tomar. Más logs/traces por sí solos no son mejora.

## 1b. Validación del frente

Cada candidato pasa por el «Protocolo de validación de candidatos» del contrato
antes de seleccionarse: registro con tres veredictos; verificación **antes de
seleccionar** de todo candidato propuesto, de la causa más fuerte del alcance aunque
la conclusión sea MADURO o NO VALE LA PENA, de todo `mandatory` o `bloqueante` y de
los previos con fuente cambiada; modalidad `quick`, y `standard` automática para
`mandatory` y `bloqueante`; crítico de cobertura y ledger de cobertura. Esta sección
sólo agrega lo propio de observabilidad.

Sola, la pasada lanza un verificador fresco por candidato, que no lo descubrió, y,
al cerrar el diagnóstico, el crítico del contrato, con las herramientas de
«Herramientas por host»: `Agent` en Claude Code o `spawn_agent` en Codex; en un VPS
del fleet, sólo `Agent`. Delegada, la pasada no lanza subagentes: el crítico y los
verificadores los lanza el conductor, y la pasada recibe el registro ya decidido.
Sin verificador o sin crítico rige «Sin subagentes, sin verificador o sin crítico»
del contrato.

Misma causa raíz = el mismo punto donde la señal se pierde o el efecto se repite
(un cliente compartido sin timeout, un `except` que traga el error), aunque lo
atraviesen varios procesos.

El registro nombra el modo de falla, su disparador y la corrección propuesta. El
verificador re-lee cada `file:line` y trata de refutar:

1. **Falla real y alcanzable.** Camino completo desde el disparador (llamada
   externa, tarea, request, cancelación) hasta el punto donde la falla se
   pierde, se repite o corrompe estado.
2. **Hoy no se ve.** Recorrer lo que ya podría mostrarla: `LOGGING` y sus
   handlers, manejo de errores del framework, middleware, registro de fallos de
   Huey/Celery, `/api/health/` o una integración existente. Si ya la muestra con
   contexto suficiente, `rejected`. Si podría verse por algo que no está en el
   repo (journald, alertas del host, nginx), no suponer presencia ni ausencia:
   `needs_validation` con la observación concreta que haría el operador.
3. **Decisión que habilita.** Quién actúa y qué hace con la señal nueva. Una
   señal sin dueño, que no cambia ninguna decisión o que sólo suma volumen se
   refuta.
4. **Resiliencia demostrable.** Timeout realmente ausente o incompleto en esa
   frontera (la llamada y sus defaults); retry que envuelve un efecto no
   idempotente (pago, notificación, escritura) según su configuración real;
   ventana de lectura-modificación-escritura sin lock, constraint, `F()` ni clave
   de idempotencia. Se demuestra con fixtures, reloj simulado o dos llamadas en
   la DB de test, nunca con dependencias reales.
5. **Corrección dentro del frente.** No crea ni instala un emisor, no registra
   fuentes, no genera tokens, no envía HTTP a ProjectApp, no activa monitoreo ni
   contrata servicios; un payload propuesto respeta la API v1 de
   `PROJECTAPP_EVENT_CONTRACT.md`. Lo que exceda eso queda como propuesta o
   «integración pendiente», nunca como recepción verificada.
6. **Privacidad.** La corrección no registra argumentos, SQL completo, bodies,
   tokens ni identificadores de usuarios como dimensión. Si el código actual ya
   expone datos sensibles, la causa es de [[security-pass]]: se reasigna, no se
   duplica.

El verificador no lee logs vivos ni datos reales: usa los extractos saneados ya
citados en el reporte y, si lo decisivo sólo está en el host, deja un plan de
lectura filtrada para el operador. Anclas de impacto, subordinadas a la escala
del contrato: ALTO = pérdida de datos, efecto externo duplicado o proceso
crítico que falla en silencio, con camino demostrado (y no se difiere por
economía); MEDIO = falla visible pero no diagnosticable, o frontera sin timeout
con efecto acotado; BAJO = contexto adicional en caminos raros o de poco efecto
(`marginal`: una línea de descarte). Si al aplicar aparece evidencia que cambia el
registro, vuelve a un verificador fresco antes de seguir.

**Ledger de cobertura de observabilidad.** Una unidad por proceso crítico ×
pregunta de la tabla de §1, con el esquema del ledger de cobertura del contrato: en
`unit`, `process` y `question`; sus chequeos nombran la interfaz externa, las rutas
seguidas (inicio, error, recuperación, cancelación) y las señales existentes con
`file:line`. Un control fuera del repo que quedó sin observar deja la unidad
`blocked` con su razón. Una unidad aparte declara la integración ProjectApp:
`covered` si existe y se auditó, `deferred` con «integración pendiente», o
`not_applicable`.

## 2. Selección y aplicación acotada

Registrar candidatos `observability` con el veredicto que dejó el verificador
(§1b), según el contrato común; sólo se aplican los `confirmed` que selecciona el
umbral del contrato. Pérdida de datos, efectos duplicados comprobados o
regresiones obligatorias no se difieren por economía. No inventar una observación
exitosa donde faltan ejecución o datos.

Aplicar sólo cambios con beneficio: contexto/error estable y saneado, timeout
en la frontera, retry limitado para operaciones que permiten repetición,
deduplicación/atomicidad requerida o diagnóstico de un fallo silenciado.
Conservar contratos de errores y límites del host. Backoff/jitter, límites de
intentos y mecanismo de idempotencia deben salir de las necesidades del proceso;
no añadir un retry genérico a pagos, notificaciones o escrituras no idempotentes.
No registrar argumentos, SQL completo, bodies, tokens ni identificadores de
usuarios como dimensiones de métricas.

Instrumentación acotada de aplicación puede formar parte de `--apply`. Una
reorganización de colas, nueva plataforma APM o política de alertas operativas
fuera de alcance queda como propuesta. Seguridad del saneamiento pertenece a
[[security-pass]]; coste de la instrumentación a [[perf-pass]]; el conductor
asigna una causa única y evita aplicar dos cambios que compitan.

## 3. Integración futura con ProjectApp

El contrato común de eventos reutiliza `POST /api/monitoring/v1/ingest/` versión
1 con `Bearer`, recurso/fuente autorizados y claves admitidas. Esta entrega de
la skill define instrucciones: **no construye ni instala un emisor, cambia
ProjectApp, registra fuentes, genera tokens o envía HTTP**.

Si un proyecto carece de emisor/fuente autorizados, producir propuesta y marcar
“integración pendiente”; no declarar recepción verificada. Si ya tiene
integración, auditarla por sus límites y recibos sin enviar eventos reales.
Un futuro emisor escribirá eventos saneados a una cola acotada local,
desacoplada del flujo de negocio, y sólo confirmará entrega tras ACK válido.

`external_id` identifica la entrega y persiste en todos sus retries;
`fingerprint` identifica el problema dentro de una fuente/recurso. No usar el
request ID efímero como fingerprint. La recuperación técnica no cierra un caso
manualmente. Trace/span IDs y evidencia anidada **no están admitidos en v1**;
proponer una extensión versionada separada si resultan necesarios, sin enviar
campos inventados ni esconderlos dentro de un string.

## 4. Guion, registro y cierre

Entregar `brief-observability` a [[qa]]: entrada/fallo simulado, resultado y
estado concretos, capa dueña, test existente, `file:line` y bug. Para retries,
usar reloj simulado y asertar número de intentos/límite/cancelación, no tiempos
de pared. Para concurrencia, probar la carrera y el resultado persistente, no
sólo presencia de locks. Para saneamiento, asertar campos/valores permitidos y
estado útil además de ausencia de datos sensibles.

Si se implementa un emisor en una tarea futura autorizada, su QA deberá cubrir
duplicados, 409 por contenido distinto, rechazo/saneamiento, timeout/red/429/5xx,
cola llena, credencial revocada, ACK inválido, recuperación y carreras. La skill
no autoriza ejecutar ahora esos escenarios contra el API real.

Guardar reporte y registro, con los registros validados y el ledger de
cobertura de §1b (sin métricas; en `--check`, sólo en la respuesta); applied
espera QA. Independiente: handoff [[qa]],
sin inline QA. Delegada: devolver IDs, diff, guion, contrato/eventos propuestos
y pendientes; no abrir PR/commit propios.

## Output final

Usar [[_output-protocol]]: Señales · Diagnóstico · Resiliencia · Privacidad ·
ProjectApp · Validación · Cobertura · Valor · QA, con el estado del frente del
contrato (MADURO, NO VALE LA PENA, VALE LA PENA, BLOQUEADO o FALTA EVIDENCIA).
Separar recepción demostrada de propuesta pendiente y alcance suficiente de
ausencia de evidencia. Cerrar con
`🟢/🟡/🔴 observability-pass — <resultado>` según evidencia; bloque de avance
antes de esa última línea.

## Referencias externas

- https://github.com/cloudflare/security-audit-skill · commit c1c8a8c1471069fb0e188eeaff69b8e8db6564a8 · revisado 2026-10-10 · Adaptado de cloudflare/security-audit-skill (MIT) · validación independiente de candidatos, crítico de cobertura y ledger de cobertura
