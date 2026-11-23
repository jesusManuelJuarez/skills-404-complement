# Tipografía, Color y Contraste

Basado en **WCAG 2.2 AA**, **Refactoring UI**, **Material Design 3** y **Vercel Web Interface Guidelines**.

---

## 1. Tipografía y Legibilidad

### Jerarquía en 3 niveles (*Refactoring UI*)
No dependas únicamente del tamaño para marcar importancia; combina **peso + contraste + tamaño**:
1. **Nivel Primario (Títulos, valores clave, encabezados):** Color fuerte de alto contraste + peso `600–700`.
2. **Nivel Secundario (Cuerpo, descripciones):** Color base legible + peso `400` + tamaño `14–16px`.
3. **Nivel Terciario (Metadatos, timestamps, ayuda de campos):** Color atenuado (siempre cumpliendo ≥ 4.5:1) + tamaño `12–14px`.

### Reglas cuantitativas de texto
- **Ancho máximo de lectura:** Limita bloques de prosa y descripciones largas a `max-width: 65ch` (rango óptimo 60–75 caracteres por línea).
- **Interlineado (`line-height`):** `1.1–1.25` en títulos grandes; `1.45–1.6` en texto de lectura; `1.25–1.4` en etiquetas compactas de UI.
- **Números alineados (`tabular-nums`):** Obligatorio `font-variant-numeric: tabular-nums` en tablas numéricas, precios, temporizadores y métricas para evitar saltos horizontales al cambiar dígitos.
- **Equilibrio y truncado:** Usa `text-wrap: balance` en títulos cortos. Si truncas texto con `ellipsis` / `line-clamp`, provee siempre forma accesible de ver el contenido completo.
- **Casing consistente (Vercel):** Elige una convención y mantenla en toda la barra de acciones (en español, *Sentence case* consistente: *"Guardar cambios"*, *"Nueva factura"*).

---

## 2. Color, Contraste y Superficies

- **Tokens semánticos:** Define tokens (`--surface-canvas`, `--surface-elevated`, `--text-primary`, `--text-muted`, `--border-subtle`, `--accent-primary`, `--danger`) en lugar de hexadecimales dispersos.
- **Contraste mínimo WCAG 2.2 AA:**
  - Texto normal (`< 24px`, o `< 18.66px` bold): **≥ 4.5:1**.
  - Texto grande (`≥ 24px`, o `≥ 18.66px` bold): **≥ 3:1**.
  - Componentes interactivos, bordes de inputs, iconos funcionales y anillos de foco (SC 1.4.11): **≥ 3:1** contra el fondo adyacente.
- **Nunca dependas solo del color:** En errores, éxitos o estados de tabla, combina **color + icono + texto/etiqueta**.
- **Superficies y Dark Mode (M3 + Vercel):**
  - En modo oscuro, evita `#000000` plano y sombras negras pesadas; usa escala tonal escalonada + bordes finos semitransparentes (`rgba(255, 255, 255, 0.08)`).
  - En modo claro, combina bordes sutiles semitransparentes con sombras multicapa suaves.
