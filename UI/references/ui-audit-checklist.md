# Checklist de Auditoría UI (Auto-Verificación y Reporte)

Úsalo antes de entregar cualquier pantalla/componente nuevo o al auditar código UI existente.

---

## 1. Grilla, Alineación y Geometría
- [ ] Todos los márgenes, paddings y gaps siguen la escala `4, 8, 12, 16, 24, 32, 48, 64` (sin números mágicos).
- [ ] Se cumple la ley de proximidad (gap interno < gap entre grupos distintos).
- [ ] Elementos horizontales comparten *baseline* o centro óptico; iconos asimétricos tienen ajuste ±1px si lo requieren.
- [ ] Contenedores anidados cumplen radios concéntricos (`R_inner = Math.max(0, R_outer - padding)`).
- [ ] No hay desbordamiento horizontal en móvil (`320–375px`), hay `min-width: 0` en flex/grid con truncado y se respetan *safe areas*.

## 2. Botones y Áreas Interactivas
- [ ] Todos los targets miden al menos **24×24px** en desktop y **44–48px** en táctil, con **≥ 8px** de separación.
- [ ] Hay máximo **1 botón Primary** compitiendo por vista o modal.
- [ ] Todo botón tiene los 6 estados: `default`, `hover`, `focus-visible`, `active`, `loading` (sin cambiar ancho) y `disabled`.
- [ ] Microcopy es imperativo y específico (*"Guardar cambios"*, no *"Aceptar"*).

## 3. Tipografía y Color
- [ ] Jerarquía clara en 3 niveles (peso + contraste + tamaño) usando tokens semánticos.
- [ ] Párrafos largos limitados a `max-width: 65ch` y alineados al inicio (`start`), nunca centrados en bloque.
- [ ] Cifras en tablas, precios y contadores usan `tabular-nums` y se alinean al final (`end`).
- [ ] Contraste ≥ **4.5:1** en texto normal y ≥ **3:1** en texto grande, bordes de inputs, iconos funcionales y foco.
- [ ] Ningún estado depende únicamente del color (color + icono + texto).

## 4. Semántica DOM y Cero Duplicados (Crítico)
- [ ] **Cero `id` duplicados:** componentes repetidos y listas generan IDs únicos (`useId()` o prefijo de ítem).
- [ ] **Cero etiquetas ambiguas repetidas:** botones en filas (`Editar`, `Eliminar`) incluyen contexto en `aria-label` manteniendo el texto visible al inicio.
- [ ] **Cero redundancia:** `<label>` no se duplica literalmente en `placeholder` y `aria-label`.
- [ ] Un único `<h1>` y un único `<main>`; múltiples `<nav>` diferenciados con `aria-label`.
- [ ] Todo elemento clicable usa `<button>` o `<a href>`, nunca `<div onClick>`.
- [ ] Iconos decorativos llevan `aria-hidden="true"`; botones solo-icono llevan `aria-label`.

## 5. Formularios, Modales y Movimiento
- [ ] `<label>` visible arriba del campo; `input` móvil `>= 16px`; no se bloquea tecleo ni `paste`.
- [ ] Botón *Submit* no oculta errores deshabilitándose prematuramente; errores inline enlazados con `aria-describedby`.
- [ ] Checkboxes y radios tienen hit target completo compartido con su `<label>`.
- [ ] Modales implementan *focus trap*, cierre con `Escape` y devuelven el foco al disparador.
- [ ] Animaciones solo usan `transform`/`opacity` (150–250ms) y respetan `prefers-reduced-motion: reduce`.

## 6. Anti-*AI Slop*
- [ ] El diseño tiene intención visual propia (no tarjetas genéricas apiladas sin jerarquía ni gradientes decorativos sin función).
- [ ] Para rendimiento técnico de red/carga (WebP/AVIF, `loading="lazy"`, *skeletons*, *debounce*, virtualización, *code splitting*), también se revisó con [QA Validator](../../QA-VALIDATOR/SKILL.md).

---

## Formato Estándar de Reporte (`archivo:línea`)

Usa este formato conciso (inspirado en Vercel Web Interface Guidelines) cuando operes en **Modo Auditoría UI**:

```markdown
### Veredicto UI: PASS | PASS WITH WARNINGS | BLOCK

#### Crítico (Semántica, Accesibilidad, Layout roto)
- `Component.tsx:L24` — **[ID duplicado / Semántica]** `id="email"` hardcodeado dentro de un `.map()`. Usar `useId()` o `email-${item.id}`. *(WCAG 2.2 / Semántica DOM)*
- `Component.tsx:L41` — **[Target / Nativo]** `<div onClick={...}>` con icono de `16×16px`. Reemplazar por `<button type="button">` con hit target mínimo `24×24px` (desktop) / `44×44px` (móvil) y `aria-label`. *(Apple HIG / WCAG 2.5.8)*

#### Alto (Botones, Formularios, Contraste, Geometría)
- `styles.css:L18` — **[Input móvil]** `font-size: 13px` dispara auto-zoom en iOS Safari; subir a `16px`. *(Vercel / Apple HIG)*

#### Medio / Bajo (Pulido y Tipografía)
- `PricingTable.tsx:L62` — **[Tipografía]** Columna de precios sin `font-variant-numeric: tabular-nums` ni alineación `end`. *(Refactoring UI / Vercel)*
```
