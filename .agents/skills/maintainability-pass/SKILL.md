---
name: "maintainability-pass"
description: "Pasada de mantenibilidad de UN proyecto: complejidad, duplicación, dependencias entre módulos, arquitectura y deuda técnica con efecto demostrado. Diagnostica hasta 3 candidatos nuevos; --apply hace refactors acotados que conservan el comportamiento y prepara pruebas para $qa. Registra decisiones y detiene cambios cuyo beneficio no justifica coste/riesgo. Funciona sola o delegada por $improvement-pass; no hace reescrituras globales ni cambia el producto por estilo."
---

# Maintainability pass

Reducir el coste comprobable de entender/cambiar un proyecto preservando sus
contratos. Leer
`$HOME/webapps/vps-ops-toolkit/workflows/improvement/IMPROVEMENT_STANDARDS.md`
para modos, registro, criterio de beneficio, contexto delegado y el «Protocolo de
validación de candidatos».

## Cómo invocar este skill

Sin menú por diseño (§4): el alcance y el modo se resuelven de la invocación y del contexto; la delegación conserva la selección del conductor.

Sin picker por diseño: sin flags diagnostica, valida (§2b) y registra hasta tres
candidatos nuevos, sin modificar la aplicación. `--apply` aplica los `confirmed`
elegibles; `--check`
no escribe nada; `--refresh` descubre/registra sin refactors; `--review3` revisa
hasta tres anteriores. Modos excluyentes; proyecto/cwd y alcance explícito no se
vuelven a preguntar. Delegada por $improvement-pass, hereda su contexto.

Qué NO se pregunta: nombres internos o elecciones locales reversibles. Una
reorganización de arquitectura o cambio de contrato fuera del alcance se
registra como propuesta, no se supone autorizado por “mejorar mantenibilidad”.

## 1. Preflight y causas

Resolver coordenada, políticas y módulos reales. Para aplicar, usar el worktree
de la sesión o crear `session-worktree.sh create refactor maintainability-<slug>`
antes del primer write; Claude entra con `EnterWorktree`, Codex con `cd` y workdir
correspondiente. Los cambios no requieren tocar el clon que sirve la app.

Buscar mecanismos, no un número objetivo de líneas o complejidad:

| Problema | Beneficio que debe poder demostrarse |
|---|---|
| Complejidad | Ramas/efectos difíciles de modificar causan bugs o dificultan un cambio concreto. |
| Duplicación | La misma regla de negocio diverge o exige modificar varios sitios al cambiar. |
| Arquitectura | Ciclos, límites de responsabilidad rotos o dependencia transversal con efecto real. |
| Deuda técnica | Workaround, código obsoleto o acoplamiento bloquea una operación/prueba/mantenimiento. |

Leer call sites, contratos, pruebas existentes e historial relevante. Dos
fragmentos parecidos con responsabilidades distintas no obligan a abstraerlos.
Un archivo largo, un nombre imperfecto o una métrica alta no justifican por sí
solos una intervención. $repo-cleanup orienta la auditoría de archivos sin uso;
la eliminación de tests pertenece a $test-audit, nunca a esta pasada.

## 2. Propuesta y valor

Crear candidatos `maintainability` con causa y paths exactos. Nombrar el cambio
que será más seguro/sencillo tras el refactor y su evidencia. Tras la
validación de §2b, aplicar el umbral del contrato (impacto ALTO, o MEDIO con
esfuerzo y riesgo BAJOS); lo que no lo alcanza, incluidas las reorganizaciones
grandes con retorno marginal, se difiere con razón.

Elegir la menor transformación que elimine la causa: extracción de una regla
compartida, función pura para una decisión, límite entre coordinación/efecto o
reducción de ramas redundantes. Preservar APIs, validaciones, errores, orden de
efectos, transacciones y persistencia. Una modificación de resultados públicos
es trabajo funcional para $implement, salvo que ya forme parte del pedido.

No agregar un framework/capa/abstracción para una única variante hipotética. No
mezclar formateo global, renombres masivos ni modernizaciones de dependencias
con el refactor elegido. Un candidato que exige ese alcance queda `blocked` o
se formula como propuesta separada, sin ocultarlo bajo “limpieza”.

## 2b. Validación del frente

Cada candidato, con su propuesta de transformación, pasa por el «Protocolo de
validación de candidatos» del contrato antes de cerrar su valor y de aplicarse:
registro con tres veredictos; verificación **antes de seleccionar** de todo
candidato propuesto, de la causa más fuerte del alcance aunque la conclusión sea
MADURO o NO VALE LA PENA, de todo `mandatory` o `bloqueante` y de los previos con
fuente cambiada; modalidad `quick`, y `standard` automática para `mandatory` y
`bloqueante`; crítico de cobertura y ledger de cobertura. Esta sección sólo agrega
lo propio de mantenibilidad.

Sola, la pasada lanza un verificador fresco por candidato, que no lo descubrió, y,
al cerrar el diagnóstico, el crítico del contrato, con las herramientas de
«Herramientas por host»: `Agent` en Claude Code o `spawn_agent` en Codex; en un VPS
del fleet, sólo `Agent`. Delegada, la pasada no lanza subagentes: el crítico y los
verificadores los lanza el conductor, y la pasada recibe el registro ya decidido.
Sin verificador o sin crítico rige «Sin subagentes, sin verificador o sin crítico»
del contrato.

Misma causa raíz = la misma regla, decisión o límite de responsabilidad, aunque
aparezca en varios archivos: un solo registro con todos sus sitios.

El verificador re-lee cada `file:line` citado y trata de refutar:

1. **Consecuencia concreta y comprobable, no estilo.** Vale una divergencia ya
   existente (misma entrada, resultado distinto en dos caminos, con ambas
   líneas), un bug o regresión del historial ligado a ese código (commit o
   issue que lo muestren), un cambio pedido o documentado que obliga a tocar
   varios sitios (lista exacta), un ciclo de imports real o una prueba que hoy
   no puede aislar una decisión por un efecto en `file:line`. Longitud,
   cantidad de funciones, una métrica de complejidad, nombres, «quedaría más
   limpio» o una variante futura hipotética refutan el candidato.
2. **Misma responsabilidad.** En duplicación, los fragmentos aplican la misma
   regla de negocio; parecido textual con responsabilidades distintas es
   `rejected`. En arquitectura, el ciclo o la dependencia ocurre en ejecución,
   no en un import bajo `TYPE_CHECKING`, local a una función o muerto.
3. **Contrato público conservado.** Enumerar APIs, respuestas, validaciones,
   errores, orden de efectos, transacciones y persistencia que toca la
   transformación y comprobar que no cambian. Si la propuesta altera un
   resultado público, el hallazgo puede ser cierto, pero su arreglo es trabajo
   de $implement y queda fuera del frente.
4. **Alcance mínimo.** Archivos exactos y la menor transformación que elimina la
   causa, sin formateo global, renombres masivos ni dependencias mezclados.
5. **Riesgo de regresión.** Buscar todos los call sites, incluidos despachos
   dinámicos (señales, `getattr`, imports por string, templates), y las pruebas
   conductuales que hoy los cubren, con `file:line`. Riesgo BAJO exige un cambio
   local con verificación conductual disponible; call sites sin prueba o
   despacho dinámico lo suben al menos a MEDIO.

Anclas de impacto, subordinadas a la escala del contrato: ALTO = divergencia o
bug ya demostrados, u operación requerida bloqueada; MEDIO = un cambio concreto
y cercano que hoy es más caro o riesgoso; BAJO = legibilidad sin cambio previsto
(`marginal`: una línea de descarte).

`needs_validation` cuando lo decisivo no está en el repo (por ejemplo, si el
cambio de regla está realmente previsto): plan exacto y no destructivo, de
lectura local (`grep` de call sites, lectura de pruebas) o una pregunta concreta
al operador. Para decidir nunca se ejecuta la app ni se consultan datos reales.
Sólo un candidato `confirmed` puede aplicarse.

**Ledger de cobertura de mantenibilidad.** Una unidad por módulo × mecanismo
(complejidad · duplicación · arquitectura · deuda), con el esquema del ledger de
cobertura del contrato: en `unit`, `module` y `mechanism`; sus chequeos nombran el
método (call sites, imports, historial, pruebas existentes leídas) y los contratos
identificados. Lo no examinado queda `deferred` con su razón y nunca se presenta
como sano.

## 3. Aplicación y guion

Trabajar secuencialmente dentro de los paths aprobados, conservando cambios
ajenos. Si el descubrimiento contradice la propuesta, reducir alcance o
registrar `needs-evidence`; no seguir refactorizando para justificar el trabajo.
Un registro corregido al aplicar vuelve a un verificador fresco (§2b) antes de
seguir: un reemplazo no conserva el `confirmed` anterior.

Entregar `brief-maintainability` a $qa con la capa dueña, test existente a
extender y `file:line`. Cada caso debe invocar el comportamiento con entradas
concretas, asertar resultado/estado exacto y nombrar el bug que detectaría.
Cubrir los bordes que el refactor puede alterar: ramas válidas e inválidas,
orden de efectos, errores propagados y equivalencia del contrato.

No pedir tests de cantidad de funciones, orden de imports, estructura interna
o “la función nueva fue llamada”. Si la transformación es local y no tiene
comportamiento nuevo que probar, declarar abstención de autoría y utilizar las
pruebas conductuales existentes adecuadas. El guion documenta qué debe correr
QA para validar que no se alteró el comportamiento; no inventa cobertura.

## 4. Registro y cierre

Guardar reporte y estados con el helper común; los análisis o mediciones van
en el reporte, no como objetivo en el ledger. Un cambio aplicado espera QA.
El reporte lleva los registros validados y el ledger de cobertura de §2b, sin
métricas; en `--check`, todo va sólo en la respuesta.

Independiente: handoff a $qa, sin ejecutarlo en esta pasada. Delegada: devolver
IDs, diff, contratos preservados, guion, evidencias y pendientes al conductor;
sin rama, commit o PR adicionales. No tratar tests fallidos como deuda menor ni
modificar expectativas para acomodar un refactor incorrecto.

## Output final

Usar $output-protocol: Complejidad · Duplicación · Arquitectura · Deuda ·
Validación · Cobertura · Valor · QA, con el estado del frente del contrato (MADURO,
NO VALE LA PENA, VALE LA PENA, BLOQUEADO o FALTA EVIDENCIA). Explicar cada parada
con alcance, beneficio restante y coste/riesgo, sin afirmar que todo el proyecto
está perfecto. Cerrar con
`🟢/🟡/🔴 maintainability-pass — <resultado>` según evidencia; bloque de avance
antes de esa última línea.

## Referencias externas

- https://github.com/cloudflare/security-audit-skill · commit c1c8a8c1471069fb0e188eeaff69b8e8db6564a8 · revisado 2026-10-10 · Adaptado de cloudflare/security-audit-skill (MIT) · validación independiente de candidatos, crítico de cobertura y ledger de cobertura
