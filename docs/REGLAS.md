# Reglas de Firestore para pegar

Cada fase que suma una colección deja acá su bloque. Se pegan a mano en la consola de Firebase
(**Firestore Database → Reglas**), **sin borrar las reglas que ya están**, y se toca **Publicar**.
Hacerlo antes del `git push` de esa fase.

## Fase 1 — `comentarios`

Pegá este bloque **adentro** de `match /databases/{database}/documents { … }`, al lado de los otros
`match` (por ejemplo, justo antes de la última `}`).

```
    match /comentarios/{id} {
      // Mail de Google → persona del equipo (los mismos de EQUIPO en index.html).
      function miId() {
        return request.auth == null ? '' : {
          'manucontreras3@gmail.com': 'manu',
          'colanskymica@gmail.com': 'mica',
          'agustinan.contacto@gmail.com': 'agus',
          'luchixo34@gmail.com': 'luciano',
          'salvacontre.c@gmail.com': 'salva',
          'bishelsalexismaximiliano@gmail.com': 'maxi',
          'maxterom@gmail.com': 'santi'
        }.get(request.auth.token.email.lower(), '');
      }
      function esEquipo() { return miId() != ''; }
      function esAdmin() { return miId() in ['manu', 'mica', 'agus']; }

      // Leer: cualquiera del equipo.
      allow read: if esEquipo();
      // Crear: solo a nombre propio, y sin venir ya resuelto.
      allow create: if esEquipo()
        && request.resource.data.de == miId()
        && request.resource.data.resuelto == false;
      // Resolver un pendiente: quien lo pidió o a quien se le pidió. Solo cambia los campos de "resuelto".
      allow update: if miId() in [resource.data.de, resource.data.para]
        && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['resuelto', 'resueltoPor', 'resueltoTs'])
        && request.resource.data.resueltoPor == miId();
      // Borrar: cada uno lo suyo. Manu, Mica y Agustina también, porque al borrar una tarea se borran sus comentarios.
      allow delete: if miId() == resource.data.de || esAdmin();
    }
```

**Si se suma alguien al equipo** (o cambia un mail), hay que agregarlo también en la lista de `miId()`.

Cómo probar que quedó bien: entrar a Marcebra, abrir una tarea y mandar un comentario. Si aparece
"Los comentarios todavía no están activados" o "No tenés permiso para comentar", la regla no se publicó.
