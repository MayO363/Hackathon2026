# Nicaragua Creativa — Mapa de la Red Nacional de Ciudades Creativas

El objetivo de esta aplicacion es ayudar a facilitar optimizar y automatizar uno de los mas grandes
problemas que tienen los turistas hoy en dia en cual es no saber en donde estan, la mayoria no puede contar con un guia turistico es por eso que tras tus huellas llega al rescate, nuestra aplicacion incluye un mapa interactivo donde el usuario podra ubicr distintas zonas comerciales famosas y concurridas del pais asi podra tener variedad a la hora de elegir donde explorar, ademas agrega un sistema de afiliacion entre local y usuario para que puedan seguir en contacto y un sistema de reseñas a los locales afiliados a nuestra aplicacion. 

# Cómo usarla

1. Descomprime la carpeta `ciudades-creativas`.
2. Abre `index.html` con doble clic en cualquier navegador (Chrome, Firefox,
   Safari, Edge) — funciona igual en computadora y en celular.
3. Para publicarla en internet, súbela tal cual a cualquier hosting estático:
   GitHub Pages, Netlify, Vercel o el hosting que prefieras. No requiere
   servidor, base de datos ni claves de API.

## Estructura del proyecto

```
ciudades-creativas/
├── index.html        página principal
├── css/style.css      estilos (paleta, tipografía, responsive)
├── js/data.js         las 10 ciudades y lugares de ejemplo
└── js/app.js          lógica del mapa, panel, formularios y reseñas
```

## Funcionalidad incluida

- Mapa interactivo (Leaflet + OpenStreetMap, gratuito, sin API key) con las
  10 ciudades creativas marcadas.
- Selector de ciudades (chips) que centra el mapa y filtra los lugares.
- Buscador de lugares por nombre.
- Botón **"Agregar lugar"**: activa un modo en el que, al tocar el mapa, se
  abre un formulario para registrar el nombre, categoría, ciudad y
  descripción del lugar. El nuevo punto aparece de inmediato en el mapa.
- Reseñas por lugar: calificación de 1 a 5 estrellas, nombre y comentario.
  El promedio de estrellas se ve tanto en el panel como en el globo del
  mapa (popup).
- Panel lateral en computadora / hoja deslizable en móvil, con vista de
  ciudad, listado de lugares y detalle de cada lugar.

## Sobre los datos y la persistencia

Los lugares y reseñas que agregues quedan guardados **solo en la memoria
del navegador durante esa sesión** (se reinician al recargar la página).
Esto fue así a propósito para que el archivo funcione en cualquier
navegador sin depender de un servidor. Si quieres que los datos se
mantengan entre visitas, hay dos caminos sencillos a partir de este mismo
código:

- **`localStorage`** (persistencia solo en ese navegador/dispositivo):
  en `js/app.js`, guarda `state.places` en `localStorage` cada vez que
  cambie, y léelo al iniciar `initData()`.
- **Un backend real** (para que las reseñas se compartan entre todos los
  usuarios): reemplaza las funciones `guardarLugar()` y `guardarResena()`
  en `js/app.js` por llamadas `fetch()` a tu propia API (por ejemplo con
  Firebase, Supabase o un backend propio).

## Personalización

- **Colores y tipografía:** variables al inicio de `css/style.css`
  (`:root { --primary, --accent, ... }`).
- **Ciudades y lugares:** editables directamente en `js/data.js`.
- **Categorías creativas:** objeto `CATEGORIAS` en `js/data.js` (color e
  ícono de cada tipo de lugar).
