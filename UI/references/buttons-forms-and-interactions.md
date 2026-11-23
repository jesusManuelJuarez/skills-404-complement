# Botones, Dimensiones, Formularios e Interacciones

Basado en **Apple HIG**, **Material Design 3**, **Vercel Web Interface Guidelines**, **Refactoring UI** y **WCAG 2.2 AA**.

---

## 1. Dimensiones de Botones y Áreas Interactivas (*Hit Targets*)

### Estándares mínimos no negociables

| Contexto / Estándar | Mínimo de *Hit Target* | Altura visual habitual | Separación mínima entre targets |
| :--- | :--- | :--- | :--- |
| **WCAG 2.2 AA (SC 2.5.8)** | **24 × 24 px** (absoluto web) | `32–36 px` (compacto desktop) | ≥ `8 px` |
| **Apple HIG (táctil / iOS)** | **44 × 44 pt** | `44 pt` | ≥ `8 px` |
| **Material Design 3** | **48 × 48 dp** | `40 dp` visual + área expandida a `48 dp` | ≥ `8 dp` |

### Regla de expansión invisible del *Hit Target*
Si el icono visual mide `16×16px` o `20×20px` (ej. botón cerrar `×`, menú `⋮`, eliminar fila), **expande su área clicable** con padding o un pseudo-elemento `::after` transparente hasta alcanzar al menos `24×24px` en desktop y `44×44px` en móvil.

```css
.icon-button {
  position: relative;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 36px;
  min-height: 36px;
}

@media (pointer: coarse) {
  .icon-button {
    min-width: 44px;
    min-height: 44px;
  }
}
```

### Proporción, Jerarquía y Copy de Botones
- **Proporción cómoda:** Padding horizontal ≥ **2 × padding vertical** (ej. `padding: 8px 16px` o `10px 20px`) y `gap: 8px` cuando lleva icono + texto.
- **Un único botón Primary por vista/modal:** Si hay 3 botones sólidos del mismo color compitiendo en pantalla, la jerarquía está rota.
  - **Primary:** Acción principal única (relleno sólido de alto contraste).
  - **Secondary:** Acción alternativa (*Cancelar*, *Exportar*) con borde sutil o relleno neutro.
  - **Tertiary / Ghost:** Acciones auxiliares o repetidas en filas de tabla.
  - **Destructive:** Rojo semántico; si es irreversible (*Eliminar cuenta*), exige confirmación explícita o mecanismo de *Undo*.
- **Microcopy específico e imperativo:** Usa verbo + objeto (*"Guardar cambios"*, *"Crear proyecto"*, *"Eliminar factura"*). Evita textos genéricos ambiguos como *"Aceptar"*, *"OK"* o *"Enviar"*.

### Los 6 Estados Obligatorios de Todo Botón
1. **`default`:** Claramente distinguible como elemento interactivo.
2. **`hover`:** Cambio sutil de tono/fondo (solo en dispositivos con `@media (hover: hover)`).
3. **`focus-visible`:** Anillo de foco visible (`outline: 2px solid ...; outline-offset: 2px`) con contraste ≥ 3:1; **nunca** `outline: none` sin reemplazo.
4. **`active` / `pressed`:** Retroalimentación física inmediata (ej. leve oscurecimiento o `scale(0.98)`).
5. **`loading`:** Muestra indicador de carga y desactiva doble disparo **conservando el ancho del botón** (no borres la etiqueta causando salto de layout). Aplica un retraso de aparición ~150–300ms en acciones rápidas para evitar parpadeo (Vercel).
6. **`disabled`:** Contraste atenuado e `aria-disabled` / `disabled` coherente; explica por qué está bloqueado si no es obvio.

---

## 2. Reglas de Formularios Sin Fricción

1. **`<label>` siempre visible arriba del campo:**
   Las etiquetas a la izquierda rompen el escaneo visual y colapsan en pantallas estrechas. El `placeholder` **nunca** reemplaza al `<label>`; úsalo solo para mostrar formato de ejemplo (`ej. ana@example.com`).
2. **Tamaño de fuente `>= 16px` en inputs móviles (Vercel / Apple):**
   En iOS Safari, un `<input>`, `<select>` o `<textarea>` con `font-size < 16px` dispara zoom automático molesto al enfocar.
3. **No bloquees tecleo ni `paste`:**
   Permite escribir o pegar al usuario (incluidos espacios o guiones en teléfonos/tarjetas) y normaliza o muestra validación inline clara.
4. **No pre-deshabilites el botón *Submit* para validar:**
   Un botón bloqueado sin explicación frustra al usuario porque no sabe qué campo falta. Permite intentar enviar, enfoca el primer campo inválido y muestra errores inline asociados con `aria-invalid="true"` y `aria-describedby`.
5. **Checkboxes y radios sin zonas muertas (*No Dead Zones*):**
   Envuelve control y texto en un único `<label>` clickable (con `gap: 8px` y alineación óptica a la primera línea de texto si hay salto de línea).
6. **Ayuda inline antes que tooltips:**
   En pantallas táctiles no existe *hover*. Coloca el texto de ayuda clave debajo del input en texto terciario (`12–14px`) antes de esconderlo en un tooltip.

---

## 3. Modales, Drawers y Movimiento Prudente

- **Modales y Drawers:**
  - Al abrir, mueve el foco dentro del modal y aplica **focus trap** (`Tab` / `Shift+Tab` no salen al fondo).
  - Cierra con `Escape` y click en backdrop (salvo confirmación crítica bloqueante) y **devuelve el foco al botón que lo abrió**.
  - En móvil, evita modales diminutos centrados con scroll interno incómodo; prefiere *sheet* inferior o pantalla completa.
- **Animaciones (Core Web Vitals + a11y):**
  - Anima únicamente propiedades compuestas por GPU: `transform` y `opacity` (nunca `width`, `height`, `top`, `left`, `margin`).
  - Duraciones cortas de interfaz: **150–250 ms** (`ease-out` al entrar, `ease-in` al salir).
  - Obligatorio respetar `@media (prefers-reduced-motion: reduce)` anulando desplazamientos largos y parallax.
