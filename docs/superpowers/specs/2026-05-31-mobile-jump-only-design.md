# Móvil: solo saltar (tap) — Diseño

**Fecha:** 2026-05-31
**Estado:** Aprobado, listo para plan de implementación
**Archivo afectado:** `index.html` (juego de un solo archivo existente)

## Objetivo

Hacer el juego cómodo de jugar en celular: un único control (saltar) activado
con **tap en cualquier parte** de la pantalla, **solo cactus** como obstáculo, y
el canvas **escalado al ancho** de la pantalla. Sigue funcionando en escritorio
con teclado.

## Decisiones (del brainstorming)

- **Obstáculos:** solo cactus. Se eliminan todos los pájaros (al quitar el
  agacharse, el pájaro de altura de cuerpo ya no sería esquivable).
- **Control móvil:** tap en cualquier parte de la pantalla = saltar / empezar /
  reiniciar.
- **Ajuste de pantalla:** escalar el canvas al ancho disponible, manteniendo la
  relación 3:1 y la resolución interna 660×220. Funciona en cualquier
  orientación, sin pedir girar el teléfono.
- Se conserva el control por teclado (Espacio / ↑) para escritorio.

## Cambios

### 1. Quitar el agacharse
- Eliminar el sprite `HERO_DUCK`.
- Eliminar el campo `game.ducking` (y su reinicio en `resetGame`).
- En `playerSprite()`, quitar la rama de agacharse; siempre devuelve los cuadros
  de correr (`HERO_RUN1` / `HERO_RUN2`).
- Eliminar los listeners de la tecla `↓` (`keydown`/`keyup` de `ArrowDown`).

### 2. Solo cactus (quitar pájaros)
- Eliminar los sprites `BIRD1` y `BIRD2`.
- `spawnObstacle()` genera siempre un cactus (se quita la selección aleatoria de
  tipo `birdHigh`/`birdLow`).
- `makeObstacle()` se simplifica a solo cactus (mantiene `randGap` para la
  separación). Se quita la lógica de cuadros múltiples / aleteo: ya no hay
  obstáculos con `frames.length > 1`, así que se elimina el manejo de
  `flip`/`flipT` en el loop de `update()` y en `drawObstacles()`.

### 3. Control táctil — tap en cualquier parte
- Agregar un listener `pointerdown` en `window` que llama a `onJumpKey()` (la
  misma lógica de salto/empezar/reiniciar), con `e.preventDefault()` para evitar
  zoom/scroll por gesto. `pointerdown` cubre dedo y mouse en un solo handler.
- Conservar el handler de teclado existente para Espacio / ↑.

### 4. Escalar al ancho (responsive)
- CSS del canvas: `width:100%; max-width:720px; height:auto` (mantiene la
  resolución interna 660×220 y la relación 3:1; `image-rendering:pixelated`
  conserva el aspecto pixelado al escalar). Centrado vertical.
- Añadir `touch-action:none` y `user-select:none` al canvas para que el tap no
  dispare zoom ni selección de texto.
- El `<meta name="viewport" content="width=device-width, initial-scale=1">` ya
  existe; no se cambia.

### 5. Texto de ayuda
- Cambiar el hint de `Espacio/↑ saltar · ↓ agacharse` a
  **`Toca la pantalla o Espacio para saltar`**.

## Cosas que NO cambian
- Físicas de salto, velocidad creciente, score + récord en `localStorage`,
  máquina de estados READY/PLAYING/GAME OVER, colisión AABB.
- Sprite del cactus (7 líneas) y del personaje corriendo.

## Pruebas

- El harness de lógica pura (`index.html?test=1`) se mantiene. Se quitan las
  referencias a `game.ducking` en el test `getPlayerBox` (el resto de tests
  puros no cambian).
- Verificar en Node que los tests puros siguen pasando y que la geometría
  salto-vs-cactus sigue válida (el salto libra el cactus de 7 líneas).
- La verificación visual/táctil real la hace el usuario abriendo la URL de
  GitHub Pages en el celular.

### Criterios de aceptación
- [ ] No hay forma de agacharse; la tecla ↓ no hace nada.
- [ ] Solo aparecen cactus (ningún pájaro).
- [ ] Un tap en cualquier parte de la pantalla salta, empieza y reinicia.
- [ ] El teclado (Espacio/↑) sigue saltando en escritorio.
- [ ] En el celular el juego ocupa el ancho de la pantalla y se ve completo.
- [ ] El tap no provoca zoom ni scroll de la página.
- [ ] El hint indica el control táctil.
- [ ] Los tests puros (`?test=1`) siguen pasando.

## Despliegue
- Commit + push a `main` → GitHub Pages reconstruye y publica en
  `gerardoacuna.github.io/dino-vivo/`.
