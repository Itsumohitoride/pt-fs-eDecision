# 5. Seguridad

La tarjeta se ingresa en la página del proveedor de pago y no pasa por nuestros servidores, así que el riesgo está en órdenes, stock, cuentas y el portal admin. Uso el OWASP Top 10 2025 y me quedo con los seis más relevantes para esta solución.

1. A01:2025 Broken Access Control. El cliente siempre sale del token y no del body: si `customerId` no coincide, 403. Cada endpoint exige su rol de Cognito (cliente, tienda, admin) y una tienda solo ve sus propias órdenes y productos.
2. A02:2025 Security Misconfiguration. El error handler responde 500 genérico y el stack trace va solo al log. CORS permite únicamente nuestros dominios, la base no tiene acceso público y los secretos (base, proveedor, firma de webhooks) están en Secrets Manager, no en variables del repositorio.
3. A03:2025 Software Supply Chain Failures. `package-lock.json` versionado, `npm ci` en el build y `npm audit` en CI que corta el pipeline con vulnerabilidades altas.
4. A05:2025 Injection. Todo el SQL va parametrizado (`$1`, `$2`) y los SKU nunca se concatenan en una consulta. El body se valida antes de tocar la base.
5. A07:2025 Authentication Failures. Login, contraseñas y bloqueo por intentos los maneja Cognito. MFA obligatorio para admin y la API valida firma, emisor, audiencia y expiración del JWT en cada request.
6. A08:2025 Software or Data Integrity Failures. El webhook de pago solo se procesa si la firma es válida; si no, 400 y no cambia nada.
