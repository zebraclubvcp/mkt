# Marcebra — reglas de trabajo

Panel del equipo de marketing de Zebra Club. Lo usan todos los días Manu (jefe), Mica y Agustina (cargan y revisan) y producción (Luciano, Salvador, Maxi, Santi y freelancers), sobre todo desde el celular.

El plan de trabajo está en `docs/PLAN.md`. Seguilo fase por fase.

## Cómo es el proyecto

- Un solo `index.html` con HTML, CSS y JS. Sin build, sin npm, sin frameworks. Mantenerlo así.
- Firebase compat 10.12.2 desde CDN (Auth con Google + Firestore), proyecto `zebra-mkt`. La config de Firebase en el código es pública por diseño; no agregues ninguna otra clave.
- Se publica en GitHub Pages: `git push` a `main` = el equipo ya lo ve en zebraclubvcp.github.io/mkt/.
- `vip.html` no se toca.

## Reglas

1. **No romper los datos que ya existen.** Solo se agregan campos o colecciones. Nada que obligue a migrar o que deje tareas viejas sin mostrarse. Si un campo nuevo falta, la app usa un valor por defecto.
2. **Cada fase en su propio commit**, con mensaje en español. **Nunca hagas `git push` sin que Manu lo pida.**
3. Probar en `http://localhost:5500` (Live Server). Tiene que ser `localhost`: con `127.0.0.1` el login de Google falla. Probar también en ancho de celular (390 px).
4. Antes de las acciones que no se pueden deshacer (borrar), doble confirmación: aviso y segunda confirmación.
5. Toda la interfaz en español rioplatense, textos cortos y claros. Nada de jerga técnica para el equipo.
6. Respetar el estilo actual: variables CSS de `:root`, tipografía Onest, modo oscuro. No sumar colores sueltos.
7. Lo que varias personas escriben sobre lo mismo va en documentos propios (ver `guardar()`: escribe documentos enteros).
8. Si algo del plan no cierra con el código, o hay una decisión de diseño no menor, pará y preguntale a Manu antes de seguir.
9. Al terminar cada fase, decile a Manu en pocas líneas qué cambió, cómo probarlo y si hay reglas de Firestore para pegar.
