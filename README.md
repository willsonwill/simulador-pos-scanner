# Simulador POS · Escáner móvil

Página única (`index.html`, sin dependencias de build) que simula la pantalla
de "Nueva venta" de un sistema POS, con soporte para recibir lecturas de
código de barras desde el celular vía WebSocket, como si fuera un lector
inalámbrico.

## Cómo ejecutarlo

No requiere instalación ni servidor. Basta con abrir el archivo en el
navegador:

```bash
xdg-open index.html
```

o simplemente hacer doble clic sobre `index.html`.

## Qué hace

- **Catálogo simulado**: una lista fija de productos (ver `BASE` en el
  script) con código de barras EAN-13, precio, unidad de medida y stock.
- **Búsqueda manual**: escribe o pega un código en el campo de búsqueda y
  presiona `Enter` o `F9` para agregarlo a la venta.
- **Códigos de prueba**: el panel "Códigos de prueba" dibuja el código de
  barras (SVG) de cada producto del catálogo, más uno que no existe (para
  probar el caso de error). Se pueden:
  - ampliar en una ventana modal para escanearlos con la cámara del celular,
  - o simular su lectura con un clic, sin necesidad de un escáner físico.
- **Escáner remoto por WebSocket**: la página puede conectarse como cliente a
  un servidor WebSocket que corre en el celular (una app de escaneo), recibir
  los códigos leídos y agregarlos automáticamente a la venta.
- **Registro de mensajes**: panel con el historial de mensajes enviados y
  recibidos por el socket, útil para depurar la conexión.

## Conectar un escáner (celular)

1. En el celular, corre una app/servidor que abra un WebSocket en
   `ws://<IP_DEL_CELULAR>:<PUERTO>` (por defecto se usa el puerto `8765`) y
   que muestre un PIN de 6 dígitos.
2. En la página, escribe la IP del celular, el puerto y el PIN, y presiona
   **Conectar**.
3. Cada código que la app lea se envía a la página y se agrega como línea de
   venta.

La configuración (IP, puerto, PIN) se guarda en `localStorage` para no tener
que volver a escribirla.

### Protocolo de mensajes

Los mensajes son JSON con un campo `tipo`. `→` = enviado por la página,
`←` = recibido del celular.

| Dirección | tipo         | Campos                                   | Descripción                                   |
|-----------|--------------|-------------------------------------------|------------------------------------------------|
| →         | `auth`       | `pin`                                      | Se envía al abrir el socket, para autenticarse. |
| ←         | `auth_ok`    | `dispositivo?`                             | PIN correcto, la conexión queda activa.         |
| ←         | `auth_error` | `mensaje?`                                 | PIN incorrecto, la conexión se cierra.          |
| ←         | `desplazado` | `mensaje?`                                 | Otra PC tomó el control del escáner.            |
| ←         | `codigo`     | `id`, `valor`, `cantidad?`, `formato?`      | Lectura de un código de barras.                 |
| →         | `resultado`  | `id`, `encontrado`, `producto?`, `precio?`, `mensaje?` | Respuesta a un mensaje `codigo`.   |
| →         | `ping` / ← `pong` | —                                      | Keep-alive cada 10 s; si no hay respuesta en 25 s, se reconecta. |

Si la conexión se pierde de forma inesperada, la página reintenta
automáticamente con backoff exponencial (1 s, 2 s, 4 s... hasta un máximo de
15 s entre intentos).

## Estructura del código

Todo vive en `index.html`, dividido en bloques comentados:

- **Utilidades**: helpers genéricos (`$`, `esc`, `bs`, `hora`, `checkDigit`).
- **Catálogo simulado**: los productos de prueba y sus códigos EAN-13.
- **EAN-13 en SVG**: generación de los códigos de barras dibujados en
  pantalla para poder "escanearlos" desde la cámara del celular.
- **Venta**: estado de las líneas de venta, búsqueda de productos y render
  de la tabla/totales.
- **Registro**: panel de log de mensajes.
- **Cliente WebSocket**: conexión, autenticación, reconexión y manejo de los
  mensajes descritos arriba.
- **Inicio**: carga de configuración guardada y arranque de la página.

## Notas

- Los códigos de barras usan el dígito verificador EAN-13 real, calculado
  automáticamente a partir de los 12 dígitos base de cada producto.
- Es un simulador pensado para pruebas/demo: el catálogo y el "stock" son
  datos fijos en el propio archivo, no hay persistencia de ventas.
