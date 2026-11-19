# Marcebra — plan de mejoras (tanda noviembre 2026)

Marcebra es el panel del equipo de marketing de Zebra Club. Se usa ya, todos los días, y más adelante se muda al super admin (`sistema-zebra`, Laravel + Filament). Esta tanda lo pule para el uso inmediato y deja la lógica clara para esa mudanza.

Antes de tocar nada, leé `CLAUDE.md` (reglas de trabajo) y mirá la maqueta en `docs/maqueta/` (una pantalla por archivo `.dc.html`; el HTML de adentro muestra la estructura y los textos, la lógica de ejemplo está en el `<script>` del final). La maqueta es la referencia visual y de comportamiento. Los datos son de ejemplo.

## Quién pidió qué

Agustina (18/11), en sus palabras:
1. Las pautas no se ordenan por fecha y no se sabe en qué concluyen.
2. Hay que hacerlo más digerible visualmente para que entren a completarlo; la experiencia es tediosa.
3. Aparecen muchas cosas y se pierde la data más importante.
4. Tiene que ser más didáctico.
5. Que cada persona tenga un máximo de tareas posible de cumplir.
6. Poder arrastrar las tareas a otro día en el calendario.
7. Comentarios en la tarea, tanto del que pide como del que hace, y un lugar para pedir algo que queda pendiente o por confirmar (ej. el diseñador necesita el set time).
8. Que la hoja de ruta sea más visual, que den ganas de completarla y se vea el progreso.

Manu: una sección de **Reuniones** del equipo de marketing para anotar lo que se habla, buscar después e ir haciendo balance de los acuerdos.

## Decisiones ya tomadas (no reabrir)

- **Pauta sin API de Meta.** Carga manual. La conexión con Meta queda para el super admin, que tiene servidor; acá una clave quedaría pública.
- **El tope de tareas bloquea.** Si una persona llegó a su tope, no se le puede asignar otra tarea. La salida es que Mica o Agustina le suban el tope.
- **Reuniones es solo para marketing.** La ven y editan Manu, Mica y Agustina. Producción no la ve. Reuniones con encargados, RRPP o dueños no van acá.
- **Temas deja de ser una sección.** Los temas abiertos se muestran como "Temas para la próxima" dentro de Reuniones. Los datos de `temas` no se migran ni se borran.
- **Tablero queda igual.**

## Cómo está hoy el código (lo que importa para este plan)

- Todo vive en `index.html` (HTML + CSS + JS, sin build). Firebase compat 10.12.2: Auth con Google y Firestore, proyecto `zebra-mkt`.
- `EQUIPO` está escrito en el código (ids `manu`, `mica`, `agus`, `luciano`, `salva`, `maxi`, `santi`). Hay además `freelancers` en Firestore.
- `S` guarda las colecciones en memoria: `tareas`, `fechas`, `campanias`, `freelancers`, `temas` (incluye proyectos de la hoja de ruta con `tipo: 'proyecto'`), `notas`.
- **`guardar()` escribe cada documento entero con `.set()`**, comparando contra `ultimo`. Si dos personas editan el mismo documento a la vez, gana la última. Por eso todo lo que escriben varias personas sobre una misma cosa (comentarios) va en **documentos propios**, no como array dentro de la tarea.
- Las vistas están en `VISTAS` y se dibujan en `render()` con el helper `h()`. Avisos/novedades: `avisar()` / `registrarAviso()`.
- El calendario mezcla `tareas` (fecha = `entrega`) y `fechas` (fecha = `fecha`); ver `trabajos()`.
- Estados de tarea: `pedido` (Sin empezar), `proceso`, `revision`, `aprobado` (Para publicar), `publicado`.

## Fases

Cada fase se termina, se prueba en `http://localhost:5500` y se commitea por separado. Manu decide cuándo se hace `git push` (eso publica para el equipo). No arranques la fase siguiente sin que Manu diga que la anterior está bien.

Las fases 1 a 3 son de funcionamiento y se pueden hacer ya. Las fases 4 a 7 cambian pantallas: antes de empezarlas, preguntale a Manu si hay ajustes de Mica y Agustina sobre la maqueta.

### Fase 1 — Comentarios y pendientes en la tarea

Pantalla de referencia: `docs/maqueta/Tarea.dc.html`.

- Colección nueva **`comentarios`**: `{ id, tarea, de, texto, ts, pendiente: bool, para, resuelto: bool, resueltoPor, resueltoTs }`. `tarea` es el id de la tarea o de la fecha del calendario.
- Se escucha completa con `onSnapshot` (es poca data) y se agrupa por tarea en memoria. Cada comentario se crea con `.add()`/`.set()` propio; resolver un pendiente es un `.update()` de ese documento. No pasa por `guardar()`.
- En el modal de la tarea (`abrirTarea`, y también `abrirFecha`): bloque "Conversación" con los comentarios en orden, caja para escribir, casilla "Es un pedido que queda pendiente" y selector "A quién" (cualquier persona del equipo).
- **Tarea trabada:** si tiene un pendiente sin resolver, se muestra una franja roja arriba ("Luciano espera a Mica: …") con botón "Ya lo resolví", que puede tocar la persona a la que se le pidió o quien lo pidió. En listas, tablero y calendario la tarea muestra la etiqueta **Trabada**.
- Avisos: comentar avisa a quien pidió y a los responsables (menos al que escribe). Un pendiente avisa a `para`. Usar `avisar()` con tipos nuevos (`comentario`, `pendiente`, `resuelto`) y su texto en `TXT_AVISO`.
- Las `devoluciones` de revisión siguen como están.
- Al borrar una tarea, borrar sus comentarios (igual que hoy se hace con `imagenes`).
- Permisos: lee y escribe cualquier persona del equipo logueada; cada uno solo borra lo suyo.

### Fase 2 — Tope de tareas por persona

Pantalla de referencia: `docs/maqueta/Equipo.dc.html`.

- **Tareas activas** = tareas donde la persona está en `resps` y el estado es `pedido`, `proceso` o `revision`. Las `fechas` del calendario no cuentan.
- Tope guardado en Firestore, documento **`config/equipo`**: `{ topes: { luciano: 5, ... } }`. Default si no hay valor: 5 para producción y 6 para Mica y Agustina. Lo editan solo Mica, Agustina y Manu.
- **Bloqueo:** en todo lugar donde se agrega una persona a `resps` (selector de personas de `abrirTarea`, duplicar, crear tarea desde un tema o un paso de la hoja de ruta), una persona al tope aparece deshabilitada con "Al tope (5/5)". Si se intenta, mensaje: "X está al tope. Elegí a otra persona o pedile que cierre algo antes." Las tareas que ya estaban asignadas no se tocan aunque alguien quede pasado del tope.
- La validación es en la pantalla (no en reglas de Firestore).
- Vista **Equipo y carga** (reemplaza Carga del equipo y Equipo): una fila por persona con casilleros (sin empezar / en proceso / en revisión / libres), estado ("Al tope", "Le queda 1"), botones − y + del tope, y entregadas a tiempo en la semana. El Balance que existe pasa a ser una pestaña dentro de esta vista, sin cambios de lógica.

### Fase 3 — Arrastrar en el calendario

Pantalla de referencia: `docs/maqueta/Calendario.dc.html`.

- En escritorio: drag & drop nativo de HTML5 sobre las vistas Semana y Mes. Al soltar en otro día: en una tarea cambia `entrega`; en una fecha del calendario cambia `fecha`. Resaltar el día de destino mientras se arrastra.
- En el celu (pantalla táctil): tocar la tarjeta abre el detalle como hoy; agregar ahí un botón **Mover a…** con un selector de fecha.
- Puede mover quien hoy puede editar (`puedeMover`). Avisar a los responsables: "Mica movió X al vie 20/11".
- Toast de confirmación con **Deshacer** durante unos segundos.

### Fase 4 — Inicio, Lo mío y menú más corto

Pantallas de referencia: `docs/maqueta/Main.dc.html` (Mica/Agus/Manu) y `docs/maqueta/LoMio.dc.html` (producción).

- Menú para Manu, Mica y Agustina: **Inicio, Calendario, Tablero, Equipo y carga, Pauta, Hoja de ruta, Reuniones**. Resumen y Novedades se absorben en Inicio (la campanita sigue igual). Temas sale del menú (ver Fase 7).
- Menú para producción: **Lo mío, Calendario**, y **Pauta** solo si `puedePauta()` (Santi).
- **Inicio (admins):** contadores que filtran (Para revisar, Te piden = pendientes de comentarios dirigidos a vos, Vencidas, Gente al tope) y una lista "Necesita tu atención" con un botón de acción por fila. Debajo, "Sale esta semana" agrupado por día. Al costado: carga del equipo, próxima reunión y una idea de la hoja de ruta.
- **Lo mío (producción):** barra de carga con el tope, "Esperando a otros" (sus pendientes abiertos), "Para hacer" ordenado por entrega con **un solo botón de próximo paso** por tarea (Empezar → Mandar a revisión; si está en revisión, texto "Lo está revisando…"; si tiene devolución, se avisa). Abajo, sus novedades.
- **Didáctico:** cada vista con una línea corta de qué es y qué hacer; los vacíos explican qué va ahí. Nada de texto largo.
- Menos ruido: sacar de las tarjetas lo que no ayuda a decidir. En la tarjeta del calendario solo título, responsable, estado, formato y prioridad (urgente).

### Fase 5 — Pauta por fecha y conclusión

Pantalla de referencia: `docs/maqueta/Pauta.dc.html`.

- Campañas ordenadas por `inicio`, la más reciente arriba, agrupadas por mes.
- Pestañas: Todas, Activas, **Falta conclusión**, Cerradas.
- Campo nuevo en la campaña: `conclusion: { texto, decision: 'repetir'|'ajustar'|'no', por, ts }`.
- Estado derivado: activa si hoy está entre `inicio` y `fin`; **falta conclusión** si ya pasó `fin` y no tiene `conclusion`; cerrada si la tiene.
- En las de "Falta conclusión": botón "Escribir conclusión", texto + decisión. Las cerradas muestran la conclusión con su etiqueta.
- Mantener la carga de métricas y notas que ya existe.

### Fase 6 — Hoja de ruta visual

Pantalla de referencia: `docs/maqueta/HojaDeRuta.dc.html`.

- Arriba: avance del mes (pasos hechos / total) y cuántos pasos se tildaron esta semana.
- Cada proyecto como tarjeta: anillo de progreso, "4 de 7 pasos", el **próximo paso** con su fecha, y responsables. Agrupadas por horizonte (Ahora, Próximo…).
- Al tocar una tarjeta se abren sus pasos para tildar; al tildar, la barra se mueve y aparece un mensaje corto ("¡Bien! Faltan 3 pasos").
- Usa los datos que ya existen (`temas` con `tipo: 'proyecto'`, `pasos`). Para "esta semana" hace falta guardar cuándo se tildó cada paso (`hechoTs`) si hoy no se guarda.

### Fase 7 — Reuniones

Pantalla de referencia: `docs/maqueta/Reuniones.dc.html`.

- Colección nueva **`reuniones`**: `{ id, fecha, titulo, asistentes: [ids], notas: texto, acuerdos: [{ id, texto, resp, fecha, hecho, hechoTs }], creadoPor }`. Si dos personas editan la misma reunión a la vez puede pisarse; es aceptable porque la edita casi siempre una persona.
- Solo Manu, Mica y Agustina (reglas de Firestore).
- Arriba: acuerdos cumplidos / pendientes / vencidos y reuniones del mes.
- Buscador en título, notas y acuerdos (sin distinguir tildes ni mayúsculas), con el fragmento que coincide.
- Detalle: asistentes, "Quedó pendiente de la reunión anterior" (acuerdos sin cumplir de la reunión previa), qué se habló, acuerdos tildables.
- Un acuerdo puede convertirse en tarea con un botón (abre `abrirTarea` con el texto y la fecha; respeta el tope).
- **Temas para la próxima:** lista de los `temas` abiertos (no proyectos, no decididos) con su detalle; desde ahí se pueden abrir y decidir como hoy.

## Reglas de Firestore

Cada fase que agregue una colección deja el bloque de reglas a agregar en `docs/REGLAS.md`, explicado para pegar en la consola de Firebase **sin borrar las reglas que ya están**. Manu las pega a mano antes del `git push` de esa fase.

## Al terminar todo

Actualizar `docs/MUDANZA.md` con lo que el super admin tiene que copiar de Marcebra: colecciones y campos nuevos, reglas de tope y bloqueo, flujo de pendientes, conclusión de pauta y reuniones.
