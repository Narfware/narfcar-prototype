# Taller Demo

Prototipo navegable para entrevistas con talleres de mecánica general. No es el producto: sirve para observar cómo reacciona un mecánico al registrar un coche desde el móvil.

## Qué hace
- **Tablero** con los coches en recepción, esperando OK, en reparación y listos.
- **Entrada de coche**: matrícula (con validación del formato español y detección de cliente habitual), km, depósito, motivo, fotos de estado.
- **Orden de trabajo**: plantillas de trabajos frecuentes, piezas, horas, fotos de piezas y dictado por voz con sugerencias.
- **Presupuesto** tal como lo vería el cliente y envío por WhatsApp (enlace `wa.me`).
- **Trabajo extra** con aprobación del cliente registrada.
- **Entrega**: borrador de factura, garantía (3 meses / 2.000 km) y recordatorio de próxima revisión.
- **Historial** con la hora de cada dato.

Los datos se guardan solo en el navegador del móvil (`localStorage`). No hay servidor.

## Publicar en GitHub Pages
1. Sube estos archivos a la rama `main` de un repo público.
2. En el repo: **Settings → Pages → Build and deployment → Source: Deploy from a branch → Branch: `main` / `(root)`** → Save.
3. En 1–2 minutos estará en `https://<tu-usuario>.github.io/<repo>/`.

## Consejos para la demo
- Ábrela en **Chrome en Android** para que funcione el dictado por voz (en iPhone, Safari también lo admite en versiones recientes; si no, usa el micrófono del teclado).
- Desde el menú del navegador, **"Añadir a pantalla de inicio"** para abrirla como una app.
- En **Ajustes** puedes poner el nombre del taller, el precio hora y reiniciar los datos de ejemplo.
