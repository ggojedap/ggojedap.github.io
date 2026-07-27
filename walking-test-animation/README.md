# Mini aplicación para probar personajes animados basados en pixi.js

Esta mini aplicación vanilla JS basada en pixi.js consiste en un único archivo HTML monolítico que contiene todo (estructura, estilos y JS) para probar la animación del ciclo de caminado de 4 personajes, utilizando el estándar JSON de spritesheets de PixiJS para lograr animaciones fluidas y una posición de descanso (resting pose) al liberar las teclas de movimiento.

Con las flechas del teclado, el personaje activo avanza en las 4 direcciones, desplazándose por el escenario mientras la animación del ciclo de caminado se ejecuta en la dirección correspondiente. Al liberar las teclas, el personaje adopta automáticamente su pose de descanso (standing) orientada hacia la última dirección de movimiento.

En la parte superior hay un selector de personaje, que intercambia el archivo JSON cargado de forma dinámica.

Los nombres de los personajes son: alberto, clara, diana y german. Para cada uno existen 2 archivos: uno .webp (textura) y otro .json (metadatos de spritesheet estándar de PixiJS).

## Características de los sprite sheets para los personajes

- **Nombre de archivo:** `[nombre personaje]-full-cycle-31.webp` y `[nombre personaje]-full-cycle-31.json`
- **Tamaño total de la imagen:** 2816px x 512px
- **Disposición:** 4 filas x 32 columnas
- **Tamaño de fotograma:** 88px x 128px
- **Fila 1:** animación ciclo de caminado "front" (31 fotogramas) + 1 fotograma final "standing front"
- **Fila 2:** animación ciclo de caminado "left" (31 fotogramas) + 1 fotograma final "standing left"
- **Fila 3:** animación ciclo de caminado "right" (31 fotogramas) + 1 fotograma final "standing right"
- **Fila 4:** animación ciclo de caminado "back" (31 fotogramas) + 1 fotograma final "standing back"

---

## Detalles Técnicos Indispensables para la Integración

Si deseas integrar este sistema de animación de personajes en un proyecto o aplicación más grande, debes tener en cuenta los siguientes aspectos técnicos clave:

### 1. Estructura del JSON del Spritesheet

El JSON debe cumplir con el estándar de spritesheets de PixiJS. Esto incluye:

- Una sección de `frames` con la definición detallada del rectángulo de cada frame de animación (`x`, `y`, `w`, `h`).
- Una sección de `animations` que agrupe y nombre las secuencias de fotogramas de la siguiente forma:

  ```json
  "animations": {
    "walk_front": ["front_00", "front_01", ..., "front_30"],
    "standing_front": ["standing_front"],
    "walk_left": ["left_00", ..., "left_30"],
    "standing_left": ["standing_left"],
    "walk_right": ["right_00", ..., "right_30"],
    "standing_right": ["standing_right"],
    "walk_back": ["back_00", ..., "back_30"],
    "standing_back": ["standing_back"]
  }
  ```

- Un bloque `meta` que defina la propiedad `"image"` apuntando al archivo de imagen correcto (ej. `"image": "alberto-full-cycle-31.webp"`). Ambos archivos deben coexistir en el mismo directorio público para que PixiJS resuelva automáticamente la ruta de la textura al cargar el JSON.

### 2. Carga de Recursos con PixiJS v7+

En lugar de segmentar las texturas manualmente con bucles de JavaScript, utiliza el cargador de recursos integrado de PixiJS (`PIXI.Assets.load`). Al cargar el archivo JSON, el loader de PixiJS detecta y analiza automáticamente las animaciones declaradas, exponiéndolas de forma inmediata en la propiedad `.animations` de la instancia del spritesheet:

```javascript
PIXI.Assets.load('ruta/del/personaje.json')
  .then((spritesheet) => {
    // Almacena las animaciones (mapeo de nombre de animación -> array de texturas)
    const characterTextures = spritesheet.animations;

    // Crea el sprite animado inicial usando la pose de descanso
    const character = new PIXI.AnimatedSprite(characterTextures.standing_front);
    character.anchor.set(0.5);
    character.stop(); // No requiere animación continua en reposo
    app.stage.addChild(character);
  });
```

### 3. Máquina de Estados para Animaciones (Movimiento vs. Reposo)

Para incorporar adecuadamente la posición de descanso al liberar las teclas, es necesario implementar un control de estados en el `ticker` de la aplicación (game loop) mediante dos variables de estado en el sprite:

- `character.currentDirection`: `"front"`, `"back"`, `"left"`, o `"right"`.
- `character.isWalking`: Booleano para rastrear si el personaje se encuentra actualmente en movimiento o reposo.

**Lógica de Transición en el Ticker:**

1. **Si el personaje se está moviendo (`moving === true`):**
   - Si la dirección cambia (`currentDirection !== newDirection`) **o** si el personaje acaba de empezar a caminar (`!isWalking`), actualiza las texturas a `"walk_" + newDirection`, establece la velocidad adecuada (`animationSpeed`), llama a `.play()` y marca `isWalking = true`.
2. **Si el personaje se detiene (`moving === false`):**
   - Si se encontraba caminando previamente (`isWalking === true`), actualiza las texturas a `"standing_" + currentDirection`, detiene la animación con `.stop()` para evitar ciclos innecesarios, y marca `isWalking = false`.

### 4. Limpieza de Recursos (Destrucción del Sprite)

En aplicaciones grandes con múltiples vistas o transiciones de escena, destruye el sprite y limpia la memoria correctamente al cambiar de personaje o salir del escenario para evitar fugas de memoria:

```javascript
if (character) {
  app.stage.removeChild(character);
  character.destroy({ children: true, texture: false, baseTexture: false });
  character = null;
}
```

*(Nota: Mantén `texture: false` y `baseTexture: false` si planeas reutilizar el spritesheet más tarde, o `true` si deseas liberar la memoria de texturas por completo al destruir la escena)*
