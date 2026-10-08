# Documentación de la API de Enlaces de Pago

[English Version](../payment-links.md)

Esta documentación proporciona detalles para utilizar la API de Enlaces de Pago para crear y gestionar enlaces de pago de forma programática.

## Entornos

| Entorno | URL base |
|---------|----------|
| Producción | `https://arnipay.com.py/api/v1/` |
| Sandbox | `https://sandbox.arnipay.com.py/api/v1/` |

Los ejemplos de endpoints usan el host de producción. Sandbox usa las mismas rutas en `https://sandbox.arnipay.com.py`. Los enlaces de checkout que devuelve la API de sandbox usan ese host (por ejemplo `https://sandbox.arnipay.com.py/checkout/{id}`).

## Autenticación

Todas las solicitudes a la API requieren autenticación basada en firma utilizando sus credenciales de Comercio. Los siguientes encabezados deben incluirse con **cada** solicitud:

| Nombre del Encabezado | Descripción |
|-------------|-------------|
| `X-Client-ID` | Su ID de cliente de Comercio (formato UUID) |
| `X-Timestamp` | Marca de tiempo Unix actual como entero (segundos desde el 1 de enero de 1970 UTC) |
| `X-Signature` | Firma HMAC-SHA256 para verificación de solicitud |

Las solicitudes expiran a los 15 minutos desde el valor indicado en `X-Timestamp`.

### Cadena Canónica y Generación de Firma

Utilizado tanto para solicitudes de API como para verificar webhooks.

1.  **Construya los componentes canónicos:**
    1.  Método HTTP en mayúsculas (por ejemplo `GET`, `POST`)
    2.  URI (solo path + query; sin esquema/host), p. ej. `/api/v1/payment?id=123`
    3.  Marca de tiempo Unix (mismo entero que en el encabezado `X-Timestamp`)
    4.  Identificador estable (mismo valor que `X-Client-ID`)
    5.  Hash SHA-256 del cuerpo en base64: `base64(sha256(raw_body))`. Para solicitudes sin cuerpo use `base64(sha256(""))`

2.  **Una los componentes** con saltos de línea `"\n"` para formar la cadena canónica.

3.  **Calcule la firma** con HMAC-SHA256 usando su clave secreta:

    ```php
    $metodo = strtoupper($metodoSolicitud);
    $uri = $rutaConQuery; // path + query, sin esquema/host
    $marcaTiempo = (string) $timestampUnix;
    $idCliente = $xClientId;
    $cuerpoCrudo = $cuerpoCrudoSolicitud; // exactamente como se envía por la red
    $hashCuerpo = base64_encode(hash('sha256', $cuerpoCrudo, true));
    $canonica = implode("\n", [$metodo, $uri, $marcaTiempo, $idCliente, $hashCuerpo]);
    $firma = hash_hmac('sha256', $canonica, $clavePrivada);
    ```

4.  **Envíe y firme cuerpos JSON** sin escapes extra usando `json_encode` con `JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE` para que el cuerpo crudo coincida con lo que verifica el servidor.

**Notas de Seguridad:**
- Las solicitudes son válidas solo dentro de los 15 minutos del `X-Timestamp` para prevenir ataques de repetición.
- Su secreto nunca se transmite por la red.
- El hash del cuerpo garantiza la integridad del payload.

Puede encontrar o regenerar su ID de cliente y clave privada en su configuración de Comercio.

## Endpoints de la API

Todos los endpoints requieren los encabezados de [Autenticación](#autenticación).

### Terminología

- **Enlace de pago** (link payment): El enlace o producto que el cliente paga (ej. “Comprar X”). Identificado por **ID de enlace de pago** (`link_payment_id`). Se usa al crear o consultar enlaces (`/api/v1/payment`) y al filtrar transacciones por enlace.
- **Transacción** (pago): Un intento de pago o pago completado. Identificado por **ID de transacción** (`id`). Se usa en `GET /api/v1/transactions/{id}` y `POST /api/v1/transactions/{id}/reverse`.  
  Use el **ID de transacción** en la ruta de esos endpoints—no el ID del enlace de pago.

### Enlaces de Pago

#### Crear un Enlace de Pago

Crea un nuevo enlace de pago asociado a su cuenta de Comercio.

**Endpoint:** `POST https://arnipay.com.py/api/v1/payment`

**Encabezados:**
- `Content-Type: application/json`
- *Se requieren encabezados de autenticación estándar*

**Parámetros del Cuerpo de la Solicitud:**

| Parámetro | Tipo | Requerido | Descripción |
|-----------|------|----------|-------------|
| `price` | número | Sí | El precio del ítem (mínimo 1) |
| `title` | cadena | Sí | Título del enlace de pago (máx 255 caracteres) |
| `description` | cadena | No | Descripción opcional del enlace de pago |
| `image` | cadena (URL) | No | URL opcional a una imagen para el enlace de pago |
| `payment_methods` | array de cadenas | No | Lista opcional de métodos de pago (si se omite, se permiten todos los métodos) |
| `reference` | cadena | No | Código de referencia opcional (máx 255 caracteres) |
| `start_date` | fecha | No | Fecha opcional en la que el enlace se activa |
| `expiration_date` | fecha | No | Fecha de vencimiento opcional (debe ser posterior a start_date) |
| `approved_redirection_url` | cadena (URL) | No | URL opcional para redirigir después de un pago exitoso |
| `failed_redirection_url` | cadena (URL) | No | URL opcional para redirigir después de un pago fallido |
| `process_redirection_url` | cadena (URL) | No | URL opcional para redirigir durante el procesamiento del pago |

**Ejemplo de Solicitud:**

```json
{
  "price": 150000,
  "title": "Suscripción Premium",
  "description": "Acceso de 1 año a todo el contenido premium",
  "payment_methods": ["qr", "tigo"],
  "reference": "SUB-2025",
  "approved_redirection_url": "https://example.com/success",
  "failed_redirection_url": "https://example.com/failed"
}
```

**Respuesta Exitosa (201 Created):**

```json
{
  "status": "success",
  "message": "Enlace de pago creado exitosamente",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "url": "https://arnipay.com.py/checkout/550e8400-e29b-41d4-a716-446655440000",
    "commerce_id": 123,
    "title": "Suscripción Premium",
    "price": 150000,
    "created_at": "2025-03-10T09:00:00Z"
  }
}
```

**Respuestas de Error:**
- `422 Unprocessable Entity`: Errores de validación en los datos de la solicitud

#### Obtener un Enlace de Pago Específico

Recupera información detallada sobre un enlace de pago específico (por ID de enlace de pago).

**Endpoint:** `GET https://arnipay.com.py/api/v1/payment/{id}`

**Parámetros:**
- `id`: El UUID del enlace de pago

**Respuesta Exitosa (200 OK):**

```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "url": "https://yourdomain.com/checkout/550e8400-e29b-41d4-a716-446655440000",
    "commerce_id": 123,
    "title": "Suscripción Premium",
    "price": 150000,
    "description": "Acceso de 1 año a todo el contenido premium",
    "enabled": true,
    "payment_methods": ["qr", "tigo"],
    "reference": "SUB-2025",
    "stock": null,
    "quantity": null,
    "start_date": null,
    "expiration_date": null,
    "created_at": "2025-03-10T09:00:00Z",
    "updated_at": "2025-03-10T09:00:00Z",
    "is_paid": false,
    "status": null
  }
}
```

`is_paid` es verdadero solo mientras un pago del enlace sigue en `paid`. `status` es el estado del último pago, o `null` si el enlace todavía no tiene un pago. Después de un reembolso, `is_paid` es `false` y `status` es `refunded` o `auto_refunded`. No interprete `is_paid: false` como “nunca se pagó”.

Valores de `status`: `created`, `pending`, `paid`, `failed`, `cancelled`, `refunded`, `auto_refunded`, `expired`, `voided`, `pending_refund`, `pending_void`, `pending_chargeback`, o `null`.

El listado (`GET /api/v1/payment`) también incluye `is_paid` y `status` en cada enlace.

**Respuestas de Error:**
- `404 Not Found`: Enlace de pago no encontrado

#### Obtener Métodos de Pago

Obtiene una lista de métodos de pago disponibles.

**Endpoint:** `GET https://arnipay.com.py/api/v1/payment_methods`

**Respuesta Exitosa (200 OK):**

```json
{
    "status": "success",
    "data": [
        {
            "code": "qr",
            "name": "Código QR"
        },
        {
            "code": "tigo",
            "name": "Tigo"
        },
        {
            "code": "personal",
            "name": "Personal"
        }
    ]
}
```

### Transacciones

#### Listar Transacciones

Obtiene una lista paginada de transacciones (pagos) de su cuenta de Comercio.

**Endpoint:** `GET https://arnipay.com.py/api/v1/transactions`

**Parámetros de consulta (opcionales):**

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `link_payment_id` | entero | Filtrar por ID de enlace de pago |
| `page` | entero | Número de página para paginación (predeterminado: 1) |

**Respuesta Exitosa (200 OK):**

La lista de transacciones está en el array de primer nivel `data`. La información de paginación está en `meta`. No espere una estructura anidada `data.data`.

```json
{
  "status": "success",
  "data": [
    {
      "id": 1,
      "link_payment_id": 5,
      "commerce_id": 1,
      "amount": 10000,
      "status": "paid",
      "payment_method": "tigo",
      "paymentable_type": "tigo",
      "paymentable_id": 123,
      "created_at": "2025-02-03T12:00:00.000000Z",
      "link_payment": {
        "id": 5,
        "title": "Example link"
      }
    }
  ],
  "meta": {
    "current_page": 1,
    "last_page": 1,
    "per_page": 20,
    "total": 1
  }
}
```

**Objeto transacción:** Cada ítem incluye al menos: `id`, `link_payment_id`, `commerce_id`, `amount`, `status`, `payment_method` (ej. `"tigo"`, `"personal"`, `"qr"`), `paymentable_type`, `paymentable_id`, `created_at`, y opcionalmente `link_payment`.  
**Valores de estado:** `created`, `pending`, `paid`, `failed`, `cancelled`, `refunded`, `auto_refunded`, `expired`, `voided`, `pending_refund`, `pending_void`, `pending_chargeback`. Son estados del pago. El nombre del evento de webhook es distinto (`payment.refunded` cubre tanto `refunded` como `auto_refunded`).

#### Obtener una Transacción

Recupera información detallada de una transacción (pago) por ID de transacción.

**Endpoint:** `GET https://arnipay.com.py/api/v1/transactions/{id}`

**Parámetros:**
- `id`: El ID de la transacción (pago) (entero o UUID, según implementación)

**Respuesta Exitosa (200 OK):** Misma estructura que un elemento de la lista anterior, con `link_payment` y `paymentable` completos cuando se incluyan. El campo `payment_method` está siempre presente para clientes de la API.

**Respuestas de Error:**
- `404 Not Found`: Transacción no encontrada o no perteneciente al comercio

#### Revertir una Transacción

Inicia una reversión (reembolso) asíncrona para una transacción completada. Use el **ID de transacción** en la ruta—no el ID del enlace de pago.

**Endpoint:** `POST https://arnipay.com.py/api/v1/transactions/{id}/reverse`

**Parámetros:**
- `id`: El ID de la transacción (pago)—no el ID del enlace de pago

**Encabezados:**
- `Content-Type: application/json`
- *Se requieren encabezados de autenticación estándar*

**Prerrequisitos:**
- Solo se pueden revertir transacciones en estado **paid** (pagado).
- El método de pago debe soportar reversión automática. Algunos métodos (ej. QR) no la soportan; la API devuelve `supports_reversal: false` en ese caso.

**Parámetros del Cuerpo de la Solicitud:**

| Parámetro | Tipo | Requerido | Descripción |
|-----------|------|----------|-------------|
| `reason` | cadena | No | Motivo de la reversión (ej. "Cliente solicitó reembolso"). Por defecto puede ser "Solicitud vía API" o similar. |

**Respuesta Exitosa (200 OK):**

```json
{
  "status": "success",
  "message": "Proceso de reversión iniciado",
  "data": {
    "id": 1,
    "status": "processing_refund"
  }
}
```

**Respuestas de Error:**

- **404 Not Found** – Transacción no encontrada o no perteneciente al comercio:

```json
{
  "status": "error",
  "message": "Payment not found"
}
```

- **400 Bad Request** – El método de pago no soporta reversión automática:

```json
{
  "status": "error",
  "message": "Payment method does not support automatic reversal. Please contact support.",
  "data": {
    "id": 1,
    "payment_method": "qr",
    "supports_reversal": false
  }
}
```

- **400 Bad Request** – La transacción no está en estado pagado:

```json
{
  "status": "error",
  "message": "Payment cannot be reversed. It must be in PAID status.",
  "data": {
    "id": 1,
    "current_status": "failed",
    "required_status": "paid"
  }
}
```

## Manejo de Errores

La API devuelve códigos de estado HTTP estándar para indicar éxito o fracaso:

- `200 OK`: Solicitud exitosa (GET)
- `201 Created`: Recurso creado exitosamente (POST)
- `400 Bad Request`: Estado de solicitud o parámetros inválidos
- `401 Unauthorized`: Autenticación fallida (Credenciales o firma inválidas)
- `404 Not Found`: Recurso solicitado no encontrado
- `422 Unprocessable Entity`: Errores de validación
- `500 Internal Server Error`: Error del lado del servidor

Las respuestas de error incluyen un cuerpo JSON. Los errores de validación usan un objeto `errors`; algunos endpoints (ej. revertir transacción) pueden incluir un objeto `data` con contexto:

```json
{
  "status": "error",
  "message": "Descripción del mensaje de error",
  "errors": {
    "nombre_del_campo": ["Mensaje de error de validación"]
  }
}
```

Algunos errores también devuelven un objeto `data` (ej. `current_status`, `required_status`, `supports_reversal`).

## Paginación

El endpoint **Listar Transacciones** (`GET /api/v1/transactions`) devuelve resultados paginados. La lista está en el array de primer nivel `data`; los metadatos de paginación están en `meta` (`current_page`, `last_page`, `per_page`, `total`). Otros endpoints de lista pueden implementar paginación en el futuro.

## Gestión de Enlaces de Pago

- Los enlaces creados a través de la API tendrán un campo `source` establecido en `"api"`.
- Estos enlaces son completamente funcionales pero no se muestran en la interfaz de usuario para evitar desorden.
- Los pagos para enlaces creados por API serán visibles en el historial de actividad/pago con una insignia de API.

## Notificaciones Webhook

Nuestro sistema puede notificar a su aplicación sobre eventos de pago en tiempo real utilizando notificaciones webhook.

### Eventos de Webhook

| Evento | Descripción |
|-------|-------------|
| `payment.completed` | Un pago se completó. `data.status` es `paid`. |
| `payment.pending` | Un pago se creó o sigue pendiente. `data.status` es `created` o `pending`. |
| `payment.failed` | Un pago falló. `data.status` es `failed`. |
| `payment.cancelled` | Un pago se canceló. `data.status` es `cancelled`. |
| `payment.refund_pending` | Un reembolso empezó y no terminó. `data.status` es `pending_refund`. Incluye una reversión por falta de stock. |
| `payment.refunded` | Los fondos se devolvieron. `data.status` es `refunded` (manual) o `auto_refunded` (sistema). No es `payment.completed`. |
| `payment.expired` | Un pago expiró. `data.status` es `expired`. |
| `payment.voided` | Un pago se anuló. `data.status` es `voided`. |
| `payment.void_pending` | Una anulación empezó y no terminó. `data.status` es `pending_void`. |
| `payment.chargeback_pending` | Un contracargo está pendiente. `data.status` es `pending_chargeback`. |

`payment.completed` es el único evento que significa que el pago está cobrado. Un reembolso es `payment.refunded` o `payment.refund_pending`.

### Formato de Carga Útil del Webhook

Las notificaciones webhook se envían como solicitudes HTTP POST:

```json
{
  "event": "payment.completed",
  "timestamp": "2025-03-10T15:30:45Z",
  "data": {
    "link_id": "550e8400-e29b-41d4-a716-446655440000",
    "payment_id": "12345",
    "status": "paid",
    "payment_method": "qr",
    "amount": 150000,
    "payment_details": {
      "payment_date": "2025-03-10T15:30:40Z"
    }
  }
}
```

### Configuración de Webhook

Para recibir notificaciones webhook, configure su URL y ajustes en su configuración de Comercio:

- **URL de Webhook**: El endpoint en su servidor que recibirá las notificaciones webhook.
- **Secreto de Webhook**: Clave secreta para verificación.
- **Máximo de Intentos**: Intentos de entrega en caso de fallo (predeterminado: 5).

### Seguridad de Webhook

Las solicitudes de webhook se firman usando las mismas reglas que la API. Verifíquelas usando su **Secreto de Webhook**.

**Encabezados:**
- `X-Client-ID`
- `X-Timestamp`
- `X-Signature`
- `X-Webhook-ID`

**Proceso de Verificación:**
1. Lea el cuerpo crudo y calcule `base64(sha256(raw_body))`
2. Construya la cadena canónica (mismas reglas que API).
3. Calcule el HMAC-SHA256 con su **secreto de webhook** y compare con `X-Signature`.

*(Ver sección de [Autenticación](#autenticación) para ejemplos de código)*

### Mecanismo de Reintento de Webhook

Si su servidor responde con un código no 2xx, se reintenta con retroceso exponencial:
- 1 min, 5 min, 15 min, 30 min, 60 min

### Mejores Prácticas para Webhooks

1. Verificar firmas.
2. Procesar de forma idempotente.
3. Responder rápidamente.
4. Usar HTTPS.
5. Registrar errores pero responder con éxito si se recibió.

## Uso en Producción

1. Almacene de forma segura sus credenciales.
2. Implemente manejo de errores adecuado.
3. Considere límites de tasa.
4. Valide el estado de éxito en la respuesta.
