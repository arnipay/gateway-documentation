# Payment Links API Documentation

[Versión en Español](es/payment-links.md)

This documentation provides details for using the Payment Links API to create and manage payment links programmatically.

## Environments

| Environment | Base URL |
|-------------|----------|
| Production | `https://arnipay.com.py/api/v1/` |
| Sandbox | `https://sandbox.arnipay.com.py/api/v1/` |

Endpoint examples below use the production host. Sandbox uses the same paths on `https://sandbox.arnipay.com.py`. Checkout links returned by the sandbox API use that host as well (for example `https://sandbox.arnipay.com.py/checkout/{id}`).

## Authentication

All API requests require signature-based authentication using your Commerce credentials. The following headers must be included with **every** request:

| Header Name | Description |
|-------------|-------------|
| `X-Client-ID` | Your Commerce client ID (UUID format) |
| `X-Timestamp` | Current Unix timestamp as an integer (seconds since January 1, 1970 UTC) |
| `X-Signature` | HMAC-SHA256 signature for request verification |

Requests expire 15 minutes from the time indicated by `X-Timestamp`.

### Canonical String and Signature Generation

Used for both API requests and verifying webhooks.

1.  **Build the canonical components:**
    1.  Uppercased HTTP method (e.g. `GET`, `POST`)
    2.  URI (path + query only; no scheme/host), e.g. `/api/v1/payment?id=123`
    3.  Unix timestamp (same integer value as in the `X-Timestamp` header)
    4.  Stable identifier (same value as `X-Client-ID`)
    5.  Base64-encoded SHA-256 hash of the raw request body: `base64(sha256(raw_body))`. For requests without a body use `base64(sha256(""))`.

2.  **Join the components** with newline characters `"\n"` to form the canonical string.

3.  **Compute the signature** with HMAC-SHA256 using your secret key:

    ```php
    $method = strtoupper($requestMethod);
    $uri = $requestPathWithQuery; // path + query, no scheme/host
    $timestamp = (string) $unixTimestamp;
    $clientId = $xClientId;
    $rawBody = $requestRawBody; // exactly as sent over the wire
    $bodyHash = base64_encode(hash('sha256', $rawBody, true));
    $canonical = implode("\n", [$method, $uri, $timestamp, $clientId, $bodyHash]);
    $signature = hash_hmac('sha256', $canonical, $privateKey);
    ```

4.  **Send JSON bodies** without extra escaping using `json_encode` with `JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE` so the raw body matches what the server verifies.

**Security Notes:**
- Requests are valid only within 15 minutes of the `X-Timestamp` to prevent replay attacks.
- Your secret is never transmitted over the network.
- The body hash ensures integrity of the payload.

You can find or regenerate your client ID and private key in your Commerce settings.

## API Endpoints

All endpoints require the [Authentication](#authentication) headers.

### Terminology

- **Payment link** (link payment): The link or product the customer pays (e.g. “Buy X”). Identified by **payment link ID** (`link_payment_id`). Used when creating or fetching payment links (`/api/v1/payment`) and when filtering transactions by link.
- **Transaction** (payment): A single payment attempt or completed payment. Identified by **transaction ID** (`id`). Used in `GET /api/v1/transactions/{id}` and `POST /api/v1/transactions/{id}/reverse`.  
  Use the **transaction ID** in the path for those endpoints—not the payment link ID.

### Payment Links

#### Create a Payment Link

Creates a new payment link associated with your Commerce account.

**Endpoint:** `POST https://arnipay.com.py/api/v1/payment`

**Headers:**
- `Content-Type: application/json`
- *Standard authentication headers required*

**Request Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `price` | number | Yes | The price of the item (minimum 1) |
| `title` | string | Yes | Title of the payment link (max 255 chars) |
| `description` | string | No | Optional description of the payment link |
| `image` | string (URL) | No | Optional URL to an image for the payment link |
| `payment_methods` | array of strings | No | Optional list of payment methods (if omitted, all methods are allowed) |
| `reference` | string | No | Optional reference code (max 255 chars) |
| `start_date` | date | No | Optional date when the link becomes active |
| `expiration_date` | date | No | Optional expiration date (must be after start_date) |
| `approved_redirection_url` | string (URL) | No | Optional URL to redirect after successful payment |
| `failed_redirection_url` | string (URL) | No | Optional URL to redirect after failed payment |
| `process_redirection_url` | string (URL) | No | Optional URL to redirect during payment processing |

**Example Request:**

```json
{
  "price": 150000,
  "title": "Premium Subscription",
  "description": "1 year access to all premium content",
  "payment_methods": ["qr", "tigo"],
  "reference": "SUB-2025",
  "approved_redirection_url": "https://example.com/success",
  "failed_redirection_url": "https://example.com/failed"
}
```

**Success Response (201 Created):**

```json
{
  "status": "success",
  "message": "Payment link created successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "url": "https://arnipay.com.py/checkout/550e8400-e29b-41d4-a716-446655440000",
    "commerce_id": 123,
    "title": "Premium Subscription",
    "price": 150000,
    "created_at": "2025-03-10T09:00:00Z"
  }
}
```

**Error Responses:**
- `422 Unprocessable Entity`: Validation errors in the request data

#### Get a Specific Payment Link

Retrieves detailed information about a specific payment link (by payment link ID).

**Endpoint:** `GET https://arnipay.com.py/api/v1/payment/{id}`

**Parameters:**
- `id`: The UUID of the payment link

**Success Response (200 OK):**

```json
{
  "status": "success",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "url": "https://yourdomain.com/checkout/550e8400-e29b-41d4-a716-446655440000",
    "commerce_id": 123,
    "title": "Premium Subscription",
    "price": 150000,
    "description": "1 year access to all premium content",
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

`is_paid` is true only while a payment on the link is still `paid`. `status` is the latest payment status, or `null` when the link has no payment yet. After a refund, `is_paid` is `false` and `status` is `refunded` or `auto_refunded`. Do not treat `is_paid: false` as “never paid”.

`status` values: `created`, `pending`, `paid`, `failed`, `cancelled`, `refunded`, `auto_refunded`, `expired`, `voided`, `pending_refund`, `pending_void`, `pending_chargeback`, or `null`.

The list endpoint (`GET /api/v1/payment`) includes `is_paid` and `status` on each link as well.

**Error Responses:**
- `404 Not Found`: Payment link not found

#### Get Payment Methods

Retrieves a list of available payment methods.

**Endpoint:** `GET https://arnipay.com.py/api/v1/payment_methods`

**Success Response (200 OK):**

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

### Transactions

#### List Transactions

Retrieves a paginated list of transactions (payments) for your Commerce account.

**Endpoint:** `GET https://arnipay.com.py/api/v1/transactions`

**Query Parameters (optional):**

| Parameter | Type | Description |
|-----------|------|-------------|
| `link_payment_id` | integer | Filter by payment link ID |
| `page` | integer | Page number for pagination (default: 1) |

**Success Response (200 OK):**

The list of transactions is in the top-level `data` array. Pagination info is in `meta`. Do not expect a nested `data.data` structure.

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

**Transaction object:** Each item includes at least: `id`, `link_payment_id`, `commerce_id`, `amount`, `status`, `payment_method` (e.g. `"tigo"`, `"personal"`, `"qr"`), `paymentable_type`, `paymentable_id`, `created_at`, and optionally `link_payment`.  
**Status values:** `created`, `pending`, `paid`, `failed`, `cancelled`, `refunded`, `auto_refunded`, `expired`, `voided`, `pending_refund`, `pending_void`, `pending_chargeback`. These are payment statuses. Webhook `event` names are separate (`payment.refunded` covers both `refunded` and `auto_refunded`).

#### Get a Single Transaction

Retrieves detailed information about a specific transaction (payment) by transaction ID.

**Endpoint:** `GET https://arnipay.com.py/api/v1/transactions/{id}`

**Parameters:**
- `id`: The transaction (payment) ID (integer or UUID, depending on implementation)

**Success Response (200 OK):** Same structure as one element of the list above, with full `link_payment` and `paymentable` when included. The `payment_method` field is always present for API clients.

**Error Responses:**
- `404 Not Found`: Transaction not found or not belonging to the commerce

#### Reverse a Transaction

Initiates an asynchronous reversal (refund) for a completed transaction. This operation refunds the amount to the customer's original payment method. Use the **transaction ID** in the path—not the payment link ID.

**Endpoint:** `POST https://arnipay.com.py/api/v1/transactions/{id}/reverse`

**Parameters:**
- `id`: The transaction (payment) ID—not the payment link ID

**Headers:**
- `Content-Type: application/json`
- *Standard authentication headers required*

**Prerequisites:**
- Only transactions in **paid** status can be reversed.
- The payment method must support automatic reversal. Some methods (e.g. QR) do not support it; the API returns `supports_reversal: false` in that case.

**Request Body Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `reason` | string | No | Reason for the reversal (e.g. "Customer requested refund"). Default may be "Solicitud vía API" or similar. |

**Success Response (200 OK):**

```json
{
  "status": "success",
  "message": "Reversal process initiated",
  "data": {
    "id": 1,
    "status": "processing_refund"
  }
}
```

**Error Responses:**

- **404 Not Found** – Transaction not found or not belonging to the commerce:

```json
{
  "status": "error",
  "message": "Payment not found"
}
```

- **400 Bad Request** – Payment method does not support automatic reversal:

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

- **400 Bad Request** – Transaction is not in paid status:

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

## Error Handling

The API returns standard HTTP status codes to indicate success or failure:

- `200 OK`: Request successful (for GET requests)
- `201 Created`: Resource created successfully (for POST requests)
- `400 Bad Request`: Invalid request state or parameters
- `401 Unauthorized`: Authentication failed (Invalid headers or signature)
- `404 Not Found`: Requested resource not found
- `422 Unprocessable Entity`: Validation errors
- `500 Internal Server Error`: Server-side error

Error responses include a JSON body. Validation errors use an `errors` object; some endpoints (e.g. reverse transaction) may include a `data` object with context:

```json
{
  "status": "error",
  "message": "Error message description",
  "errors": {
    "field_name": ["Validation error message"]
  }
}
```

Some errors also return a `data` object (e.g. `current_status`, `required_status`, `supports_reversal`).

## Pagination

The **List Transactions** endpoint (`GET /api/v1/transactions`) returns paginated results. The list is in the top-level `data` array; pagination metadata is in `meta` (`current_page`, `last_page`, `per_page`, `total`). Other list endpoints may implement pagination in the future.

## Payment Link Management

- Links created via the API will have a `source` field set to `"api"`.
- These links are fully functional but are not displayed in the UI interface to prevent cluttering.
- Payments for API-created links will be visible in the activity/payment history with an API badge.

## Webhook Notifications

Our system can notify your application about payment events in real-time using webhook notifications. This allows your application to receive updates about payments without polling the API.

### Webhook Events

The following events trigger webhook notifications:

| Event | Description |
|-------|-------------|
| `payment.completed` | A payment has been successfully completed. `data.status` is `paid`. |
| `payment.pending` | A payment was created or is still pending. `data.status` is `created` or `pending`. |
| `payment.failed` | A payment has failed. `data.status` is `failed`. |
| `payment.cancelled` | A payment was cancelled. `data.status` is `cancelled`. |
| `payment.refund_pending` | A refund has started and has not finished. `data.status` is `pending_refund`. This includes an out-of-stock reversal. |
| `payment.refunded` | Funds were returned. `data.status` is `refunded` (manual) or `auto_refunded` (system). This is not `payment.completed`. |
| `payment.expired` | A payment expired. `data.status` is `expired`. |
| `payment.voided` | A payment was voided. `data.status` is `voided`. |
| `payment.void_pending` | A void has started and has not finished. `data.status` is `pending_void`. |
| `payment.chargeback_pending` | A chargeback is pending. `data.status` is `pending_chargeback`. |

`payment.completed` is the only event that means the payment is collected. A refund is `payment.refunded` or `payment.refund_pending`, even though an older gateway build sent those updates as `payment.completed`.

### Webhook Payload Format

Webhook notifications are sent as HTTP POST requests to your configured webhook URL with a JSON payload:

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

### Webhook Configuration

To receive webhook notifications, configure your webhook URL and settings in your Commerce settings:

- **Webhook URL**: The endpoint on your server that will receive webhook notifications.
- **Webhook Secret**: A secret key used to verify that webhook notifications are coming from our system.
- **Webhook Max Attempts**: The number of times the system will attempt to deliver a webhook in case of failure (default: 5, max: 10).

### Webhook Security

Webhook requests are signed using the same canonical string rules as API requests. You should verify them using your **Webhook Secret**.

**Headers:**
- `X-Client-ID`
- `X-Timestamp`
- `X-Signature`
- `X-Webhook-ID` (unique identifier for the webhook delivery)

**Verification Process:**

1. Read the raw body exactly as received and compute `base64(sha256(raw_body))`
2. Build the canonical string using: uppercased HTTP method, request URI (path + query), `X-Timestamp`, `X-Client-ID`, and the base64 body hash
3. Compute the HMAC-SHA256 with your **webhook secret** and compare to `X-Signature`

*(Refer to the [Authentication](#authentication) section for code logic examples)*

### Webhook Retry Mechanism

If your server responds with a non-2xx status code, we will retry sending the webhook notification using an exponential backoff strategy:

- 1st retry: 1 minute
- 2nd retry: 5 minutes
- 3rd retry: 15 minutes
- 4th retry: 30 minutes
- 5th retry: 60 minutes

### Webhook Best Practices

1. **Verify Signatures**: Always verify webhook signatures to ensure authenticity.
2. **Process Idempotently**: Design your webhook handler to be idempotent.
3. **Respond Quickly**: Respond within a few seconds to avoid timeouts.
4. **Use HTTPS**: Always use HTTPS for your webhook URL.
5. **Implement Error Handling**: Log errors but respond with a successful status code if you've received the webhook.

## Using in Production

For production use:
1. Securely store your client_id and private_key
2. Implement proper error handling
3. Consider implementing rate limiting on your side to prevent overloading the API
4. Always validate the success status in the response
