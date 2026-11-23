# Estándares de Eficiencia en Frontend y UX Percibida

Basado en **Web.dev Core Web Vitals (LCP, INP, CLS)** y patrones modernos de aplicaciones web.

---

## 1. Medios, Imágenes y Activos Estáticos

| Regla | Qué exigir / implementar | Error frecuente a evitar |
| :--- | :--- | :--- |
| **Servir imágenes en WebP / AVIF** | Usa `<picture>` con fallback o pipeline de CDN/framework (`next/image`, Sharp, Cloudinary) sirviendo AVIF primero y WebP como fallback. Define siempre `width` y `height` (o `aspect-ratio`) para CLS = 0. | Subir PNG/JPEG de 3 MB directo desde el repo sin conversión ni dimensiones explícitas. |
| **Lazy loading en imágenes fuera de pantalla** | Añade `loading="lazy"` y `decoding="async"` en imágenes *below-the-fold* (tarjetas, galerías, footer). | Poner `loading="lazy"` en la imagen principal (*hero* / LCP): retrasa el LCP. Ahí usa `loading="eager"` y `fetchpriority="high"`. |
| **Servir activos estáticos desde CDN** | Sirve JS, CSS, fuentes e imágenes versionadas por hash desde una CDN con `Cache-Control: public, max-age=31536000, immutable`. | Servir assets estáticos pesados desde el mismo proceso Node/API sin cache headers. |

---

## 2. Red, Búsquedas y Estado Reactivo

### Debounce en búsquedas + cancelación de peticiones
- **Regla:** Aplica un debounce de **250–400 ms** en inputs de búsqueda conectados a APIs y **cancela la petición anterior** con `AbortController` para evitar *race conditions* (que una respuesta vieja pise a una nueva).

```typescript
function createDebouncedSearch(
  searchFn: (query: string, signal: AbortSignal) => Promise<void>,
  delayMs = 300,
) {
  let timer: ReturnType<typeof setTimeout> | undefined;
  let activeController: AbortController | undefined;

  return (query: string) => {
    clearTimeout(timer);
    activeController?.abort();

    const trimmed = query.trim();
    if (!trimmed) return;

    timer = setTimeout(() => {
      activeController = new AbortController();
      void searchFn(trimmed, activeController.signal).catch((err) => {
        if ((err as Error).name !== "AbortError") throw err;
      });
    }, delayMs);
  };
}
```

### Evitar duplicación de solicitudes en cliente
- **Regla:** Deduplica peticiones idénticas en vuelo por clave (ej. usando TanStack Query / SWR o un mapa `inFlightRequests` en memoria), deshabilita botones durante mutaciones en curso y usa `Promise.all` para cargar datos independientes de una pantalla en paralelo en lugar de en cascada (*waterfall*).

### Actualizar interfaz sin esperar al servidor (*Optimistic UI*)
- **Cuándo usar:** Acciones frecuentes de baja fricción donde el usuario espera respuesta instantánea (*likes*, marcar tarea completada, reordenar, agregar al carrito).
- **Regla obligatoria:** Guarda una instantánea del estado previo y ejecuta **rollback + feedback de error** si el servidor rechaza la petición. **No** lo uses en pagos finales o acciones destructivas irreversibles.

```typescript
async function toggleFavoriteOptimistic(
  itemId: string,
  currentState: boolean,
  applyUi: (next: boolean) => void,
  persist: (next: boolean) => Promise<void>,
  notifyError: (msg: string) => void,
) {
  const nextState = !currentState;
  applyUi(nextState);
  try {
    await persist(nextState);
  } catch {
    // Rollback exacto ante fallo de servidor
    applyUi(currentState);
    notifyError("No pudimos guardar el cambio. Intenta nuevamente.");
  }
}
```

---

## 3. Carga Percibida, Streaming de IA y Renderizado

1. **Skeletons de carga:**
   Usa *skeletons* con la misma altura/estructura aproximada del contenido real para evitar saltos visuales (CLS) y reducir espera percibida frente a un spinner genérico centrado.
2. **Transmitir respuestas de IA token a token (*Streaming*):**
   Consume respuestas LLM mediante SSE o `ReadableStream` (`fetch` + `response.body.getReader()`) renderizando incrementalmente. Incluye siempre un botón "Detener" conectado a `AbortController` y maneja errores parciales sin borrar lo ya recibido.
3. **Renderizar en el servidor o en build (SSR / SSG / ISR):**
   - **SSG / Build-time:** Marketing, docs, catálogos estables.
   - **ISR / SSR:** Páginas dinámicas con SEO o datos frescos en el primer paint (evita enviar un `<div id="root"></div>` vacío que obliga a descargar JS antes de pedir datos).
   - **CSR:** Dashboards privados altamente interactivos tras autenticación.

---

## 4. JavaScript, Terceros y Listas Grandes

1. **Dividir el JavaScript por página (*Code Splitting*):**
   Usa `import()` dinámico por ruta y carga diferida para librerías pesadas (editores ricos, gráficas, visores PDF, mapas) que no se ven en el primer pantallazo.
2. **Cargar servicios de terceros después de la página:**
   Scripts de analytics, chat de soporte, mapas o píxeles deben cargarse de forma diferida (`defer`, `requestIdleCallback`, o tras hidratación/interacción) para no bloquear el hilo principal ni degradar INP/LCP.
3. **Virtualizar listas largas:**
   Si una tabla, feed o selector renderiza más de **100–200 ítems** complejos simultáneamente en el DOM, usa virtualización (*windowing*) manteniendo altura consistente, encabezados accesibles y navegación por teclado.
4. **Quitar dependencias sin usar:**
   Revisa con herramientas (`knip`, `depcheck`, análisis de bundle) que no queden librerías pesadas o duplicadas (`moment` + `dayjs`, `lodash` entero por una función) inflando el bundle enviado al navegador.
