# Guía Completa para el Equipo de Frontend — Días 1-30

Este documento es la fuente de verdad para renderizar el contenido de los 30 días. El backend manda los textos **tal cual están en la BD**; el front es quien decide cómo mostrarlos. No hay que pedirle al backend más campos: **todo lo que necesitas ya viene en `leccion`**. Acá te explicamos qué viene, cómo mostrarlo y dónde está lo que hoy se ve mal.

---

## 0. Modelo de datos (qué recibís en `leccion`)

```
leccion = {
  dia_actual: number,
  tipo: string,                    // "instructivo" | "cuestionario"
  titulo: string,
  datos_leccion: {
    concepto/quote: string,        // frase con comillas, al inicio
    ejercicio: {
      nombre: string,
      instruccion: string,         // puede tener \n (multi-línea)
      pasos: [ { texto, respuesta_tipo } ],  // ← LOS CAMPOS SE GENERAN DE ACÁ
    },
    contenido: string,             // cuerpo, puede tener \n, •, ✓
    suplementacion: [ { nombre, dosis, horario, beneficio } ],
    principio: string,             // recomendación final, puede tener \n
    recursos: []
  },
  campos_respuesta: [              // ← USAR ESTO PARA LOS INPUTS (no pasos)
    { id, etiqueta, tipo, min, max }
  ]
}
```

**Regla de oro:** los `campos_respuesta` se generan automáticamente desde `ejercicio.pasos`.

- Cada `paso` → un `campo_respuesta` con el mismo `id`, orden, y una `etiqueta` = su `texto` (truncada a 300 caracteres).
- `campo.tipo` es `escala`, `accion`, o `texto`.
- El front **debe iterar `campos_respuesta`**, no `pasos`. `pasos` solo sirve como referencia del texto completo.

---

## 1. Los 3 tipos de respuesta (cómo mostrarlos)

### a) `escala` (botones 1-10)
Se usan para evaluaciones numéricas. El front muestra `min..max` botones (hoy: grid de botones 1-10). **Ya funciona bien** en `Lectura.tsx`. Guarda `answers[id] = number`.

**Días que usan `escala`:**
- Día 1 — `¿Cómo están tus niveles de energía hoy? (Escala 1-10)`
- Día 2 — las 3 señales (ver tabla en §3.1)
- Día 18 — Auditoría de Fricción (ver tabla en §3.5)
- Día 25 — Auditoría de Señales de Agotamiento (ver tabla en §3.9)

### b) `accion` (toggle "Marcar como hecho")
Pasos de práctica/ritual. Guarda `answers[id] = boolean`. **Ya funciona bien** en `Lectura.tsx`.

**Días con pasos `accion`:** 3, 4, 6 (paso 2), 9, 11, 12, 14, 16, 20 (paso 5), 23, 24 (pasos 1-3), 27, 28 (pasos 1-3).
> Ojo Día 25: el paso 5 (`Suma tus puntuaciones`) es `accion` aunque los 4 anteriores son `escala`. Es un botón de "ya sumé", no un campo numérico.

También hay días **completos de `accion`** (ningún campo de escritura): 3, 4, 9, 11, 12, 14, 16, 23, 27 y 28-pasos-1-3.

### c) `texto` (textarea)
Cualquier `paso` con `respuesta_tipo: 'abierta'` (o `'registro'`/`'estructurado'` mapeado) → `campo.tipo === 'texto'` → textarea libre. Guarda `answers[id] = string`. **Ya funciona** en `Lectura.tsx`.

**PERO:** en la mayoría de los días de escritura, el `texto` del paso **contiene varios campos de datos en una sola línea**, separados por `·`, `—`, `___`, `____/10`. El textarea plano escribe todo en un bloque. Eso es lo que estamos pidiendo mejorar: **esos pasos son tablas** (ver §2).

---

## 2. EL PATRÓN QUE MÁS IMPORTA: los pasos que son "tablas"

Muchos pasos `abierta` están redactados como **una fila de tabla** con varios campos separados por `·` (o `—`). El texto ya trae la estructura; el front solo tiene que **partir el texto** y mostrar **un input por campo** en vez de un único textarea.

Fíjate en las pistas de cada paso:
- `___` (3 guiones bajos) → campo de texto que el usuario debe escribir.
- `____/10` → campo numérico 1-10.
- `·` entre un campo y otro → separador de columnas.
- `—` separa la **etiqueta** de la columna de su instrucción/descripción.
- En los títulos tipo `Registro X #1 / #2 / #3` → son **filas repetidas** de una misma tabla.

**Recomendación general de implementación:**
1. Detecta cuántos `___`/`____/10` hay en `campo.etiqueta`. Si hay ≥2 → es una tabla.
2. Parte el texto por `·` (y por `—` dentro de cada parte) para armar **título / N columnas**.
3. Renderiza una *card* o tabla con un input por campo. Como el backend guarda **un solo string por paso**, el front debe **recomponer la línea** al enviar (`join` de los valores con ` · `) respetando el mismo formato. El backend no cambia.
4. El `__/10` (texto) se puede leer como el **default del placeholder**, pero el input es libre (ver §3.9 para excepción de Día 25).

> Si prefieren no hacer parsing automático, también es válido: dejar el `texto` como **descripción "plantilla"** (viso en gris, monoespaciado) y debajo **un textarea** donde el usuario completa TODO el registro de esa fila en una línea (ej: `comida agregada: ensalada · a las: 13:00 · día: lunes`). La clave es que el usuario **sepa qué datos completar en cada campo**.

Ahora sí, **día por día** con el desglose exacto.

---

## 3. Desglose de los días con tablas / campos de escritura

### 3.1 Día 2 — Tabla de las 3 Señales
3 pasos `escala` (ya renderizan botones 1-10). Presentación sugerida con 3 columnas:

| Señal | Descripción (va en `etiqueta` tras el `—`) | Mi Puntuación |
|---|---|---|
| Hambre Física | Sensaciones reales en el estómago | botón 1-10 |
| Cansancio Corporal | Fatiga muscular y energética | botón 1-10 |
| Ansiedad Mental | Tensión psicológica y preocupación | botón 1-10 |

No requiere escritura extra; solo que LUZCA como tabla si quieren (opcional).

### 3.2 Día 7 — Contrato de Micro-Compromiso (5 campos, UNA fila)
5 pasos `abierta`, cada uno un campo del mismo contrato:

1. `Fecha:` → texto corto (o date picker)
2. `Compromiso del día` → **elegir UNO** de las opciones listadas (texto largo)
3. `Hora específica` → texto
4. `Testigo (opcional)` → texto
5. `Firma` → texto

### 3.3 Día 8 — Auditoría de Bienestar Integral (tabla 5 filas × 3 cols)
5 pasos `abierta`, uno por área:

| Área de Bienestar | Pregunta de Evaluación | Mi Observación |
|---|---|---|
| Energía Física | ¿Subiste escaleras con menos fatiga? | input libre |
| Claridad Mental | ¿Te sientes más enfocado/a durante el trabajo? | input |
| Fuerza Muscular | ¿Tus músculos se sienten más firmes al tacto? | input |
| Calidad de Sueño | ¿Despertaste más descansado/a? | input |
| Estado de Ánimo | ¿Te sientes más optimista que la semana pasada? | input |

La `etiqueta` viene como `"Energía Física — ¿Subiste escaleras con menos fatiga?"`. Partí por ` — ` para separar área (col 1) de pregunta (col 2). La observación va en textarea.

### 3.4 Día 10 — Auditoría "Momentos de Protagonismo" (tabla 3 filas × 3 cols)
3 pasos `abierta`, cada uno es un momento con 3 datos separados por `·`:

| # | Situación | Acción tomada | Cómo me sentí |
|---|---|---|---|
| 1 | ___ | ___ | ___ |
| 2 | ___ | ___ | ___ |
| 3 | ___ | ___ | ___ |

`etiqueta` ejemplo: `1) Momento de Protagonismo #1: ·Situación: ___ ·Acción tomada: ___ ·Cómo me sentí: ___`. Tres inputs por fila.

### 3.5 Día 18 — Dos tablas
**Tabla 1 — Auditoría de Fricción (4 filas; pasos 1-4 `escala`):**

| Comportamiento Deseado | Fricción Actual | Nivel de Dificultad (1-10) |
|---|---|---|
| Tomar suplementos | están guardados en armario alto | botón 1-10 |
| Beber agua suficiente | botella vacía y lejos | botón 1-10 |
| Hacer ejercicio | ropa deportiva en otro cuarto | botón 1-10 |
| Comer saludable | frutas escondidas en el refrigerador | botón 1-10 |

**Tabla 2 — Rediseño de Facilidad (pasos 5-7 `abierta`, NO es tabla repetida, son 3 pasos con descripción larga):** cada paso describe la nueva estrategia (Estación de Bienestar Matutina / Hidratación Automática / Activación de Movimiento). Renderizarlos como **cards con textarea** (1 input por paso). No tienen `___` internos → no parsear.

> Plus: `contenido` del Día 18 trae la **"Optimización de la Estación de Suplementos"** como mini-tabla en texto plano (Mañana/Pre-entrenamiento/Noche). Respeta los saltos de línea (`white-space: pre-line`), o conviértela a tabla 4 cols: Horario | Suplementos | Contenedor | Recordatorio.

### 3.6 Día 13 — Auditoría de Disparadores Ambientales (3 pasos = 3 campos de 1 registro)
3 pasos `abierta`, cada uno con 2-3 sub-campos:

- Paso 1 · **Identificación de Saboteadores**: objeto/alimento problemático `___` · ubicación actual `___` · frecuencia de uso impulsivo `___` veces/día
- Paso 2 · **Reubicación Estratégica**: nueva ubicación (menos accesible) `___` · tiempo adicional requerido `___` minutos
- Paso 3 · **Sustitución Positiva**: objeto/elemento saludable en su lugar `___` · acción que promueve `___`

Parti por ` — ` el título, y por `·` los campos. Opcional: mostrar la mini-tabla de "Ejemplos Prácticos de Transformación" de la `instruccion` (Control remoto → Cajón → Mat de yoga, etc.).

### 3.7 Día 15 — Registro de Tolerancia (tabla 3 filas × 4 cols) + 3 pasos accion
El día mezcla 2 partes:
- **Pasos 1-3 (`accion`)**: la técnica ABLANDAR-PERMITIR-AMAR. Toogles.
- **Pasos 4-6 (`abierta`)**: Registro de Tolerancia, UNA fila por emoción:

| Emoción | Intensidad (1-10) | Duración Real | Estrategia Usada |
|---|---|---|---|
| Ansiedad | ____/10 | ___ min | ABLANDAR-PERMITIR-AMAR |
| Frustración | ____/10 | ___ min | Respiración + observación |
| Aburrimiento | ____/10 | ___ min | Tolerancia sin distracción |

Para Intensidad podés mostrar botones 1-10 (está `____/10` en el texto) o input numérico.

### 3.8 Día 17 — "Porqué" Vital (7 campos, 3 fases)
7 pasos `abierta`:
- **Paso 1·5 preguntas progresivas** (¿Qué es lo más importante? + 4 de profundización). Cada una su propio input.
- **Paso 6 · Destilación del Propósito**: completar `"Cuido mi salud integral porque quiero ___ para/con ___"` → 2 inputs (quiero ___, para/con ___). No romper las comillas de la plantilla: mostrar la frase como texto y los 2 campos inline si se puede.
- **Paso 7 · Anclaje Visual**: texto libre (dónde pondrás el post-it).

### 3.9 Día 25 — Auditoría de Señales de Agotamiento (escala sumable)
5 pasos: 4 `escala` (Fatiga física, Niebla mental, Irritabilidad emocional, Motivación reducida) + 1 `accion` ("Suma tus puntuaciones").

**IMPORTANTE:** los 4 valores de escala se SUMAN (total /40). La `instruccion` ya define la interpretación:
- 0-15 → energía óptima
- 16-25 → fatiga moderada / descanso activo
- 26-40 → agotamiento significativo / descanso obligatorio

**Recomendación:** al seleccionar los 4 botones, mostrar el **total en vivo** (suma) y el nivel resultante según los rangos. Idealmente un resumen tipo "Tu puntuación es 27/40 → Descanso obligatorio". Esto le da valor UX real.

### 3.10 Día 22 — Nota de Re-enfoque (7 campos, un solo registro)
7 pasos `abierta`, cada uno un campo de la misma nota:

1. Fecha
2. Situación
3. Mi respuesta
4. Dato que esto me enseña
5. Mi próxima acción de autocuidado
6. Razón por la que elijo esta acción
7. Firma de autocompasión

`etiquetas` = `1) Fecha: ___`, `2) Situación: ___`, etc. Simplemente limpiar el `1)` y `___` para etiquetas cortas, 7 inputs verticales (formulario).

### 3.11 Día 26 — Preparación Mental (2 fases; pasos 1-6)
- **Pasos 1-3 (Fase 1 · Escenarios)**: cada uno tiene 3 sub-campos: evento social `___` · presión esperada `___` · nivel de desafío `____/10`.
- **Pasos 4-6 (Fase 2 · Guiones asertivos)**: UNA plantilla por paso: `[Reconocimiento] + [Límite claro] + [Alternativa positiva]` → un solo input de texto largo (texto libre), mostrando la fórmula de la `etiqueta` como guía.

### 3.12 Día 29 — Límites Empoderados (2 partes)
- **Pasos 1-3 (Fase 1 · Saboteadores)**: cada uno 4 sub-campos: persona `___` · tipo de presión `___` · frecuencia `___/semana` · estrategia necesaria `___`.
- **Pasos 4-6 (Fase 2 · Límites)**: un input de texto libre con plantilla `[Reconocimiento de la relación] + [Límite claro] + [Conexión con valores futuros]`.
- **Paso 7 (Fase 3 · Visualización)**: texto libre (describir la visualización).

> Frecuencia (____/semana) es numérico pero libre. Desafío (____/10) puede ser numérico libre o botones.

### 3.13 Día 24 — Registro de Interacciones Empáticas (tabla 3 filas × 4 cols)
Pasos 4-6 (`abierta`, pasos 1-3 son toggles de la Pausa Empática):

| Situación Desafiante | Reacción Inicial | Pausa Empática Aplicada | Resultado |
|---|---|---|---|
| · · · | · · · | · · · | · · · | (x3 filas)

### 3.14 Día 28 — Registro de Conexiones Auténticas (tabla 3 filas × 3 cols)
Pasos 4-6 (`abierta`, pasos 1-3 toggles):

| Persona | Algo Nuevo que Aprendí | Conexión Emocional (1-10) |
|---|---|---|
| ___ | ___ | ____/10 | (x3 filas)

La última columna es numérica (sugerido: input numérico o botones 1-10).

### 3.15 Día 30 — Cierre y Carta al Futuro (10 pasos)
Final del programa, es un formulario de cierre:
- **Pasos 1-6**: Mayor victoria de cada bloque (1-5, 6-10, 11-15, 16-20, 21-25, 26-30). Un input por bloque, con el nombre del bloque como etiqueta.
- **Paso 7**: Reflexión sobre el camino recorrido (texto libre). *(verificar índice real en seed)*
- **Paso 8**: Suplementación personalizada para la vida — admite **hasta 4 filas** de la tabla `Suplemento | Dosis | Horario | Razón específica` (la `etiqueta` dice "completa hasta 4"). Si el backend guarda un solo string, unir las filas con ` · `.
- **Paso 9**: Mis 5 Prácticas No-Negociables → 5 inputs.
- **Paso 10**: Carta a tu Futuro Yo (texto libre grande) — mostrar la plantilla de la `etiqueta` como guía.
- Idealmente, al final: pantalla/banner de **graduación + conclusión de toda la app** al completar (usa `conclusion` del Día 30).

### 3.16 Días de reflexión libres (sin tabla, sin toggles)
Días con solo textareas: **1** (3 abiertas), **5** (4), **17**, **20** (pasos 1-4 + 6-8), **21** (4). Ya se ven bien con textarea simple.

---

## 4. Problemas conocidos y fixes pendientes (orden de prioridad)

### ⚠️ ALTO — Saltos de línea colapsados
`Lectura.tsx` renderiza `contenido`, `principio` y la `instruccion` en `<p>` que **colapsa los `\n`**. Días con multi-línea crítica: **4, 12, 14, 15, 17, 22, 26, 29, 30** (y las `cabecera` de bloque). Resultado hoy: bullets y fases se ven corridos.

**Fix:** `white-space: pre-line` sobre esos nodos (o split por `\n` → map a bloques/li). Ver §6.

### ⚠️ ALTO — Pasos redactados en una sola línea se ven como texto corrido
Los `campos_respuesta` de los días del §3 traen la `etiqueta` con `___`/`____/10` en medio de la frase. Con textarea simple, el usuario tiene que escribir "entre guiones" sin saber bien qué completar. Es la mejora principal de esta entrega: **parsear y mostrar inputs por campo** (o al menos contexto de plantilla + textarea).

### ⚠️ MEDIO — Dosis/horario vacíos (Día 9)
Día 9: suplemento único `Ashwagandha + Complejo B + Omega-3` con `dosis` y `horario` = `""`. `Lectura.tsx:368` renderiza `{sup.dosis} · {sup.horario}` → muestra un ` · ` suelto.

**Fix:** mostrar la línea "dosis · horario" **solo si** `sup.dosis || sup.horario`. El `beneficio` de ese ítem trae el texto "Stack completo: ..." que debe verse como texto principal.

### ⚠️ MEDIO — Suplementación vacía []
Días **13, 18, 20, 26, 29** tienen `suplementacion: []`. El bloque de suplementos (Lectura.tsx ~línea 353) ya se salta si `.length === 0`; verificar que no muestre el header vacío ni una card vacía.

### ⚠️ BAJO — Conclusión de bloque (cierres)
Días **5, 10, 15, 20, 25, 30** tienen `conclusion` a nivel raíz del contenido (no dentro de `datos_leccion`). Cuando el usuario completa el ÚLTIMO día de un bloque, `conclusion` llega en la respuesta de `completeDay`/`advanceDay`. **El front debe mostrarla** como cierre de bloque (modal/banner). Ver §6 para el formato (usa `\n\n`).

### ✅ YA—Escala y toggle
Los botones 1-10 y los toggles de `accion` ya cumplen. Solo falta el caso Día 25 (suma en vivo, §3.9).

---

## 5. Días × tipo (resumen rápido para QA)

| Tipo de contenido | Días |
|---|---|
| Solo toggles `accion` (sin escritura) | 3, 4, 9, 11, 12, 14, 16, 23, 27 |
| Toggles + tablas de registro | 15, 24, 28 |
| Escalas (botones 1-10) | 1 (1 escala), 2 (3), 18 (4), 25 (4) |
| Formularios/campos múltiples | 7, 8, 13, 17, 19, 20, 22, 26, 29, 30 |
| Reflexión libre (textareas) | 1, 5, 21 |

> **Día 19** (Nutriendo la Energía): pasos 2-5 son "Identificación Nutricional — [Nutriente]: alimento ___ · [beneficio]" → 4 inputs con etiqueta de nutriente (Proteína, Carbohidratos, Grasas, Vitaminas/minerales). El paso 6 es frase a completar (2 campos). Paso 1 es toggle. O sea: **NO es tabla**, es lista de ítems con un input cada uno.

---

## 6. Estilo y formato sugerido

- **`white-space: pre-line`** en `contenido`, `principio`, `instruccion` y `etiquetas` de pasos, o dividir por `\n`.
- **Bullets `✓` y `•`**: convertirlos en visual bullets (`<li>`) cuando vengan al inicio de línea, o mantenerlos como texto con `pre-line`. No dejarlos pegados.
- **Plantillas con `___`**: búscalas dentro de la etiqueta y dales un **input inline** (manteniendo el resto de la frase legible). Ej. Día 26: `nivel de desafío [ input 1-10 ] /10`.
- **Fórmulas entre corchetes** `[Reconocimiento] ...`: mostrarlas como **chip/etiqueta gris** arriba del textarea (ej. Días 26, 29, 30).
- **Registro repetido (#1/#2/#3)**: agrupalo en una tarjeta por fila con numeración visible y sus inputs según las columnas de cada tabla del §3.

---

## 7. Caso prueba-acceptance para QA (qué tiene que verse bien y poder completarse)

1. **Día 2**: 3 filas con botones 1-10 (Señal | Descripción | Puntuación).
2. **Día 8**: 5 áreas con su pregunta + observación por input.
3. **Día 15**: 3 toggles (ABLANDAR-PERMITIR-AMAR) + tabla de 3 filas con intensidad/duración/estrategia.
4. **Día 25**: al tocar el 4º botón de escala, aparece el total en vivo y el nivel (óptimo/activo/obligatorio).
5. **Día 26**: 3 escenarios con 3 campos + 3 guiones con textarea.
6. **Día 30**: 10 pasos completables + al finalizar, banner de graduación/carta.
7. **Todos los días**: ningún `:\n` ni `· ` ni `—` suelto en pantalla; saltos de línea respetados.
8. **Día 9 suplementación**: no aparece el ` · ` vacío.