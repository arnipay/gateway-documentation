# Ejemplos de Código de la API de Enlaces de Pago

[English Version](../payment-links-examples.md)

Este documento proporciona ejemplos prácticos de cómo interactuar con la API de Enlaces de Pago utilizando diferentes lenguajes de programación.

Los ejemplos llaman a producción (`https://arnipay.com.py`). Para sandbox, use las mismas rutas en `https://sandbox.arnipay.com.py`.

### PHP: Construir firma canónica y enviar solicitud

```php
<?php
$clavePrivada = 'su_clave_privada_aqui';
$clienteId = '00000000-0000-0000-0000-000000000000';

$metodo = 'POST';
$rutaConQuery = '/api/v1/payment';
$timestamp = (string) time();

$cuerpo = [
  'price' => 150000,
  'title' => 'Suscripción Premium',
];

$json = json_encode($cuerpo, JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE);
$hashCuerpo = base64_encode(hash('sha256', $json, true));

$canonica = implode("\n", [strtoupper($metodo), $rutaConQuery, $timestamp, $clienteId, $hashCuerpo]);
$firma = hash_hmac('sha256', $canonica, $clavePrivada);

$encabezados = [
  'X-Client-ID: ' . $clienteId,
  'X-Timestamp: ' . $timestamp,
  'X-Signature: ' . $firma,
  'Content-Type: application/json',
];

$ch = curl_init('https://arnipay.com.py' . $rutaConQuery);
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, $json);
curl_setopt($ch, CURLOPT_HTTPHEADER, $encabezados);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$respuesta = curl_exec($ch);
$estado = curl_getinfo($ch, CURLINFO_HTTP_CODE);
curl_close($ch);

echo $estado . "\n" . $respuesta . "\n";
```

### PHP: Verificar webhook

```php
<?php
$secretoWebhook = 'su_secreto_webhook_aqui';

$cuerpoCrudo = file_get_contents('php://input');
$timestamp = $_SERVER['HTTP_X_TIMESTAMP'] ?? '';
$clientId = $_SERVER['HTTP_X_CLIENT_ID'] ?? '';
$firma = $_SERVER['HTTP_X_SIGNATURE'] ?? '';
$metodo = strtoupper($_SERVER['REQUEST_METHOD'] ?? 'POST');
$uri = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
$query = $_SERVER['QUERY_STRING'] ?? '';
if ($query !== '') {
  $uri .= '?' . $query;
}

$hashCuerpo = base64_encode(hash('sha256', $cuerpoCrudo, true));
$canonica = implode("\n", [$metodo, $uri, $timestamp, $clientId, $hashCuerpo]);
$esperada = hash_hmac('sha256', $canonica, $secretoWebhook);

if (!hash_equals($esperada, $firma)) {
  http_response_code(403);
  exit('Firma inválida');
}

http_response_code(200);
echo 'OK';
```
