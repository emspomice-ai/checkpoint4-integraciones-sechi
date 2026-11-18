# Checkpoint 4 · Integraciones avanzadas e interconexión de sistemas

**Sechi, Medical Rigging** · Proceso de negocio: Elaboración de Presupuestos de rigging de equipos médicos
Carrera *AI Automation Avanzado* (Coderhouse) · Autor: Emiliano Coronel Pomice

Archivo a importar en n8n: [`checkpoint4_emiliano_coronel-pomice.json`](./checkpoint4_emiliano_coronel-pomice.json)

## Qué hace
Parte del **Manager con memoria del Módulo 3** (sin rehacerlo) y le suma tres integraciones reales para que una solicitud de presupuesto entre sola al sistema, quede registrada y genere un borrador de respuesta que una persona revisa. Nada se envía al cliente de forma automática.

| Integración | Autenticación | Permisos | Rol en el flujo |
|---|---|---|---|
| Gmail | OAuth2 | Solo leer y crear borradores (no modifica ni envía) | Lee la solicitud reenviada por Logística y deja un **borrador** de respuesta |
| Airtable | OAuth2 | Bases autorizadas | Busca, crea y actualiza el contacto y el caso (tablas `clientes` y `casos`) |
| Slack | OAuth2 | Publicar en el canal de solicitudes | Avisa cada solicitud y cada correo no procesado |

Además de Gmail, el sistema recibe pedidos por un **formulario de n8n** (WhatsApp, teléfono u otros canales) y conserva el **chat** del Módulo 3 sin cambios.

## Flujo
1. **Entrada por mail:** Gmail Trigger → IF anti respuesta automática → un correo por ejecución → Set de payload limpio → IF de remitente y texto válidos → Agente Extractor (solo estructura datos, no cotiza) → validación en código.
2. **Entrada manual:** Formulario → normalización a la misma estructura que el mail.
3. **Contacto y caso:** búsqueda en Airtable → si existe se actualiza, si no se crea → el Nº de Presupuesto (autonumérico) arma el `sessionId` de la memoria (`PRES-000123`).
4. **Manager de M3:** clasificación, workers W1/W2/W3, memoria y resumen, ahora atados al caso y no al chat.
5. **Salida con humano en el circuito:** Gmail *Create Draft* → se actualiza el caso → Set de payload limpio → aviso a Slack.

## Decisiones de diseño
- **Seguridad y mínimo privilegio:** tres integraciones con OAuth2; Gmail limitado a leer y crear borradores, sin permiso para modificar ni enviar.
- **Contención de bucles y errores**
  - IF anti auto-reply inmediatamente después del Gmail Trigger (asuntos Auto-reply, Out of office, Undeliverable y remitentes no-reply o mailer-daemon).
  - Búsqueda previa antes de crear (contacto y caso), para evitar duplicados y el error 409.
  - Set que deja solo From, Subject y BodyText, y validación de remitente no vacío, para evitar el error 400.
  - Un correo por ejecución; los demás se listan en el aviso de Slack. Nada se descarta en silencio.
- **Humano en el circuito:** Gmail usa **solo Create Draft**. El borrador es básico (sin precios ni datos internos), termina en "Atte" y lo revisa una persona antes de enviarlo.

## Importar y configurar
1. n8n → *Import from file* → `checkpoint4_emiliano_coronel-pomice.json`.
2. El archivo **no contiene credenciales**. Seleccioná tus propias credenciales en los nodos de Gmail (Trigger y Create Draft), Airtable, Slack y Google Gemini.
3. Slack: elegí el canal de avisos en los dos nodos de Slack nuevos.
4. Airtable: el archivo apunta a las bases y tablas de Sechi (`clientes`, `casos`, `memoria_clientes`); para usarlo en otra cuenta hay que recrearlas con los campos indicados en las notas del lienzo.
5. Los workers W1, W2 y W3 del Módulo 2 deben estar publicados en la misma instancia.
6. El Manager corre con timeout de 180 s (límite del plan de n8n cloud utilizado).

## Pruebas realizadas en n8n
| Prueba | Resultado |
|---|---|
| Regresión del chat de M3 (2 mensajes, memoria recurrente, workers W1 a W3) | Sin cambios; sale por la rama "consulta por chat" sin acciones M4 |
| Formulario con datos de prueba | Crea contacto y caso, memoria `PRES-000001`, borrador y aviso en Slack |
| Correo "Out of office" | El IF lo desvía a *Ignorar*; no se crea caso ni borrador |
| Mail real reenviado (cadena RV con adjuntos) | Contacto y datos extraídos correctamente; cliente real tomado del cuerpo, no del remitente |
| Mismo mail por segunda vez | Actualiza el caso existente; no crea otro (Look up) |
| Falla del modelo del Extractor (modelo inexistente a propósito) | El flujo continúa y Slack avisa "correo recibido pero NO procesado"; no se crea caso ni borrador |
| Modelo que no figura en la Biblioteca | El Manager escala a Logística (`MODELO_NO_ENCONTRADO`) en lugar de inventar un peso |

Las capturas de las pruebas no se incluyen porque muestran datos reales de clientes.

## Detalles técnicos para auditoría

**Condiciones del IF anti auto-reply** (combinador O, sin distinguir mayúsculas). Se corta si el asunto contiene: `Auto-reply`, `Autoreply`, `Out of office`, `Undeliverable`, `Automatic reply`, `Respuesta automática`, `Fuera de la oficina` o `Delivery Status Notification`; o si el remitente contiene: `no-reply@`, `noreply@`, `mailer-daemon@` o `postmaster@`.

**Inyección dinámica en el prompt del Extractor.** El mensaje de usuario se arma con estas expresiones literales, y el correo va entre delimitadores para que el modelo lo trate como datos y no como instrucciones:

```
Fecha de hoy: {{ $now.setZone('America/Argentina/Buenos_Aires').toFormat('yyyy-MM-dd') }}
Asunto: {{ $json.Subject }}
<<<INICIO_CORREO>>>
{{ $json.BodyText }}
<<<FIN_CORREO>>>
```

**Contrato de salida del Extractor** (JSON puro; lo valida un nodo de código antes de usarlo):

| Campo | Tipo |
|---|---|
| `es_solicitud` | booleano |
| `cantidad_pedidos_detectados` | número entero |
| `empresa` | texto (lista cerrada de clientes o texto libre) o null |
| `contacto_nombre`, `contacto_email`, `contacto_celular` | texto o null |
| `contacto_puesto` | **lista (array) de textos**; valores permitidos: Ventas, Logística, Bioingeniería |
| `cliente_final`, `referencia_cliente`, `equipo`, `modelo` | texto o null |
| `origen`, `destino`, `localidad_destino`, `provincia_destino` | texto o null |
| `fecha_pedida` | texto o null |
| `fecha_exacta` | texto `AAAA-MM-DD` o null |
| `cantidad_bultos`, `peso_bulto_mas_pesado_kg` | número o null |
| `detalle_bultos`, `condiciones_acceso` | texto o null |
| `menciona_fotos_del_camino` | booleano |

**Esquema de datos en Airtable** (base "Clientes y Casos"):

| Tabla `clientes` | Tipo |
|---|---|
| Nombre (campo primario) | texto |
| Empresa | selección única |
| Puesto | selección múltiple (lista) |
| Email | email |
| Celular | teléfono |
| Notas | texto largo |
| Origen del dato | selección única (Creado por el agente / Cargado a mano) |
| casos | vínculo inverso |

| Tabla `casos` | Tipo |
|---|---|
| Nº de Presupuesto (campo primario) | autonumérico |
| Asunto normalizado, Referencia del cliente, Cliente final, Equipo, Modelo, Origen, Destino, Fecha pedida | texto |
| Mails en la cadena, Cantidad de bultos, Peso del bulto más pesado (kg) | número |
| Contacto | vínculo a `clientes` |
| Fecha exacta | fecha |
| Detalle de bultos, Condiciones de acceso, Datos que faltan, Resultado del Manager | texto largo |
| Estado del caso | selección única |
| Origen de la solicitud | selección única (Mail / WhatsApp / Teléfono / Otro) |
| Borrador creado | casilla de verificación |
| Última actualización / Primer mail | fecha de modificación / de creación (automáticos) |
| Empresa | búsqueda (lookup) desde el contacto |

**Manejo de fallas.**
- El Extractor reintenta 3 veces y, si sigue fallando, el flujo continúa por su salida normal: el nodo de validación lo trata como lectura inválida y Slack avisa. El pedido no se pierde en silencio.
- Si el Manager no puede resolver un caso, escala a una persona. Ejemplo real de contingencia del Worker 1 (modelo que no está en la Biblioteca): `Escalado a Logística: MODELO_NO_ENCONTRADO`, con el caso en "Datos incompletos". El sistema nunca escribe "no haría falta tercero" si el Worker 2 no respondió con éxito.

## Privacidad
Este repositorio no contiene correos, nombres, teléfonos ni credenciales de clientes o terceros. Los datos reales viven solo en la base de Airtable de la empresa.

## Limitaciones conocidas
- Si un correo trae varios pedidos, se procesa solo el primero y se avisa por Slack.
- El mail llega reenviado a mano a la casilla del agente (la casilla de Logística está en otro proveedor).
- El Manager depende de la Biblioteca de modelos y no usa el peso que el cliente escribe en el mail; si el modelo no está, escala a una persona.
- El Clasificador del Manager de M3 no es determinista: a veces escala una consulta válida.
- El extractor devuelve texto; una validación en código descarta lo inválido en vez de confiar en el modelo.
