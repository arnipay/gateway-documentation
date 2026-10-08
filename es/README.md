# Documentación de la API

[English Version](../README.md)

Bienvenido a la documentación de la API para nuestro sistema de procesamiento de pagos.

## Documentación Disponible

- [Documentación de la API de Enlaces de Pago](../payment-links.md) - Documentación completa de los endpoints de la API de Enlaces de Pago, autenticación, formatos de solicitud/respuesta y mejores prácticas.
- [Ejemplos de Código de Enlaces de Pago](../payment-links-examples.md) - Ejemplos prácticos de código en múltiples lenguajes de programación para integrar con la API de Enlaces de Pago.

## Autenticación

Todos los endpoints de la API requieren autenticación utilizando sus credenciales de Comercio:

- `X-Client-ID`: Su ID de cliente de Comercio (formato UUID)
- `X-Private-Key`: Su clave privada de Comercio

Puede encontrar estas credenciales en su página de configuración de Comercio, o regenerarlas si es necesario.

## URL Base de la API

| Entorno | URL base |
|---------|----------|
| Producción | `https://arnipay.com.py/api/v1/` |
| Sandbox | `https://sandbox.arnipay.com.py/api/v1/` |

Use credenciales de sandbox contra el host de sandbox. Las rutas son las mismas en ambos entornos. Los ejemplos de esta documentación usan el host de producción.

## Webhooks

Nuestro sistema de pago puede enviar notificaciones en tiempo real a su servidor utilizando webhooks cuando ocurren eventos de pago. Configure su URL de webhook y la configuración en la página de configuración de Comercio para recibir estas notificaciones.

Características clave de los webhooks:
- Notificaciones de pago en tiempo real (completado, fallido, pendiente)
- Entrega segura con verificación de firma HMAC-SHA256
- Mecanismo de reintento configurable
- Cargas detalladas de eventos

Para información detallada sobre webhooks y ejemplos de implementación, vea:
- [Documentación de Configuración de Webhook](../payment-links.md#webhook-notifications)
- [Ejemplos de Implementación de Webhook](../payment-links-examples.md#webhook-examples)

## Límites de Tasa

Nuestra API implementa límites de tasa para proteger el servicio. Los límites actuales son:

- 60 solicitudes por minuto por cliente
- 1000 solicitudes por día por cliente

Si excede estos límites, recibirá una respuesta `429 Demasiadas Solicitudes`.

## Soporte

Si tiene alguna pregunta o necesita asistencia para integrarse con nuestra API, por favor contacte a nuestro equipo de soporte en info@arnipay.com.py.

## SDK

Proporcionamos SDKs oficiales para ayudarle a integrarse con nuestro sistema de pago de manera rápida y fácil en su lenguaje de programación preferido:

Cada SDK incluye documentación completa, ejemplos y manejo de errores para agilizar su proceso de integración.

### SDK de JavaScript

Una biblioteca JavaScript/TypeScript con soporte completo para TypeScript tanto para entornos de navegador como de Node.js.

Nuestro SDK de JavaScript está disponible a través de los siguientes canales:

- **GitHub**: [github.com/arnipay/sdk-js](https://github.com/arnipay/sdk-js)
- **npm**: Instalar vía npm:
  ```bash
  npm install gw-sdk
  ```

### SDK de PHP

Una biblioteca PHP que proporciona interfaces simples para crear enlaces de pago, obtener información de pago y manejar webhooks.

Nuestro SDK de PHP está disponible a través de los siguientes canales:

- **GitHub**: [github.com/arnipay/sdk-php](https://github.com/arnipay/sdk-php)
- **Composer**: Instalar vía Composer:
  ```bash
  composer require arnipay/sdk-php
  ``` 
