# Mudanza de Marcebra al super admin

Qué tiene que copiar `sistema-zebra` (Laravel + Filament) de Marcebra después de la tanda de noviembre 2026.
Marcebra es un solo `index.html` con Firebase (proyecto `zebra-mkt`). Este documento junta los datos, las
reglas de negocio y las pantallas que hay que reproducir. Las reglas de Firestore de cada colección están en
`docs/REGLAS.md`. Lo que todavía tienen que validar Mica y Agustina está en `docs/REVISAR.md` (resumen al final).

## Quién es quién

- **Equipo fijo** (en el código, `EQUIPO`): `manu` (jefe), `mica` y `agus` (admins), `luciano`, `salva`, `maxi`
  (producción) y `santi` (pauta). Cada uno entra con su mail de Google; el mail define la persona.
- **Freelancers** (colección `freelancers`): no entran al panel. Se les asignan tareas y cuentan para el tope.
- **Permisos:** Manu, Mica y Agustina ven y editan todo. Producción ve solo lo suyo y el calendario; Santi
  también ve y carga datos de Pauta. Reuniones y Hoja de ruta son solo de Manu, Mica y Agustina.

## Colecciones y campos

Regla general de Marcebra: **nunca se migró ni se borró nada**. Los campos nuevos son opcionales y la app pone
un valor por defecto si faltan. Al importar, tratar cada campo nuevo como opcional.

### Nuevas

| Colección | Documento | Quién lee / escribe |
|---|---|---|
| `comentarios` | uno por comentario | todo el equipo lee y comenta; cada uno borra lo suyo |
| `config/equipo` | uno solo | todos leen; Manu, Mica y Agustina cambian topes |
| `reuniones` | uno por reunión | solo Manu, Mica y Agustina |

**`comentarios`** — `{ tarea, de, texto, ts, pendiente, para, resuelto, resueltoPor, resueltoTs }`
- `tarea`: id de una tarea **o** de una fecha del calendario (`fechas`). En el super admin conviene una relación
  polimórfica.
- `de`, `para`, `resueltoPor`: id de persona. `para` vacío si no es pendiente.
- `ts`, `resueltoTs`: milisegundos (epoch).
- Va en documentos propios (no adentro de la tarea) porque varias personas escriben a la vez sobre la misma
  tarea. En una base relacional es simplemente su propia tabla.

**`config/equipo`** — `{ topes: { <idPersona>: número } }`. Solo están los topes que alguien cambió.

**`reuniones`** — `{ fecha, titulo, asistentes: [ids], notas, acuerdos: [...], creadoPor, creado }`
- `notas`: texto libre, una idea por renglón (la pantalla lo muestra como lista).
- `acuerdos`: `[{ id, texto, resp, fecha, hecho, hechoTs, tarea }]`. `tarea` es el id de la tarea creada desde
  ese acuerdo (si se creó). En el super admin, `acuerdos` conviene que sea su propia tabla.

### Campos nuevos en colecciones que ya existían

| Colección | Campo | Qué es |
|---|---|---|
| `tareas` | `reunionId`, `acuerdoId` | si la tarea salió de un acuerdo de reunión |
| `tareas` / `fechas` | `log[].tipo` nuevos | `comentario`, `pendiente`, `resuelto`, `movida` (ver Avisos) |
| `campanias` | `conclusion` | `{ texto, decision: 'repetir' \| 'ajustar' \| 'no', por, ts }` |
| `temas` (proyectos) | `pasos[].hechoTs` | momento exacto en que se tildó el paso (epoch ms) |

Ya existían y se siguen usando igual: `tareas.entregadaEl` (día en que pasó a revisión o más), `tareas.devoluciones`,
`tareas.temaId`, `temas.pedidos`, `pasos[].hechoEl` (día en que se tildó; los pasos viejos solo tienen este).

**Ojo con `temas`:** guarda dos cosas distintas. Los temas de charla (`tipo: 'urgente'`, estados `abierto`,
`charla`, `decidido`, con `decision` y `decididoEl`) y los **proyectos de la hoja de ruta** (`tipo: 'proyecto'`,
con `pasos`, `horizonte`, `estado`, `marca`, `objetivo`, `mes`, `notas`). En el super admin conviene separarlos en
dos tablas.

## Reglas de negocio

### Tope de tareas y bloqueo

- **Tareas activas** de una persona = tareas donde está entre los responsables (`resps`) y el estado es `pedido`,
  `proceso` o `revision`. Las fechas del calendario no cuentan.
- **Tope:** el de `config/equipo`; si no hay, **5** para producción y freelancers, **6** para Mica y Agustina.
  **Manu no tiene tope.** Se ajusta entre 1 y 20 con − y + en Equipo y carga.
- **Bloqueo:** en todo lugar donde se agrega un responsable (crear o editar tarea, duplicar, pasar un tema, un
  paso de la hoja de ruta o un acuerdo a tarea), quien está al tope aparece deshabilitado ("Al tope (5/5)") y no
  se puede elegir. Mensaje: "X está al tope. Elegí a otra persona o pedile que cierre algo antes."
- Las tareas ya asignadas no se tocan aunque alguien quede pasado del tope.
- Hoy se valida **solo en la pantalla**. En el super admin conviene validarlo también en el servidor.

### Pendientes en la conversación de la tarea

1. Cualquiera del equipo comenta en una tarea o fecha. Si marca "Es un pedido que queda pendiente" y elige
   **a quién**, el comentario queda `pendiente: true, resuelto: false`.
2. Mientras haya un pendiente sin resolver, la tarea está **Trabada**: franja roja arriba del detalle
   ("Luciano espera a Mica: …") y etiqueta "Trabada" en listas, tablero, calendario, Inicio y Lo mío.
3. Lo resuelve ("Ya lo resolví") **quien lo pidió o a quien se le pidió**. Solo cambian `resuelto`,
   `resueltoPor` y `resueltoTs`.
4. Avisos: un comentario avisa a quien pidió la tarea y a los responsables (menos al que escribe); un pendiente
   avisa a `para`; resolver avisa a los dos. (Hoy el aviso solo se registra si quien escribe es admin o responsable
   de la tarea, porque el `log` vive adentro de la tarea; en el super admin no hace falta esa limitación.)
5. Al borrar una tarea o fecha se borran sus comentarios (y las imágenes, como antes).
6. Las `devoluciones` de revisión siguen aparte (son el "devolver con cambios" de quien revisa) y se muestran
   mezcladas en la misma conversación.

### Mover fechas en el calendario

- Solo Manu, Mica y Agustina, y no en tareas ya publicadas. En la compu se arrastra; en el celu (y también en la
  compu) está "Mover a…" en el detalle.
- Cada tarjeta mueve su fecha: la de entrega cambia `entrega`, la de publicación cambia `publica`, una fecha del
  calendario cambia `fecha`. Si entrega y publicación eran el mismo día se mueven juntas; la entrega nunca queda
  después de la publicación.
- Avisa a los responsables (`log` tipo `movida`) y deja **Deshacer** unos segundos.

### Conclusión de pauta

- El estado de la campaña **no se guarda, se calcula**: *programada* (no empezó), *activa* (hoy entre `inicio` y
  `fin`), *falta conclusión* (pasó `fin` y no tiene `conclusion`), *cerrada* (tiene `conclusion` con texto).
- Escriben la conclusión Manu, Mica, Agustina y Santi: texto + decisión para la próxima (repetir, ajustar o no
  repetir).
- La lista va por `inicio`, la más reciente arriba, agrupada por mes, con pestañas Todas / Activas / Falta
  conclusión / Cerradas. El menú muestra cuántas faltan concluir.

### Reuniones

- Solo marketing (Manu, Mica, Agustina). Reuniones con encargados, RRPP o dueños no van acá.
- Contadores: acuerdos cumplidos / pendientes / **vencidos** (con fecha pasada y sin cumplir) y reuniones del mes.
- Buscador en título, notas y acuerdos, sin distinguir tildes ni mayúsculas, mostrando el pedacito que coincide.
- En el detalle: "Quedó pendiente de la reunión anterior" = acuerdos sin cumplir de la reunión previa por fecha.
- **Pasar a tarea:** crea una tarea con el texto, la fecha y el responsable del acuerdo (respeta el tope). Al
  guardarla se enlazan las dos (`tarea.reunionId/acuerdoId` y `acuerdo.tarea`).
- **Temas para la próxima:** los temas abiertos (no proyectos, no decididos). Abajo, desplegable **Temas
  decididos** con la decisión de cada uno.
- Si dos editan la misma reunión a la vez gana el último. Se aceptó porque casi siempre la edita una persona;
  en el super admin, con acuerdos en su propia tabla, desaparece el problema.

### Hoja de ruta

- Avance del mes = pasos con fecha en el mes (hechos / total), sin contar proyectos pausados.
- "Esta semana" = pasos tildados desde el lunes (usa `hechoTs`, o `hechoEl` en los viejos).
- Al tildar el último paso el proyecto **no** pasa solo a Terminado: se marca a mano.

## Pantallas a reproducir

- **Inicio** (Manu, Mica, Agustina): contadores que filtran (Para revisar, Te piden, Vencidas, Gente al tope),
  lista "Necesita tu atención" con un botón por fila, "Sale esta semana" (hoy + 6 días, por fecha de publicación
  o de entrega) y al costado carga del equipo, próxima reunión y la idea de la hoja de ruta con el próximo paso
  más cercano.
- **Lo mío** (producción): barra de carga con el tope, "Esperando a otros" / "Te piden", "Para hacer" por fecha de
  entrega con **un solo botón de próximo paso** (Empezar → Mandar a revisión; en revisión, "Lo está revisando…";
  trabada, "esperás a X"), y sus novedades.
- **Equipo y carga:** pestaña Carga (casilleros por persona con − y + del tope), Balance (por período: cumplidas,
  sin cumplir, a tiempo, devoluciones recibidas, horas reales vs. estimadas, anotaciones) y Horas.
- **Calendario:** tarjeta con título, responsable, estado, formato y si es urgente; arrastrar para mover.
- **Pauta**, **Hoja de ruta** y **Reuniones** como se describe arriba. **Novedades** es la campanita de arriba.
- **Tablero** quedó igual que antes.

## Avisos (novedades)

No hay colección de avisos: cada tarea, fecha, campaña o tema guarda sus últimos 30 en `log`
(`{ ts, autor, tipo, para: [ids], txt?, que? }`). La campanita junta los dirigidos a cada uno; lo leído se
recuerda en el dispositivo. Después de cada acción se ofrece mandar el aviso por WhatsApp con el texto armado.
Tipos: `asignada`, `revision`, `devuelta`, `aprobada`, `publicada`, `paso`, `proyecto`, `dato`, y los nuevos
`comentario`, `pendiente`, `resuelto`, `movida`. En el super admin esto pide una tabla de notificaciones propia
(con leído por usuario, no por dispositivo).

## Pendiente de validar (de `docs/REVISAR.md`)

Decisiones que se tomaron por simples y que Mica y Agustina todavía tienen que confirmar. Antes de copiarlas,
revisar ese archivo por si cambiaron:

- "Mis tareas" quedó como **Lista de tareas** (botón en Inicio), fuera del menú.
- **Resumen** desapareció; lo que mostraba se repartió entre Inicio y Equipo y carga → Balance (ahora por período,
  no "últimos 30 días").
- **Temas** salió del menú; vive en Reuniones (abiertos y decididos).
- Producción ya no ve Tablero ni Equipo y carga.
- Inicio → Vencidas incluye las de Mica y Agustina.
- Lo mío no muestra las tareas "Para publicar".
- El calendario solo resalta la prioridad Urgente (Alta no).
- Hoja de ruta: abre en "Ideas", la vista por día quedó como pestaña; avance del mes solo cuenta pasos con fecha;
  el último paso no cierra el proyecto solo.
