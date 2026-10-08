# Payment Links API Code Examples

[Versión en Español](es/payment-links-examples.md)

This document provides practical examples of how to interact with the Payment Links API using different programming languages.

Samples call production (`https://arnipay.com.py`). For sandbox, use the same paths on `https://sandbox.arnipay.com.py`.

### PHP: Build canonical signature and send request

```php
<?php
$privateKey = 'your_private_key_here';
$clientId = '00000000-0000-0000-0000-000000000000';

$method = 'POST';
$pathWithQuery = '/api/v1/payment';
$timestamp = (string) time();

$body = [
  'price' => 150000,
  'title' => 'Premium Subscription',
];

$json = json_encode($body, JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE);
$bodyHash = base64_encode(hash('sha256', $json, true));

$canonical = implode("\n", [strtoupper($method), $pathWithQuery, $timestamp, $clientId, $bodyHash]);
$signature = hash_hmac('sha256', $canonical, $privateKey);

$headers = [
  'X-Client-ID: ' . $clientId,
  'X-Timestamp: ' . $timestamp,
  'X-Signature: ' . $signature,
  'Content-Type: application/json',
];

$ch = curl_init('https://arnipay.com.py' . $pathWithQuery);
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, $json);
curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$response = curl_exec($ch);
$status = curl_getinfo($ch, CURLINFO_HTTP_CODE);
curl_close($ch);

echo $status . "\n" . $response . "\n";
```

### PHP: Verify webhook

```php
<?php
$webhookSecret = 'your_webhook_secret_here';

$rawBody = file_get_contents('php://input');
$timestamp = $_SERVER['HTTP_X_TIMESTAMP'] ?? '';
$clientId = $_SERVER['HTTP_X_CLIENT_ID'] ?? '';
$signature = $_SERVER['HTTP_X_SIGNATURE'] ?? '';
$method = strtoupper($_SERVER['REQUEST_METHOD'] ?? 'POST');
$uri = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
$query = $_SERVER['QUERY_STRING'] ?? '';
if ($query !== '') {
  $uri .= '?' . $query;
}

$bodyHash = base64_encode(hash('sha256', $rawBody, true));
$canonical = implode("\n", [$method, $uri, $timestamp, $clientId, $bodyHash]);
$expected = hash_hmac('sha256', $canonical, $webhookSecret);

if (!hash_equals($expected, $signature)) {
  http_response_code(403);
  exit('Invalid signature');
}

http_response_code(200);
echo 'OK';
```
