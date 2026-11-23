# Tarea 3 — Procesamiento de Imágenes

Dos aplicaciones web con Gradio que comparten el mismo núcleo (puntos de interés → emparejamiento → homografía con RANSAC):

- **Antes y después:** alinea dos fotos de una escena y marca los objetos que aparecieron o desaparecieron.
- **Panorama:** une 2 o 3 fotos tomadas girando el teléfono.

| Hito | Fecha |
|---|---|
| **Entrega** (cuaderno + zip de fotos propias + video) | **martes 1 de diciembre, 23:59** |
| **Demostración e interrogación** | **jueves 3 de diciembre**, en clases |

Evaluación: demostración grupal 50 % · **interrogación individual 50 %**. No hay informe.

> ⚠️ **Interrogación individual:** el profesor puede preguntarle a **cualquiera** sobre **cualquier parte**. El reparto es para avanzar en paralelo, pero todos deben entender el flujo completo (ver *Antes de la demo*).

---

## Estructura del entregable

```
Tarea3_Apellido1_Apellido2_Apellido3.ipynb   # construido sobre Tarea3_base.ipynb, ejecutado con salidas
imagenes/
├── cambios/        # set común: pares facil, medio, dificil (del profesor, no tocar)
├── panorama/       # set común: panorama de 3 fotos (del profesor, no tocar)
└── propias/        # nuestras fotos: ≥1 par de cambios + 1 panorama de 3 fotos
entrega/            # NO se sube al repo: propias.zip y video de respaldo
```

Se entrega en Canvas:
1. **Cuaderno** `Tarea3_Apellido1_Apellido2_Apellido3.ipynb`, ejecutado de principio a fin **con las salidas guardadas** (incluidas las 3 tablas).
2. **Fotos propias:** `imagenes/propias/` comprimida en `.zip`.
3. **Video de respaldo:** hasta 2 minutos, grabado desde el teléfono, con ambas aplicaciones funcionando.

---

## Reglas del cuaderno base (no romper)

- **No modificar** lo que ya viene implementado: `cargar`, `medir_tiempo`, `dibujar_pares`, `cortina`, `tablero`, `mostrar`, `analizar_par`, `dibujar_cambios`, `app_cambios`, `panorama_opencv`, `app_panorama` ni la interfaz de Gradio.
- Las funciones a implementar traen `raise NotImplementedError`. **Sus nombres y argumentos quedan exactamente como están**: la interfaz las llama directamente.
- **Parámetros fijos, no se cambian:** `LADO_MAX = 1024`, `RAZON = 0.8`, `UMBRAL_RANSAC = 5.0`, `MIN_INLIERS = 15`.
- Los modelos del método aprendido se cargan **una sola vez, fuera de `emparejar`**, para no contar su carga en el tiempo.
- En las tablas, indicar si los tiempos son en **CPU o GPU** (variable `DISPOSITIVO`).

---

## Reparto del trabajo

| | Persona 1 — **Núcleo** | Persona 2 — **Cambios** | Persona 3 — **Panorama y demo** |
|---|---|---|---|
| Código | **Paso 1** `emparejar` (SIFT, ORB, método aprendido) · **Paso 2** `estimar_homografia` · **Paso 3** `medidas` | **Paso 4** `alinear` · **Paso 5** `detectar_cambios` | **Paso 6** `panorama` |
| Tabla | **Tabla 1**: métodos en los pares de cambios (columna *alineó* mirando cortina/tablero) | **Tabla 2**: detección de cambios (reales, detectados, correctos, falsos, parámetros) | **Tabla 3**: métodos en el panorama + fila `cv2.Stitcher` |
| Fotos propias | — | **Par de cambios**: ≥ 2 objetos agregados/retirados + una dificultad (ángulo, luz, poca textura) | **Panorama de 3 fotos** |
| Entrega / demo | — | Caso de fracaso de cambios (paralaje, iluminación) | Colab + enlace Gradio en el teléfono, **video de respaldo**, zip de fotos propias |

Escriban su nombre en la tabla:
- Persona 1: *(nombre)*
- Persona 2: *(nombre)*
- Persona 3: *(nombre)*

### Dependencias entre partes

Pasos 4, 5 y 6 necesitan que `analizar_par` funcione, y eso depende de los Pasos 1, 2 y 3. Por eso:

1. **Persona 1** sube primero **SIFT + Pasos 2 y 3** (lo mínimo para que el resto pueda probar). ORB y el método aprendido van después.
2. Mientras tanto, **Personas 2 y 3** toman las fotos propias y preparan su código (pueden probar `alinear` y `panorama` en cuanto SIFT esté arriba).

### Guía por paso

**Paso 1 — `emparejar(a, b, metodo)` → `(pa, pb, n_a, n_b)`**
- `pa`, `pb`: `float32`, forma `(N, 2)`, fila *i* de `pa` ↔ fila *i* de `pb`.
- SIFT / ORB: `detectAndCompute` → `cv2.BFMatcher` (L2 para SIFT, Hamming para ORB) → `knnMatch(..., k=2)` + prueba de razón con `RAZON`.
- Aprendido: sugerido **DISK + LightGlue** de kornia (`kornia.feature.DISK`, `kornia.feature.LightGlueMatcher`); alternativas LoFTR o SuperPoint + LightGlue. Si se cambia, actualizar el nombre en `METODOS`.

**Paso 2 — `estimar_homografia(pa, pb)` → `(H, inliers)`**
- `cv2.findHomography(pa, pb, cv2.USAC_MAGSAC, UMBRAL_RANSAC)`. `inliers` booleano de largo N.
- Con menos de 4 pares → `None` y un arreglo de `False`.

**Paso 3 — `medidas(pa, pb, H, inliers, n_a, n_b, ms)` → dict**
- Claves exactas: `"puntos A"`, `"puntos B"`, `"pares"`, `"inliers"`, `"% inliers"`, `"error (px)"`, `"tiempo (ms)"`.
- Error: raíz del promedio de ‖H(pa) − pb‖² sobre los inliers, con `cv2.perspectiveTransform`. `NaN` si `H` es `None`. `% inliers` = 0 si no hay pares.

**Paso 4 — `alinear(antes, despues, H)` → `(antes_alineada, valida)`**
- `cv2.warpPerspective` con el tamaño de `despues`. `valida`: deformar con la misma `H` una imagen de unos.

**Paso 5 — `detectar_cambios(antes_alineada, despues, valida, umbral, k_morf, area_min)` → `(mascara, cajas)`**
- Pasos sugeridos: gris + suavizado → igualar media y desviación de `antes_alineada` a las de `despues` **dentro de `valida`** → diferencia absoluta + `umbral` → apertura y cierre `k_morf × k_morf` → erosionar `valida` para quitar el borde → `cv2.connectedComponentsWithStats` descartando área < `area_min` (entrega las cajas).
- `mascara` uint8 en {0, 1}; `cajas` lista de `(x, y, ancho, alto)`.

**Paso 6 — `panorama(imagenes, metodo)` → `(pano, filas)`**
- Referencia: `imagenes[len(imagenes) // 2]`. Homografía de cada otra imagen hacia la referencia con `analizar_par`; si < `MIN_INLIERS` → `ValueError` con mensaje claro.
- Lienzo: transformar las 4 esquinas, rectángulo envolvente, traslación `T`; si mide más de `4 · LADO_MAX` por lado → `ValueError`.
- Deformar con `T · H` y **promediar** donde se superponen.

---

## Fechas internas sugeridas

| Fecha | Hito |
|---|---|
| mar 24-nov | Persona 1: SIFT + Pasos 2 y 3 en el repo |
| mié 25-nov | Fotos propias tomadas (Personas 2 y 3) y subidas a `imagenes/propias/` |
| vie 27-nov | Pasos 4, 5 y 6 funcionando con SIFT |
| dom 29-nov | ORB + método aprendido; Tablas 1, 2 y 3 completas |
| lun 30-nov | Prueba completa en **Colab** + enlace Gradio en el teléfono; grabar **video** |
| **mar 1-dic** | **Entrega** (cuaderno con salidas, zip, video). No dejarlo para las 23:00 |
| mié 2-dic | **Ensayo de demo** y sesión cruzada de preguntas |
| **jue 3-dic** | **Demostración** |

---

## Antes de la demo (todos)

Cada uno explica su parte a los otros dos y se hacen preguntas cruzadas. Temas que el enunciado marca como interrogables:

- Flujo completo: detectar → describir → emparejar → RANSAC → homografía.
- Prueba de razón de Lowe, por qué L2 vs Hamming, qué cambia el método aprendido.
- Por qué muchos inliers no garantizan buena alineación, y por qué el error de inliers siempre es menor que el umbral de RANSAC.
- **Casos de fracaso:** paralaje (la homografía alinea un plano), iluminación, poca textura, ángulo. Tener **uno identificado** para mostrar.
- Cómo leer cada tabla.

Checklist del día:
- [ ] Cuaderno corre en Colab de principio a fin (dependencias instaladas).
- [ ] Enlace de Gradio abre en el teléfono y se puede proyectar.
- [ ] Tablas listas para mostrar.
- [ ] Un caso de fracaso preparado.
- [ ] Video de respaldo a mano.

---

## Cómo trabajar con git

Trabajamos todos en **el mismo cuaderno**. Para evitar conflictos:

1. **Antes de empezar:** `git pull`.
2. Editen **solo las celdas de su parte** (no muevan ni borren celdas de otros).
3. **Antes de hacer commit:** *Restart & Clear Output* (las salidas generan la mayoría de los conflictos). Solo la **versión final de entrega** se sube con salidas.
4. Commits chicos y seguidos: `git add`, `git commit`, `git pull`, `git push`.
5. Avisen por el grupo cuando suban algo y eviten editar el cuaderno dos al mismo tiempo.

Las fotos propias sí se suben (`imagenes/propias/`). El video y el zip van en `entrega/`, que está en `.gitignore`.
