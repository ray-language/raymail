# raymail

Cliente SMTP **de correo real** + servidor "sink" de desarrollo, escrito en [raylang](https://github.com/ray-language/raylang). Es el estreno de `tls_upgrade` (STARTTLS es su caso canónico y ninguna app lo usaba): EHLO → STARTTLS → el mismo socket envuelto en TLS → EHLO de nuevo → AUTH → MAIL/RCPT/DATA con MIME completo (cabeceras plegadas RFC 5322, asuntos no-ASCII RFC 2047, multipart con adjuntos base64, dot-stuffing).

```text
# Enviar de verdad (STARTTLS + auth)
$ raymail send --to eva@example.com --from yo@midominio.com \
    --subject "informe ñoño" --body-file informe.txt --attach datos.pdf \
    --host smtp.midominio.com --port 587 --starttls --user yo --password …

# Desarrollo: un mailhog-lite que guarda .eml
$ raymail sink --dir ./inbox &
raymail sink on port 2525, storing into ./inbox
$ raymail send --to test@x --from dev@x --subject hola --body "prueba"
sent: OK stored as 1787607333772-38c5e225.eml
```

**Verificado contra infraestructura real**: el camino completo
EHLO → STARTTLS → `tls_upgrade` → EHLO → MAIL contra `smtp.gmail.com:587`
llega hasta el `530 Authentication Required` de Gmail — respondido A TRAVÉS
de la sesión TLS. El upgrade in-place del handle funciona.

## Qué implementa

- **SMTP cliente**: respuestas multilínea (`250-…`), STARTTLS opcional (falla
  honesto si el servidor no lo anuncia), AUTH PLAIN (o LOGIN como fallback),
  dot-stuffing en DATA, QUIT limpio, timeouts.
- **MIME**: plegado de cabeceras a 78 columnas, `=?UTF-8?B?…?=` para asuntos
  con ñ/emoji (partidos en varias encoded-words de ≤75 caracteres, cortando en
  frontera de carácter UTF-8), `Date` RFC 1123, Message-ID aleatorio,
  multipart/mixed con adjuntos en base64 a 76 columnas. Las codificaciones
  vienen de `net/mail` (el paquete `net`), que nació de las versiones a mano
  de esta app.
- **Sink**: acepta EHLO/MAIL/RCPT/DATA/RSET/QUIT, des-dot-stuffea, guarda
  cada mensaje como `.eml` con cabeceras `X-Sink-From/To`, y rechaza STARTTLS
  con un 454 honesto (servir TLS necesita certificado → v2 con `tls_accept`).

## Estado actual

| Capacidad | Estado |
|-----------|--------|
| SMTP + STARTTLS (`tls_upgrade`, probado contra Gmail) + AUTH PLAIN/LOGIN | ✅ |
| MIME: folding, RFC 2047, multipart, base64, dot-stuffing | ✅ |
| Sink de desarrollo → archivos .eml | ✅ |
| Binario nativo | ✅ |
| Tests (MIME unit + E2E cliente→sink por SMTP real) | ✅ 6 |
| Sink con TLS (`tls_accept` + certificado) y UI web | 📋 v2 |
| quoted-printable | 📋 v2 (base64 cubre v1) |

## Hallazgos de dogfood

Anotados en `raylang/IDEAS.md` §71:

1. **`tls_upgrade` funciona a la primera en su estreno** (positivo y citable):
   el envoltorio TLS in-place del handle contra Gmail real, sin sorpresas.
2. **[RESUELTO — `net/mail`, raylang M131]** Las codificaciones de correo se
   escribían a mano (RFC 2047, plegado, dot-stuffing, base64 a 76 columnas).
   raymail usa ahora `net/mail`; de paso se corrige un bug real de la versión
   a mano: un asunto no-ASCII largo salía como UNA encoded-word de más de 75
   caracteres (RFC 2047 §2) en una línea de más de 78.
3. **[RESUELTO — raylang M118]** `'\0'` no era expresable como literal: el NUL
   de AUTH PLAIN es ahora el escape `"\0"`.

## Desarrollo

Requiere raylang 1.27.13+. La única dependencia es `net = "^0.3.7"` (registro de
paquetes, fijada en `ray.lock`), por `net/mail`; `ray test` la descarga sola.

```sh
ray test        # 6 tests
ray build --native src/main.ray -o raymail --release
```

Estructura: `src/main.ray` (CLI) · `mime.ray` (mensaje) · `smtp.ray`
(cliente) · `sink.ray` (servidor dev).
