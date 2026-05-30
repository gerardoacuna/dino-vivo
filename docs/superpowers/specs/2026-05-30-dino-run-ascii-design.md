# Dino Run ASCII — Diseño

**Fecha:** 2026-05-30
**Estado:** Aprobado, listo para plan de implementación

## Resumen

Clon del juego "Dino Run" de Google Chrome, pero el personaje es una criatura
coral hecha con arte ASCII (caracteres de bloque) en lugar del dinosaurio. El
personaje corre, salta para esquivar obstáculos de suelo y se agacha para
esquivar obstáculos voladores. La velocidad y el puntaje aumentan con el tiempo,
y el récord se guarda localmente.

## Decisiones clave (del brainstorming)

- **Formato:** un solo archivo `index.html`, sin dependencias, sin build, sin backend.
- **Estilo visual:** colorido/juguetón (cielo con degradado, suelo café).
- **Personaje:** arte ASCII de bloques, color coral, recreación exacta del arte aprobado.
- **Obstáculos:** también en ASCII (cactus verde, pájaro morado).
- **Mecánicas:** salto + agacharse, velocidad creciente, score + récord.
- **Sin** ciclo día/noche.

## Arquitectura

Un único `index.html` en `Sites/dino/` con `<style>` y `<script>` embebidos.
Render sobre un `<canvas>` 2D. El personaje y los obstáculos se dibujan con
`ctx.fillText` usando caracteres de bloque en fuente monoespaciada. Tamaño de
canvas fijo, centrado en la página.

## Arte (sprites ASCII)

Personaje (correr, cuadro 1 y 2 — alternan patitas), coral `#cf6a4c`:

```
 ▐▛███▜▌      ▐▛███▜▌
▝▜█████▛▘    ▝▜█████▛▘
  ▘▘ ▝▝        ▖▖ ▗▗
```

Personaje agachado: pose más baja/aplanada (a definir en implementación,
manteniendo el mismo lenguaje de bloques).

Cactus (suelo), verde `#3f9e54`:

```
  █
█ █
█ █ █
███ █
  ███
  █
  █
```

Pájaro (volador, aletea — cuadro 1 y 2), morado `#6b4fa3`:

```
▝▙   ▟▘       ▟███▙
 ▜███▛       ▝▛   ▜▘
```

Escena: cielo degradado (`#bfe3ff` → `#eafaff`), suelo café `#d9b382` con línea
`#a87f4f` y textura de guiones. Score arriba a la derecha en monoespaciada.

## Estados del juego

- **READY:** personaje quieto, texto "Presiona ESPACIO para empezar".
- **PLAYING:** game loop activo, spawn de obstáculos, score y velocidad suben.
- **GAME OVER:** congela la escena, muestra "GAME OVER" + score + récord +
  "ESPACIO para reintentar".

Transiciones: READY → (Espacio) → PLAYING → (colisión) → GAME OVER → (Espacio) → PLAYING.

## Componentes lógicos (dentro del archivo)

- **Sprites:** matrices ASCII + `drawBlock(ctx, lines, x, topY, fontPx, color)`.
- **Player:** posición Y, estado (corriendo / saltando / agachado), física de
  salto (impulso + gravedad), animación de patas (alterna cuadros por tiempo).
- **Obstacles:** lista de obstáculos activos; spawner que genera cactus (suelo) o
  pájaros (dos alturas: bajo = saltar, a la altura del cuerpo = agacharse) con
  separación aleatoria; se mueven a la izquierda a la velocidad actual.
- **Engine:** game loop con `requestAnimationFrame` y delta-time limitado;
  control de velocidad creciente; detección de colisiones AABB; dibujo de escena.
- **Input:** teclado.
- **Score:** puntaje actual (sube con el tiempo) + récord en `localStorage`.

## Controles

- **Salto:** `Espacio` o `↑`. Sin doble salto.
- **Agacharse:** `↓` (mantener presionado; al soltar vuelve a correr).
- **Empezar / reintentar:** `Espacio`.

## Mecánicas

- El personaje corre en su lugar (patas alternando); el mundo se desplaza a la
  izquierda.
- **Salto:** impulso hacia arriba + gravedad constante hasta volver al suelo.
- Obstáculos de **suelo** (cactus): se esquivan saltando.
- Obstáculos **voladores** (pájaro): a dos alturas — bajos (saltar) o a la altura
  del cuerpo (agacharse).
- **Velocidad:** empieza lenta y aumenta gradualmente con el score; la frecuencia
  de spawn se ajusta a la velocidad para que el juego siga siendo esquivable.

## Score

- Sube continuamente mientras se juega (incremento por tiempo/frame).
- **Récord (HI)** guardado en `localStorage`, persiste entre sesiones. Se muestra
  arriba a la derecha.

## Colisiones

Cajas rectangulares (AABB) alrededor del personaje y de cada obstáculo. La caja
del personaje será algo más pequeña que el sprite (perdón generoso) para que se
sienta justo. Choque = GAME OVER.

## Manejo de bordes / errores

- `localStorage` no disponible → el juego corre igual, sin persistir récord.
- Pestaña en segundo plano → `requestAnimationFrame` se pausa; al volver se usa
  delta-time limitado para evitar saltos bruscos.
- Redimensionar ventana → canvas de tamaño fijo centrado.

## Pruebas

Verificación principalmente **manual** (juego visual de un solo archivo): abrir
el HTML y comprobar cada criterio. La lógica pura (colisión AABB, avance de
score, spawn) se extrae en funciones puras dentro del script para poder
razonarlas/verificarlas por separado.

### Criterios de aceptación

- [ ] Al abrir, muestra READY con el personaje y la instrucción.
- [ ] Espacio inicia el juego.
- [ ] El personaje corre con animación de patas.
- [ ] Salto (Espacio/↑) esquiva cactus; no hay doble salto.
- [ ] Agacharse (↓) esquiva pájaros a la altura del cuerpo.
- [ ] El score sube continuamente y se muestra.
- [ ] La velocidad aumenta gradualmente con el score.
- [ ] Chocar lleva a GAME OVER con score y récord.
- [ ] El récord persiste tras recargar la página (si hay localStorage).
- [ ] Espacio reinicia desde GAME OVER.

## Fuera de alcance (YAGNI)

- Ciclo día/noche.
- Backend, leaderboard online, multijugador.
- Sonido (a menos que se pida después).
- Soporte táctil/móvil (se puede añadir luego).
