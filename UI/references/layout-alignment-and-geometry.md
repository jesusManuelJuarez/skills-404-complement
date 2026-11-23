# Layout, Alineación, Grilla y Geometría

Basado en **Apple HIG**, **Material Design 3**, **Refactoring UI** y **Vercel Web Interface Guidelines**.

---

## 1. Grilla de 8px / 4px y Escala de Espaciado

Usa exclusivamente una escala discreta basada en **8px** con sub-pasos de **4px** para ajustes finos internos:

| Token | Valor | Uso típico |
| :--- | :--- | :--- |
| `--space-1` | `4px` | Separación icono-texto en chips, ajuste fino entre label y descripción corta. |
| `--space-2` | `8px` | Gap mínimo entre botones/controles vecinos, padding interno compacto. |
| `--space-3` | `12px` | Gap interno de filas, separación `<label>` → `<input>`. |
| `--space-4` | `16px` | Padding estándar de tarjetas móviles, gap entre campos relacionados. |
| `--space-6` | `24px` | Padding de tarjetas/modales desktop, separación entre grupos de formulario. |
| `--space-8` | `32px` | Separación entre bloques funcionales dentro de una misma vista. |
| `--space-12` | `48px` | Separación entre secciones principales de página. |
| `--space-16` | `64px` | Respiro de secciones hero / landing pages. |

**Regla estricta:** Prohibidos valores arbitrarios (`10px`, `13px`, `19px`, `27px`). Si un ajuste óptico requiere `1px` o `2px`, documéntalo como compensación óptica explícita, no como espaciado de layout.

---

## 2. Ley de Proximidad (*Refactoring UI* + *Apple HIG*)

El ojo agrupa lo que está cerca y separa lo que está lejos. Evita la **trampa del espaciado uniforme** (*equal spacing trap*) donde todo tiene `gap: 16px` y no se distingue qué pertenece a qué.

- **Invariante matemático:** `gap_interno <= gap_hermanos < gap_entre_grupos < padding_contenedor_externo` (cuando hay borde o fondo).
- **Ejemplo en formularios:**
  - `<label>` a su `<input>`: `4px–8px`.
  - Entre campos del mismo grupo (*Nombre* y *Apellido*): `16px`.
  - Entre grupos lógicos (*Datos personales* vs. *Dirección*): `24px–32px`.

---

## 3. Alineación Matemática y Óptica

1. **Toda posición debe justificarse:** Cada elemento debe alinearse a un eje común (borde izquierdo de columna, línea base de texto o centro óptico). Nada flotando "a ojo".
2. **Alineación de línea base (*Baseline*):**
   Cuando un icono acompaña a un texto, o una etiqueta acompaña a un valor, alinéalos al eje vertical/baseline (`align-items: center` en una sola línea, o `align-items: baseline` / ajuste de altura de línea en párrafos multilínea) equilibrando peso visual del trazo.
3. **Ajuste óptico ±1px (Vercel / Apple HIG):**
   La geometría pura a veces *parece* descentrada al ojo humano:
   - Un triángulo de *Play* centrado matemáticamente parece corrido a la izquierda → desplázalo `+1px` o `+2px` hacia la derecha.
   - Un icono circular junto a texto en mayúsculas puede requerir `-1px` vertical para equilibrar su masa visual.
4. **Alineación por tipo de contenido:**
   - **Texto y párrafos:** Siempre alineados al inicio (`text-align: start` / izquierda en LTR). Reserva `text-align: center` exclusivamente para *empty states*, encabezados cortos de marketing o tarjetas pequeñas (≤ 3 líneas).
   - **Números en tablas, precios y contadores:** Siempre alineados al final (`text-align: end` / derecha en LTR) con `font-variant-numeric: tabular-nums` para que unidades, decenas y decimales caigan exactamente en la misma columna.

---

## 4. Radios Concéntricos y Prevención de Desbordamientos

### Radios concéntricos (*Nested Border Radii* — Vercel)
Cuando un contenedor con `border-radius` envuelve otro elemento (botón, imagen, tarjeta interna), los radios deben ser concéntricos para que el grosor del espacio entre ambos sea uniforme en las esquinas:

$$R_{\text{interno}} = \max(0,\; R_{\text{externo}} - \text{padding})$$

```css
/* R_inner = R_outer (16px) - padding (8px) = 8px */
.card {
  border-radius: 16px;
  padding: 8px;
}

.card__inner-media {
  border-radius: 8px;
}
```
*Regla práctica:* Nunca pongas un hijo con `border-radius` mayor o igual al del padre si está separado por padding (`R_child <= R_parent` siempre).

### Prevención de cortes y desbordamientos (*Overflow* & *Safe Areas*)
- **Cero scroll horizontal accidental:** Ninguna pantalla estándar debe generar `overflow-x` en `320px–375px` (móvil) ni en `1280px+` (desktop).
- **Trampa clásica de Flex/Grid:** Los hijos flex/grid tienen `min-width: auto` por defecto y rompen el layout con strings largos (emails, URLs, IDs). Añade `min-width: 0` en contenedores que truncan o envuelven texto.
- **Safe Areas (Apple HIG):** En layouts *edge-to-edge* móviles, barras fijas inferiores y headers deben respetar `env(safe-area-inset-bottom)` y `env(safe-area-inset-top)`.
