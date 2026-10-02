# El formulario «Solicitar demo» de una web estática

Cómo se armó el formulario de https://recompensalo.com. Está escrito para
pasárselo a otra sesión que tenga que hacer lo mismo en otra web, por ejemplo
respondelo.app.

## La idea en una línea

La web es **estática** (GitHub Pages), así que no tiene dónde recibir datos.
Un **Worker de Cloudflare** recibe el formulario, lo guarda en un **KV** y
sirve un **panel** para leerlo. Es el mismo patrón que el cuestionario de
mflowsuite.com (el Worker `cuestionario`).

```
navegador ──POST JSON──▶ Worker recompensalo-contacto ──▶ KV recompensalo-contactos
                                   │
                                   └─(opcional) Resend ──▶ mail a Martín
Martín ──▶ /panel del Worker (pide la clave) ──GET /api/contactos──▶ lista
```

## Las piezas

| Pieza | Dónde |
|---|---|
| Código del Worker | `worker/worker.js` de este repo (`mflowsuite/recompensalo-web`) |
| Worker publicado | `recompensalo-contacto`, en https://recompensalo-contacto.mflowsuite.workers.dev |
| Dónde se guardan los pedidos | KV `recompensalo-contactos`, id `8f88475ac9eb4364abbed596e04686d9` |
| Panel para leerlos | https://recompensalo-contacto.mflowsuite.workers.dev/panel |
| Clave del panel | `RECOMPENSALO_CONTACTOS_TOKEN` en `C:\Users\marti\.claude\session-env\credenciales.env` |
| Formulario (HTML, CSS y JS) | `index.html`: bloque `<div class="demo" id="demo">`, estilos «El formulario de demo» y el último `<script>` |

Credenciales que se usan, todas de `credenciales.env` (**nunca se imprimen ni
se pegan en el chat**):

- `CF_API_TOKEN`: el token «chatbot cloude» (Tokens de API de cuenta). Puede
  crear Workers y KV, y editar el DNS de mflowsuite.com, recompensalo.com y
  respondelo.app.
- `CF_ACCOUNT_ID`.

## El Worker (`worker/worker.js`)

Rutas:

- `POST /api/contacto`: **público**, porque lo llama la web y no hay forma de
  darle una clave sin dejarla escrita en la página. Lo que se protege es la
  lectura.
- `GET /api/contactos`: lista los pedidos. Pide `Authorization: Bearer <ADMIN_TOKEN>`.
- `GET /panel`: una página HTML que pide la clave, la recuerda en el
  navegador y muestra los pedidos con links a WhatsApp y al mail.
- `GET /salud`: devuelve `{ok:true}`.

Lo que hace al recibir un pedido:

1. **Cupo por IP**: 5 envíos cada 10 minutos. Se guarda en el mismo KV como
   `cupo:<ip>`, con `expirationTtl: 600`.
2. **Campo trampa** `empresa_web`: es invisible para las personas. Si viene
   lleno, contesta `{ok:true}` y no guarda nada, para que el bot no se entere.
3. **Valida**: nombre, negocio, WhatsApp con 8 dígitos o más y email. Si algo
   falla devuelve `{error, campo}`, para que la web marque ese campo.
4. **Guarda** en `c:<9999999999999 - timestamp>-<rand>`. La clave va invertida
   para que la lista de KV salga con el pedido más nuevo primero.
5. **Mail (opcional)**: si existen los secretos `RESEND_API_KEY` y `AVISO_A`,
   manda un mail por Resend desde `Recompensalo <hola@mail.mflowsuite.com>`,
   con reply-to al mail de quien escribió. Si el mail falla, el pedido igual
   queda guardado.

**CORS**: solo acepta `https://recompensalo.com`, `https://www.recompensalo.com`
y `http://localhost:<puerto>`, este último para probar en local. Para otra web,
cambiar la constante `ORIGENES`.

## Cómo se creó (por la API de Cloudflare, sin wrangler)

1. **El KV:**
   ```
   POST https://api.cloudflare.com/client/v4/accounts/<CF_ACCOUNT_ID>/storage/kv/namespaces
   {"title": "recompensalo-contactos"}
   ```
2. **La clave del panel:** `secrets.token_urlsafe(24)`, escrita directo al
   final de `credenciales.env` como `RECOMPENSALO_CONTACTOS_TOKEN=…`, sin
   mostrarla.
3. **Publicar el Worker**: `PUT …/workers/scripts/recompensalo-contacto` con un
   cuerpo `multipart/form-data` de dos partes:
   - `metadata` (JSON):
     ```json
     {"main_module": "worker.js", "compatibility_date": "2026-09-01",
      "bindings": [
        {"type": "kv_namespace", "name": "CONTACTOS", "namespace_id": "<id del KV>"},
        {"type": "secret_text", "name": "ADMIN_TOKEN", "text": "<la clave>"}
      ]}
     ```
   - `worker.js`: el archivo, con `Content-Type: application/javascript+module`.
4. **Encenderlo en workers.dev**: `POST …/workers/scripts/recompensalo-contacto/subdomain`
   con `{"enabled": true}`. El subdominio de la cuenta es `mflowsuite`.

Para **actualizar** el Worker se repite el paso 3 entero. Ojo: el `PUT`
reemplaza todos los bindings, así que hay que volver a mandar el KV y los
secretos. Si alguna vez se cargan `RESEND_API_KEY` y `AVISO_A`, también van ahí.

Todo esto se hizo con un script de Python con `urllib`. Las llamadas a
Cloudflare necesitan un `User-Agent` de navegador: con el de Python, Cloudflare
contesta 403 (código 1010).

## El formulario en la página

- Los campos son `nombre`, `negocio`, `whatsapp`, `email`, `mensaje`
  (opcional), `plan` (oculto) y la trampa `empresa_web`, dentro de un `div`
  `.trampa` que se saca de la pantalla con `left: -9999px`.
- Todo botón que pide demo apunta a `#demo`. Los «Consultar» de los planes
  llevan `data-plan="Crecimiento"`: al tocarlos se completa el campo oculto
  `plan` y, si el mensaje está vacío, se escribe «Me interesa el plan X.».
- El JS valida lo obligatorio antes de mandar, hace `fetch` con JSON al
  Worker, muestra el error que devuelve (y marca el campo en rojo) o cambia el
  formulario por «Listo. Te escribimos hoy.».
- El diseño copia el de la landing de Muestralo
  (`C:\dev\utopia-catalogo\src\components\landing\v7\contacto.tsx`): tarjeta
  blanca con bordes de 28 px, campos grises de 16 px de radio y botón en
  píldora con flecha.

## Cómo se probó

1. Con `curl` contra el Worker publicado:
   - un pedido incompleto devuelve 400 con `campo`;
   - sin clave, `/api/contactos` devuelve 401;
   - con la trampa llena devuelve `{ok:true}` y no se guarda nada;
   - el preflight `OPTIONS` desde `https://recompensalo.com` devuelve 200.
2. Desde la web en `localhost`, de punta a punta:
   - vacío: dice «Completá tu nombre» y pone el foco ahí;
   - WhatsApp corto: muestra el error del Worker y lo marca en rojo;
   - completo: aparece «Listo».
3. En el panel apareció el pedido con el plan marcado. Después se **borraron
   los datos de prueba** con `DELETE …/storage/kv/namespaces/<id>/values/<clave>`.
   KV tarda hasta un minuto en reflejar un borrado: si recién borrado todavía
   aparece, no es un error.

## Para hacerlo en otra web (respondelo.app, por ejemplo)

1. Copiar `worker/worker.js`. Cambiar `ORIGENES`, el remitente (`REMITENTE`),
   el asunto del mail y el título del panel.
2. Crear un KV nuevo, una clave nueva en `credenciales.env`
   (`RESPONDELO_CONTACTOS_TOKEN`, por ejemplo) y publicar con otro nombre de
   Worker. No compartir el KV entre productos: los pedidos se mezclarían.
3. Copiar del `index.html` el bloque `#demo`, sus estilos y su script, y
   cambiar `URL_CONTACTO` por la del Worker nuevo.
4. Probar como se describe arriba y borrar los datos de prueba.

## Lo que falta: el mail por cada pedido

El código ya está. Falta cargar dos secretos en el Worker:

- `RESEND_API_KEY`: la clave de Resend. Es la misma que usa el chatbot de
  Guzel y tiene que ser **Full access**.
- `AVISO_A`: a quién mandarlo. Si son varias direcciones, van separadas por
  coma.

Se cargan republicando con esos dos como `secret_text` en `bindings` (paso 3),
o desde el panel de Cloudflare: Workers → recompensalo-contacto → Settings →
Variables.
