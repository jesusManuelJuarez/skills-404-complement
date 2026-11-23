# Semántica DOM, Accesibilidad y Cero Etiquetas Duplicadas

Basado en **WCAG 2.2 AA**, **Apple HIG** y **Vercel Web Interface Guidelines**.

---

## Cero Etiquetas, IDs o Landmarks Duplicados (Regla Crítica)

Las auditorías reales fallan con frecuencia por errores invisibles en el árbol de accesibilidad y el DOM:

| Error a erradicar | Por qué rompe la UI / a11y | Regla obligatoria |
| :--- | :--- | :--- |
| **`id` duplicados en el DOM** | En listas o componentes montados 2+ veces, `<input id="email">` hace que `<label for="email">` enfoque siempre el primer campo y rompe `aria-describedby`. | Usa **`useId()`** (React) o prefijos únicos por instancia (`${baseId}-${item.id}`). Nunca hardcodees `id` estáticos en componentes reutilizables. |
| **Etiquetas interactivas repetidas sin contexto** | Tener 15 botones `"Editar"` o `"Eliminar"` en una tabla impide distinguir a qué fila pertenece cada uno en lectores de pantalla y tests E2E. | Si el texto visible es `"Editar"`, añade `aria-label={`Editar ${item.name}`}` conservando el texto visible al inicio (*Label in Name*, WCAG 2.5.3). |
| **Etiquetas redundantes en el mismo campo** | Poner el mismo texto en `<label>`, `placeholder` y `aria-label` provoca que los lectores de pantalla lean tres veces lo mismo. | `<label>` da el nombre principal; `placeholder` solo muestra ejemplo de formato (o se omite); `aria-describedby` apunta al texto de ayuda/error. |
| **Múltiples `<h1>` o saltos de encabezados** | Desorienta la navegación estructural de usuarios y SEO. | Exactamente **un `<h1>`** por vista; progresión ordenada `h1 -> h2 -> h3` sin saltar niveles por tamaño visual. |
| **Landmarks duplicados sin diferenciar** | Múltiples `<main>` o dos `<nav>` sin etiqueta confunden la navegación por regiones. | Un único `<main>` por página; si hay 2+ `<nav>`, cada uno lleva `aria-label` único (ej. `"Navegación principal"`, `"Paginación"`). |
| **`<div onClick>` / `<span onClick>`** | No es enfocable por `Tab`, no responde a `Enter`/`Space` ni anuncia rol de botón. | Usa `<button type="button">` (o `type="submit"` en `<form>`) para acciones y `<a href="...">` para navegación. |
| **Iconos sin semántica correcta** | Iconos decorativos leídos como ruido o botones solo-icono sin nombre accesible (`""`). | Icono decorativo junto a texto -> `aria-hidden="true"`. Botón solo-icono -> `aria-label` descriptivo. |
