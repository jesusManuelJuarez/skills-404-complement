---
name: ui-engineering-rules
description: >-
  Diseña, construye y audita interfaces de usuario aplicando invariantes universales
  de ingeniería UI y accesibilidad (Apple HIG, Material Design 3, Vercel Web Interface
  Guidelines, Refactoring UI, Anthropic frontend-design y WCAG 2.2 AA / NN/g).
  Usar cuando el desarrollador cree o revise pantallas, componentes, botones,
  tipografía, formularios, tablas, modales, alineación, contraste, accesibilidad o
  semántica DOM (evitar etiquetas o IDs duplicados). NO usar para arquitectura
  backend/BD (hexagonal-architecture), code smells de lógica pura (clean-code) ni
  auditorías de red/N+1/seguridad servidor (qa-validator).
metadata:
  version: "1.0.0"
  category: ui
---

# Reglas Invariantes de Ingeniería UI

Skill agnóstica del estilo visual que fija los **mínimos no negociables** de alineación, dimensiones, tipografía, semántica DOM y usabilidad en cualquier interfaz profesional.

---

## 1. Marco de Referentes

| Referente / Estándar | Aporte operativo en esta skill |
| :--- | :--- |
| **Apple HIG** | Alineación predecible, jerarquía por espacio/peso, *hit targets* táctiles mínimos **44×44pt** y *safe areas*. |
| **Material Design 3** | Grilla **8px/4px**, *hit targets* **48×48dp** (separación ≥ 8dp), escala tipográfica y superficies tonales. |
| **Vercel Web Interface Guidelines** | Alineación óptica ±1px, radios concéntricos, formularios sin fricción (input móvil ≥ 16px), no pre-deshabilitar submit, reporte `archivo:línea`. |
| **Refactoring UI** | Ley de proximidad, jerarquía de 3 niveles de texto, 1 Primary CTA por vista, etiquetas encima de inputs y tokens. |
| **Anthropic `frontend-design`** | Dirección estética intencional antes de codear para evitar *AI slop*. |
| **WCAG 2.2 AA / NN/g** | Contraste ≥ 4.5:1 / 3:1, *target size* ≥ 24×24px, cero `id`/etiquetas duplicadas, lectura `65ch`, teclado y `focus-visible`. |

### Los 10 Invariantes No Negociables
1. **Grilla 8px/4px y Proximidad:** Múltiplos de `4, 8, 12, 16, 24, 32, 48, 64`; gap interno < gap entre grupos.
2. **Alineación Matemática + Óptica y Radios Concéntricos:** *Baseline* común, ajuste ±1px en iconos asimétricos y `R_child <= R_parent` (`R_inner = R_outer - padding`).
3. **Hit Targets y Separación:** ≥ **24×24px** desktop y **44–48px** táctil, con ≥ **8px** entre controles vecinos.
4. **Botones:** Máximo **1 Primary CTA** por vista/modal y **6 estados** (`default`, `hover`, `focus-visible`, `active`, `loading` sin cambiar ancho, `disabled`).
5. **Tipografía:** 3 niveles por peso/contraste, `max-width: 65ch`, `tabular-nums` en cifras e inputs ≥ `16px` en móvil.
6. **Cero Duplicados DOM:** `useId()` para IDs en listas/componentes, `aria-label` contextual en botones repetidos, único `<h1>` y único `<main>`.
7. **Formularios Sin Fricción:** `<label>` arriba (nunca solo `placeholder`), no bloquear tecleo/paste ni pre-deshabilitar submit, errores inline ARIA.
8. **Contraste WCAG 2.2 AA y No Solo Color:** ≥ **4.5:1** texto normal y **3:1** texto grande/bordes/foco; estado con color + icono + texto.
9. **Teclado y HTML Nativo:** `<button>` para acciones, `<a>` para rutas (prohibido `<div onClick>`), `focus-visible` y *focus trap* + `Escape` en modales.
10. **Movimiento Prudente:** Solo `transform` y `opacity` (150–250ms) respetando `prefers-reduced-motion: reduce`.

---

## 2. Modos de Operación

1. **Diseño / Construcción de UI:** Definir jerarquía, tokens y estados antes de emitir TSX/CSS cumpliendo los 10 invariantes.
2. **Auditoría UI (`archivo:línea`):** Reportar violaciones en escala única (`Crítico`, `Alto`, `Medio`, `Bajo`) con referente y solución.
3. **Refactorización UI:** Corregir alineación, targets, estados, semántica DOM y duplicados preservando la estética válida.
4. **Mentoría Visual:** Explicar *por qué* un espaciado, radio o jerarquía falla citando la regla correspondiente.

---

## 3. Protocolo de Ejecución

1. **Contexto e intención:** Identifica tipo de pantalla, viewport objetivo y dirección visual antes de escribir código.
2. **Semántica y estructura primero:** Landmarks (`<main>`, `<nav>`), único `<h1>`, controles nativos e IDs únicos (`useId()`).
3. **Geometría, tipografía y color:** Escala 8px/4px, radios concéntricos, 3 niveles tipográficos y contraste AA.
4. **Estados e interacción:** 6 estados en botones, validación inline de formularios y navegación completa por teclado.
5. **Auto-auditoría:** Ejecuta [ui-audit-checklist.md](./references/ui-audit-checklist.md) antes de entregar.

---

## 4. Guardrails

- **Anti-*AI Slop*:** Evita tarjetas idénticas apiladas con sombras pesadas, gradientes morados gratuitos o diseños sin jerarquía.
- **Prohibidos valores mágicos:** Sin espaciados fuera de escala (`13px`, `17px`) ni colores sin token semántico.
- **Prohibido sacrificar a11y por estética:** Nunca quites `:focus-visible`, bajes contraste de labels ni uses `<div onClick>`.
- **Higiene técnica estricta:** Cero `any` en TSX nuevo/refactorizado, cero emojis y comentarios breves (≤ 1 línea) solo para el *por qué*.

---

## 5. Mapa de Referencias y Sinergia

| Recurso | Cuándo leerlo | Enlace |
| :--- | :--- | :--- |
| **Layout, Alineación y Geometría** | Grillas 8px/4px, proximidad, ajuste óptico ±1px y radios concéntricos. | [layout-alignment-and-geometry.md](./references/layout-alignment-and-geometry.md) |
| **Botones, Formularios e Interacción** | CTAs, hit targets (24/44/48px), 6 estados, inputs, modales y animaciones. | [buttons-forms-and-interactions.md](./references/buttons-forms-and-interactions.md) |
| **Tipografía y Color** | Escala 3 niveles, `65ch`, `tabular-nums`, contraste AA y Dark Mode tonal. | [typography-and-color.md](./references/typography-and-color.md) |
| **Semántica DOM y Cero Duplicados** | `useId()`, etiquetas únicas en listas, landmarks, `<h1>` y HTML nativo. | [semantics-and-accessibility.md](./references/semantics-and-accessibility.md) |
| **Checklist de Auditoría UI** | Auto-verificación y formato de reporte `archivo:línea`. | [ui-audit-checklist.md](./references/ui-audit-checklist.md) |
| **Ejemplo 01 (Refactor TSX/CSS)** | Refactor Antes/Después corrigiendo errores reales de UI sin `any`. | [01-ui-component-refactor-before-after.md](./examples/01-ui-component-refactor-before-after.md) |
| **Ejemplo 02 (Auditoría UI)** | Salida modelo `archivo:línea` con veredicto y severidades estándar. | [02-ui-audit-report-sample.md](./examples/02-ui-audit-report-sample.md) |

### Sinergia con otras skills
- Para eficiencia de carga y red (WebP/AVIF, `loading="lazy"`, *skeletons*, *debounce*, *Optimistic UI*, streaming IA, SSR/SSG, *code splitting*, virtualización), combina con [QA Validator](../QA-VALIDATOR/SKILL.md).
- Para organizar props, estado y modularidad limpia en TypeScript, apóyate en [Clean Code](../ARQUITECTURE/CleanCode/SKILL.md).

---

## 6. Contrato de Salida

- **En Auditoría UI:** Veredicto (`PASS` | `PASS WITH WARNINGS` | `BLOCK`) → Hallazgos `archivo:línea` por severidad (`Crítico`, `Alto`, `Medio`, `Bajo`) con referente y fix concreto.
- **En Construcción / Refactor UI:** Decisiones de jerarquía y tokens → Código TSX/CSS limpio sin `any` ni emojis → Checklist breve verificado.
