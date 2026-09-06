# Política de Seguridad — Reckly

Este documento define las prácticas de seguridad obligatorias para el proyecto Reckly (PWA de finanzas personales, alojada en GitHub Pages, sin backend propio, sincronizada con Google Sheets vía OAuth 2.0).

No contiene secretos, tokens, credenciales ni valores reales. Cualquier ejemplo de aquí es ilustrativo.

---

## 1. Prácticas obligatorias de seguridad

- **Todo dato dinámico insertado en el DOM debe pasar por una función de escape** (`escapeHtml`) antes de usarse en `innerHTML`. Preferir `textContent` siempre que no se necesite HTML real.
- **Ningún valor de texto proveniente del usuario o de una URL se envía a Google Sheets con `valueInputOption=USER_ENTERED` sin neutralizar previamente** los caracteres `= + - @` al inicio de la cadena (prevención de inyección de fórmulas).
- **Todo parámetro leído de `URLSearchParams` (ej. el flujo `quickadd` del Atajo) se trata como entrada no confiable**: se valida tipo, longitud y contenido antes de usarse en cualquier parte de la UI o de una petición a la API.
- **No se usa `eval()`, `new Function()` ni `document.write()`** en ningún punto del código.
- Antes de cada entrega de una nueva versión, se revisa que no se haya introducido HTML/JS sin escapar en ningún punto que reciba datos del usuario.
- Se mantiene una única fuente de verdad del archivo desplegado (el repositorio de GitHub); nunca se asume que una copia local de trabajo está sincronizada con lo publicado sin verificarlo explícitamente.

## 2. Gestión de secretos

- **Este proyecto no debe contener nunca**: claves privadas, client secrets de OAuth, API keys de servicios de pago, contraseñas, tokens de larga duración, ni archivos de credenciales de service account.
- El único identificador público embebido en el código es el **OAuth Client ID** de Google (tipo "Web application"). Esto es aceptable porque:
  - Los Client ID de aplicaciones web son públicos por diseño en el estándar OAuth 2.0.
  - La protección real la da la lista de **"Orígenes de JavaScript autorizados"** configurada en Google Cloud Console, no la confidencialidad del ID.
- **Nunca se debe agregar un Client Secret** a este proyecto. Si en el futuro se necesita un flujo que lo requiera (ej. Authorization Code flow con backend), eso implica introducir un servidor, y el secreto debe vivir únicamente en variables de entorno de ese servidor — nunca en el código que se sirve al navegador.
- Antes de cada `git push`, revisar manualmente (o con una herramienta como `gitleaks` / `trufflehog`) que no se haya pegado por error ningún valor sensible en el `index.html`.
- Si alguna vez se detecta un secreto expuesto en el historial de git, no basta con borrarlo del archivo actual: hay que revocarlo/regenerarlo en el proveedor (Google Cloud, etc.) y reescribir el historial si es necesario.

## 3. Política de contraseñas

- Reckly **no implementa un sistema de autenticación propio** (usuario/contraseña); toda identidad se delega en el proveedor de OAuth (cuenta de Google).
- Por lo tanto, no se almacenan ni gestionan contraseñas de ningún tipo dentro de este proyecto.
- Si en algún momento se agregara cualquier mecanismo de autenticación propio (ej. un PIN local para abrir la app), debe cumplir como mínimo:
  - No almacenarse en texto plano en ningún lugar (ni `localStorage` ni el código).
  - Usar un mecanismo de hash con sal si se guarda cualquier verificador (ej. `SubtleCrypto` del navegador), nunca comparación de texto plano.

## 4. Autenticación y autorización

- La autenticación se realiza exclusivamente vía **Google Identity Services (OAuth 2.0, flujo de token implícito para SPA)**.
- El **scope solicitado debe ser el mínimo necesario** para la funcionalidad. Actualmente se usa `spreadsheets`; se recomienda migrar a `drive.file` en cuanto sea viable, para limitar el acceso únicamente a los archivos creados por la app.
- Los tokens de acceso:
  - Tienen vida corta (gestionada por Google, típicamente ~1 hora).
  - Se solicitan de nuevo cuando expiran; no se debe implementar ningún mecanismo para extender su vida artificialmente.
  - Se debe ofrecer al usuario una opción visible para **revocar el acceso** desde dentro de la app (llamando a `google.accounts.oauth2.revoke`) y limpiar el almacenamiento local asociado.
- No existe (ni debe existir) un "rol" o "permiso" interno dentro de la app: cada usuario solo controla su propio dispositivo y su propia hoja de cálculo.

## 5. Manejo seguro de archivos

- La app no permite subir archivos arbitrarios; toda persistencia de datos financieros ocurre en `localStorage` (local al dispositivo) y en Google Sheets (vía API oficial).
- Cualquier funcionalidad futura de importar/exportar archivos debe:
  - Validar extensión y tipo MIME antes de procesar.
  - Nunca ejecutar ni interpretar contenido de archivos subidos como código.
  - Limitar el tamaño máximo aceptado.

## 6. Requisitos para dependencias

- Toda librería de terceros cargada desde un CDN (ej. Chart.js) debe:
  - Fijarse a una **versión específica** (nunca `latest`).
  - Verificarse periódicamente contra avisos de seguridad conocidos (CVEs).
  - Incluir el atributo `integrity` (Subresource Integrity) siempre que el CDN lo soporte.
- No se agregan dependencias nuevas sin evaluar antes: origen, mantenimiento activo, y necesidad real (evitar dependencias innecesarias que amplíen la superficie de ataque).

## 7. Protección de APIs

- Todas las llamadas a la API de Google Sheets se hacen directamente desde el navegador del usuario con su propio token — no existe un backend intermedio que las reciba ni las reenvíe.
- Toda entrada de texto que se escriba en una celda de Sheets debe neutralizarse contra inyección de fórmulas (ver sección 1) antes de enviarse.
- Los errores de la API se muestran de forma legible al usuario, pero nunca se deben registrar ni enviar a un servicio externo de logging sin el consentimiento explícito del usuario.

## 8. Reglas para GitHub

- El repositorio que aloja este proyecto en GitHub Pages debe tratarse como **código público**: nunca asumir privacidad de nada que se suba.
- No se sube nunca ningún archivo `.env`, credenciales, ni volcados de datos personales reales (ej. una hoja de cálculo exportada con movimientos reales) al repositorio.
- Antes de cada `git push`, revisar el `diff` para confirmar que no se incluyen datos personales ni secretos.

### 8.1 Configuración obligatoria del repositorio (activar manualmente en GitHub → Settings)

Estos controles no se pueden activar desde el código; se configuran una sola vez en la interfaz web de GitHub:

- **Secret scanning + Push protection**: Settings → Code security → activar "Secret scanning" y "Push protection". En repositorios públicos, GitHub activa el escaneo básico de secretos automáticamente sin costo; "Push protection" (que bloquea el `push` antes de que el secreto llegue al repo) sí requiere activarse a mano.
- **Dependabot alerts**: Settings → Code security → activar "Dependabot alerts" y "Dependabot security updates". Ya existe `.github/dependabot.yml` en este repo listo para cuando haya algo que vigilar.
- **Protección de la rama principal (`main`)**: Settings → Branches → Add branch protection rule sobre `main`:
  - "Require a pull request before merging" (bloquea el push directo a `main`).
  - "Require approvals" con al menos 1 aprobación — solo tiene sentido real si en algún momento colaboras con alguien más; en un repo de un solo mantenedor, esta regla igual sirve como freno para no fusionar cambios propios sin pasar por una revisión consciente vía PR.
  - "Require review from Code Owners" (usa el `.github/CODEOWNERS` ya incluido en este repo).
  - "Do not allow bypassing the above settings" para que ni siquiera el administrador salte la regla sin querer.
- **Permisos de GitHub Actions** (aplica el día que agregues algún workflow): Settings → Actions → General → "Workflow permissions" dejarlo en **"Read repository contents permission"** (solo lectura) en vez de "Read and write", y activar "Require approval for first-time contributors" si el repo llegara a aceptar colaboradores externos.

### 8.2 Reglas para cualquier futuro workflow de GitHub Actions

Aunque **este proyecto no tiene ningún workflow de Actions actualmente** (se verificó explícitamente), si en el futuro se agrega alguno debe cumplir:

- Bloque `permissions:` explícito al inicio del workflow, con el mínimo necesario (ej. `contents: read`); nunca dejar los permisos por defecto ni usar `write-all`.
- Nunca usar el evento `pull_request_target` combinado con checkout del código de un PR externo — es una combinación insegura conocida (ejecuta el workflow con permisos/secretos del repo base sobre código no confiable).
- Toda Action de terceros (`uses: usuario/accion@...`) debe fijarse a un **commit SHA completo de 40 caracteres**, no a una etiqueta (`@v3`) ni a una rama — las etiquetas se pueden mover después de publicadas. El SHA se obtiene desde la página de "Releases" de la Action en GitHub.
- No exponer ningún `secrets.*` a steps que ejecuten código de terceros no auditado.
- No subir artefactos que contengan datos de usuarios reales ni credenciales.

## 9. Reglas para Google Drive / Google Sheets

- La hoja de cálculo creada por la app es propiedad del usuario dentro de su propia cuenta de Google Drive; Reckly no tiene ni debe tener acceso a hojas de otros usuarios.
- No se debe ampliar nunca el scope de OAuth más allá de lo estrictamente necesario para leer/escribir la hoja de la propia app.
- El usuario es responsable de la configuración de compartición de su propia hoja (quién más tiene acceso); la app no gestiona permisos de compartición.

## 10. Logging y monitoreo

- La app **no envía telemetría, analítica ni logs a ningún servidor externo**. Todo el registro de errores ocurre únicamente en la sesión local del navegador del usuario (visible solo para él).
- No se debe agregar en el futuro ningún servicio de analítica de terceros sin: (a) que sea estrictamente necesario, (b) que se documente qué datos recoge, y (c) que se informe al usuario.
- Los mensajes de error mostrados al usuario no deben incluir tokens, IDs de hoja completos, ni ninguna otra credencial — solo el mensaje descriptivo del problema.

## 11. Reporte de vulnerabilidades

- Si se detecta una vulnerabilidad en este proyecto, se debe:
  1. Documentarla en detalle (archivo, línea, cómo reproducirla).
  2. Priorizar su corrección según severidad (Crítico > Alto > Medio > Bajo), atendiendo primero cualquier hallazgo Crítico o Alto.
  3. Aplicar el arreglo y volver a desplegar antes de continuar con nuevas funcionalidades no relacionadas.
- Al ser un proyecto personal sin canal público de reporte, cualquier hallazgo se atiende directamente entre el desarrollador y quien lo detecte.

## 12. Respuesta ante incidentes

Si se sospecha que el token de Google fue comprometido (por ejemplo, tras detectar cambios no reconocidos en la hoja de cálculo):

1. **Revocar inmediatamente el acceso** desde [myaccount.google.com/permissions](https://myaccount.google.com/permissions), buscando la app y quitando su acceso.
2. Revisar el historial de versiones de la hoja de Google Sheets afectada (Archivo > Historial de versiones) para identificar y revertir cambios no autorizados.
3. Si se sospecha que la causa fue un XSS explotado, dejar de usar la app hasta que el hallazgo correspondiente (ver sección de análisis de seguridad del proyecto) esté corregido y verificado.
4. Volver a conectar la app únicamente después de confirmar que la causa raíz fue corregida.
5. Si se sospecha compromiso del propio Client ID (por ejemplo, orígenes autorizados modificados sin tu intervención), revisar inmediatamente la configuración en Google Cloud Console y regenerar credenciales si es necesario.

---

*Última actualización de este documento: al momento de la auditoría de seguridad inicial del proyecto. Debe revisarse y actualizarse cada vez que se implemente una recomendación de la auditoría o se agregue una nueva integración externa.*
