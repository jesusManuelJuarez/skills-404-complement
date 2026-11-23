# Ejemplo 02 — Auditoría UI Modelo (`archivo:línea`)

Muestra la respuesta esperada cuando el usuario ejecuta **`ui-engineering-rules`** en Modo Auditoría UI sobre una pantalla de administración.

---

## Prompt del desarrollador

> "Audita `UserDirectoryPage.tsx` y `UserDirectory.css` con la skill de UI antes de aprobar el diseño."

```tsx
// UserDirectoryPage.tsx
export function UserDirectoryPage({ users, onDelete, onOpenFilter }: any) {
  return (
    <div className="page">
      <h1>Usuarios</h1>
      <div className="toolbar">
        <button className="btn-primary" onClick={onOpenFilter}>Filtrar</button>
        <button className="btn-primary">Nuevo usuario</button>
      </div>
      <h1>Lista activa</h1>
      <table>
        <tbody>
          {users.map((u: any) => (
            <tr key={u.id}>
              <td>{u.name}</td>
              <td style={{ textAlign: "left" }}>{u.credits}</td>
              <td>
                <button className="mini-icon" onClick={() => onDelete(u.id)}>
                  <TrashIcon />
                </button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

```css
/* UserDirectory.css */
.page { padding: 17px; }
.toolbar { display: flex; gap: 4px; }
.mini-icon { width: 16px; height: 16px; border: none; outline: none; }
```

---

## Respuesta Modelo Esperada del LLM

### Veredicto UI: `BLOCK`
La vista presenta problemas estructurales de semántica DOM (múltiples `<h1>`, falta de `<main>`, botón de solo icono sin nombre accesible), un hit target demasiado pequeño (`16×16px`) con foco desactivado (`outline: none`) y competencia visual entre dos botones primarios.

---

#### Crítico (Semántica, Accesibilidad, Targets)
- `UserDirectoryPage.tsx:L4,L10` — **[Múltiples `<h1>` y falta `<main>`]** Hay dos `<h1>` en la misma vista (`"Usuarios"` y `"Lista activa"`) y ningún landmark `<main>`. Usar `<main>` como contenedor raíz y bajar `"Lista activa"` a `<h2>`. *(WCAG 2.2 AA / Semántica DOM)*
- `UserDirectoryPage.tsx:L17` — **[Etiqueta inaccesible y duplicada]** `<button>` con `<TrashIcon />` sin `aria-label` contextual (los lectores de pantalla anuncian botones vacíos o repetidos). Agregar `aria-label={`Eliminar usuario ${u.name}`}` y `aria-hidden="true"` en `<TrashIcon />`. *(WCAG 2.5.3 / Semántica DOM)*
- `UserDirectory.css:L3` — **[Hit target < 24px y foco eliminado]** `.mini-icon` mide `16×16px` (viola mínimo WCAG `24×24px` y móvil `44×44px`) y elimina `outline: none` sin `:focus-visible`. Expandir hit target a mínimo `36×36px` (`44×44px` en `@media (pointer: coarse)`) y definir `:focus-visible` con contraste `>= 3:1`. *(Apple HIG / Material 3 / WCAG 2.2 AA)*

#### Alto (Jerarquía, Grilla y Separación)
- `UserDirectoryPage.tsx:L7-L8` — **[Competencia de Primary CTAs]** `"Filtrar"` y `"Nuevo usuario"` usan ambos `.btn-primary`. Mantener `.btn-primary` solo en `"Nuevo usuario"` y pasar `"Filtrar"` a variante secundaria/ghost. *(Refactoring UI)*
- `UserDirectory.css:L2` — **[Separación entre controles < 8px]** `.toolbar` tiene `gap: 4px`. Subir a `gap: 8px` mínimo para evitar errores de puntero/tap. *(Material 3 / Apple HIG)*
- `UserDirectory.css:L1` — **[Espaciado fuera de escala]** `padding: 17px` rompe la grilla de 8px/4px. Cambiar a `16px` (móvil) o `24px` (desktop). *(Material 3 / Refactoring UI)*

#### Medio (Tipografía y Alineación de Datos)
- `UserDirectoryPage.tsx:L15` — **[Números en tabla]** La columna `u.credits` está alineada a la izquierda y sin `tabular-nums`. Alinear a `end` y aplicar `font-variant-numeric: tabular-nums` (incluyendo su `<th>`). *(Vercel / Refactoring UI)*

---

### Plan de Corrección Sugerido
1. Corregir semántica (`<main>`, único `<h1>`, `aria-label` único por fila en eliminar).
2. Ajustar `.mini-icon` a hit target ≥ 24/44px con `:focus-visible` y separar `.toolbar` a `gap: 8px`.
3. Diferenciar jerarquía de botones, alinear créditos con `tabular-nums` y normalizar padding a `16px`/`24px`.

> ¿Quieres que aplique estos cambios directamente sobre `UserDirectoryPage.tsx` y `UserDirectory.css`?
