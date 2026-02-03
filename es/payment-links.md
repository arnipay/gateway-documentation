# Documentación de la API de Enlaces de Pago

[English Version](../payment-links.md)

Esta documentación proporciona detalles para utilizar la API de Enlaces de Pago para crear y gestionar enlaces de pago de forma programática.

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

### Crear un Enlace de Pago

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

### Obtener un Enlace de Pago Específico

Recupera información detallada sobre un enlace de pago específico.

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
    "is_paid": false
  }
}
```

**Respuestas de Error:**
- `404 Not Found`: Enlace de pago no encontrado

### Obtener Métodos de Pago

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

### Revertir un Pago

Inicia un proceso de reversión asíncrono para un pago completado. Esta operación reembolsa el monto de la transacción al método de pago original del cliente y actualiza el estado del pago.

**Endpoint:** `POST https://arnipay.com.py/api/v1/payment/{id}/reverse`

**Encabezados:**
- `Content-Type: application/json`
- *Se requieren encabezados de autenticación estándar*

**Prerrequisitos:**
- El pago debe estar en estado `paid` (pagado).
- El método de pago utilizado debe soportar reversión automática (actualmente soportado: Tigo Money, Personal Pay).
- Los pagos con QR no soportan reversión automática vía API y requieren intervención manual.

**Parámetros del Cuerpo de la Solicitud:**

| Parámetro | Tipo | Requerido | Descripción |
|-----------|------|----------|-------------|
| `reason` | cadena | No | Motivo opcional para la reversión (ej. "Cliente solicitó reembolso"). |

**Respuesta Exitosa (200 OK):**

```json
{
  "status": "success",
  "message": "Proceso de reversión iniciado",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "status": "processing_refund"
  }
}
```

**Respuestas de Error:**
- `400 Bad Request`: Si el método de pago no soporta reversión o si el pago no está en un estado válido.
- `404 Not Found`: Pago no encontrado

## Manejo de Errores

La API devuelve códigos de estado HTTP estándar para indicar éxito o fracaso:

- `200 OK`: Solicitud exitosa (GET)
- `201 Created`: Recurso creado exitosamente (POST)
- `400 Bad Request`: Estado de solicitud o parámetros inválidos
- `401 Unauthorized`: Autenticación fallida (Credenciales o firma inválidas)
- `404 Not Found`: Recurso solicitado no encontrado
- `422 Unprocessable Entity`: Errores de validación
- `500 Internal Server Error`: Error del lado del servidor

Las respuestas de error incluyen un cuerpo JSON con detalles:

```json
{
  "status": "error",
  "message": "Descripción del mensaje de error",
  "errors": {
    "nombre_del_campo": ["Mensaje de error de validación"]
  }
}
```

## Paginación

Los endpoints de lista pueden implementar paginación en el futuro. La implementación actual devuelve todos los resultados sin paginación.

## Gestión de Enlaces de Pago

- Los enlaces creados a través de la API tendrán un campo `source` establecido en `"api"`.
- Estos enlaces son completamente funcionales pero no se muestran en la interfaz de usuario para evitar desorden.
- Los pagos para enlaces creados por API serán visibles en el historial de actividad/pago con una insignia de API.

## Estados de Pago

Los pagos pueden tener los siguientes estados:

- `PAID`: Pago completado exitosamente.
- `REFUNDED`: Reembolso manual iniciado por el comercio.
- `AUTO_REFUNDED`: Reembolso iniciado por el sistema (ej. problemas de inventario).
- `PENDING_REFUND`: Reembolso en progreso o requiere atención manual.

## Notificaciones Webhook

Nuestro sistema puede notificar a su aplicación sobre eventos de pago en tiempo real utilizando notificaciones webhook.

### Eventos de Webhook

| Evento | Descripción |
|-------|-------------|
| `payment.completed` | Un pago se ha completado exitosamente |
| `payment.failed` | Un pago ha fallado |
| `payment.pending` | Un pago está pendiente de procesamiento |
| `pending_refund` | Condición de falta de stock detectada, reembolso pendiente |
| `auto_refunded` | Fondos devueltos exitosamente al cliente (iniciado por el sistema) |

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
