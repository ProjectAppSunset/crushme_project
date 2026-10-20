---
name: "platform-delivery-authoring"
description: "Prepara fases, etapas y guías de validación de Platform desde código y fuentes comerciales/técnicas seleccionadas. Contrasta discrepancias, previsualiza y guarda borradores mediante el Gestor de la plataforma. No publica ni registra conformidad del cliente por esta invocación."
---

# Autoría de entregas en Platform

Convierte recorridos comprobables del producto del cliente en contrato/otrosí
→ alcance → fase → etapa → requerimiento con guía de validación. El código
acredita lo implementado; el contrato y sus anexos incorporados acreditan lo
acordado. Describe diferencias sin convertir una funcionalidad adicional en
obligación ni omitir una obligación todavía incumplida.

## Resultado por defecto

Una petición de preparar etapas deja **borradores revisables**: lectura,
contraste, captura de fuentes, preview y aplicación MCP. `--plan` sólo analiza
y redacta en la conversación, sin guardar inventario, contexto ni jerarquía.
`--preview` conserva la preparación autorizada y previsualiza, sin aplicar.
`--apply` equivale al resultado por defecto. Una petición sólo de analizar o
escribir texto, y el modo Plan del entorno, conservan el modo de sólo lectura.

Publicar, compartir una respuesta, enviar un correo, firmar y registrar una
conformidad histórica son acciones separadas. La invocación no concede esas
autorizaciones. No respondas como el cliente con un actor administrativo.

Las referencias están en `references/` junto a esta SKILL.md; en la fuente
Claude plana están en `platform-delivery-authoring/references/` junto a este
archivo. Lee `authoring-contract.md` antes de operar el MCP y `guide-writing.md`
antes de redactar. Si faltan, usa la copia canónica del toolkit; no sincronices
otros proyectos ni instales conectores para conseguirlas.

## Preparar la verdad y las guías

1. Descubre herramientas y esquemas actuales del **Gestor de la plataforma**
   (slug `projects`). Consulta proyecto, cliente propietario,
   `get_delivery_overview` y `get_delivery_authoring_contract`. Selecciona el
   contrato, otrosíes y fuentes aplicables, no el primer contrato ni todo el
   proyecto. La propuesta aceptada y el proyecto vinculado no prueban firma.
2. Identifica el repo y SHA del **producto del cliente**, distinto del portal
   ProjectApp. Lee recorridos visibles, permisos, datos y estados. Distingue
   código leído, pruebas ejecutadas y despliegue comprobado. Una rama nueva no
   acredita que el cliente pueda probarla en el ambiente indicado.
3. Construye una matriz privada: obligación y cita → comportamiento observado
   con repo/SHA/archivo/línea → brecha → siguiente validación. Una obligación
   sin implementación queda pendiente; una conducta adicional se identifica
   como tal, sin inventar cortesías ni ampliar el contrato. Las fuentes
   incompletas conservan incertidumbre, no justifican exclusiones.
4. En modos de escritura conserva esa matriz mediante el Gestor Documental
   como **Inventario de implementación**, privado y del proyecto/cliente
   correcto. Descubre su esquema y exige privacidad explícita. Mantén versiones
   históricas. Selecciónalo como referencia no normativa con nota de repo/SHA;
   no como adjunto público, documento de firma o documento de entrega.
5. Agrupa guías por recorridos y responsabilidades reales. Roles y permisos
   deben estar acreditados por fuentes y código; los perfiles admin/cliente de
   Platform no son roles del producto. Sin evidencia de roles, omítelos.
   Reutiliza identidades y dependencias entre etapas en vez de duplicar casos.
6. Redacta contexto y preparación, pasos concretos, resultado visible, señales
   de fallo y cómo dejar observaciones. No incluyas rutas de código, SHAs,
   credenciales, análisis administrativo ni conclusiones legales en la guía.

## Capturar, revisar y aplicar

1. Llama `create_delivery_guide_prompt` con selección explícita, versión vigente
   y `request_id` estable. Crear el contexto es escritura, aunque no publique.
   Usa la plantilla y el esquema devueltos, no un ejemplo estático como API.
2. Genera JSON v2 con su `context_id` y citas exactas `source_key`, `locator`,
   `quote` de las copias capturadas. Cada requerimiento conserva fundamento
   contractual y la referencia de implementación aplicable. Repo/SHA:file:line
   por sí solo no es una cita MCP. No inventes un respaldo contractual para
   importar una conducta adicional: conserva la guía preliminar y la brecha
   cuando las fuentes no permitan el registro trazado.
3. Conserva contrato/otrosí, claves y contenido aprobado. No importa estados,
   firmas, publicaciones ni decisiones. Un ancestro publicado acompaña al JSON
   sólo sin cambiar sus campos. Corregir una guía publicada pendiente requiere
   su editor y otra ronda, no importarla encima del historial.
4. Ejecuta `preview_delivery_import` y muestra jerarquía, efectos, advertencias
   y brechas. Cambiar JSON, fuentes o versión invalida ese preview. La validación
   estructural no certifica interpretación contractual ni validación viva.
5. Aplica **ese** payload como borradores con `apply_delivery_import` y la
   confirmación MCP vigente. Una respuesta incierta conserva `request_id` y
   contenido; consulta o reintenta idempotentemente. Un conflicto exige releer
   y previsualizar de nuevo, sin sobrescribir a ciegas.
6. Comprueba borradores, fuentes, identidades y conformidades anteriores.
   Detente: publicar y registrar resultados pertenecen a otro paso.

## Respuesta y límites

Devuelve proyecto/contrato, modo, resumen de guías, matriz de brechas y evidencia
consultada. Si hubo escrituras, incluye inventario/contexto, versión, preview y
resultado de aplicación. El cliente sólo recibe instrucciones aptas para probar.

Sin contrato, MCP, permiso, referencias o fuentes suficientes, entrega lo
preliminar verificable y el faltante concreto. No busques secretos, inventes
IDs, cambies modelos directamente ni uses JSON v1 para eludir trazabilidad.
La importación no reemplaza una auditoría del sistema del cliente ni modifica
su código. Una observación nueva conserva etapa, versión y ronda originales;
el agradecimiento o una constancia del equipo no acreditan aceptación total.
