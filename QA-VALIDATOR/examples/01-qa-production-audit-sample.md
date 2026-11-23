# Ejemplo 01 — Auditoría Full-Stack con `qa-validator`

Ejemplo de respuesta modelo cuando el desarrollador pide validar una feature antes de pasar a producción.

---

## Entrada del desarrollador

> "Valida con `qa-validator` este endpoint de búsqueda de productos con resumen IA y su componente UI antes de abrir PR."

```typescript
// api/searchProducts.ts
import _ from "lodash";

export async function searchProductsHandler(req: any, res: any, db: any) {
  const { q, tenantId } = req.query;
  const products = await db.query(`SELECT * FROM products WHERE name ILIKE '%${q}%'`);

  for (const p of products) {
    p.category = await db.query("SELECT * FROM categories WHERE id = $1", [p.categoryId]);
  }

  const aiRes = await fetch("https://ai.provider.internal/summarize", {
    method: "POST",
    body: JSON.stringify({ q, count: products.length }),
  });
  const aiData = await aiRes.json();

  return res.json({ tenantId, products, summary: aiData.text });
}
```

```tsx
// ui/ProductSearch.tsx
export function ProductSearch() {
  const [query, setQuery] = useState("");
  const [data, setData] = useState<any[]>([]);

  async function onChange(e: any) {
    setQuery(e.target.value);
    const res = await fetch(`/api/products?q=${e.target.value}`);
    setData(await res.json());
  }

  return (
    <div>
      <input value={query} onChange={onChange} />
      {data.map((p) => (
        <div key={p.id}>
          <img src={`/uploads/${p.id}.png`} />
          <span>{p.name}</span>
        </div>
      ))}
    </div>
  );
}
```

---

## Respuesta Modelo Esperada del LLM

### 1. Veredicto: `BLOCK`
No es seguro ni escalable llevar este cambio a producción: presenta **SQL Injection**, posible **IDOR** por `tenantId` sin autorizar, consulta **N+1** sin paginación y llamada externa a IA sin **timeout** ni **streaming**, además de degradación severa de UX en el cliente.

---

### 2. Hallazgos Priorizados

1. **[Crítico — Seguridad OWASP] SQL Injection y falta de autorización por `tenantId`** (`api/searchProducts.ts:L5-L6`)
   - **Problema:** `q` se interpola directo en el string SQL y `tenantId` viene del query param sin verificar pertenencia (IDOR).
   - **Solución:** Validar `q` y `limit/cursor` con esquema estricto, derivar `tenantId` de la sesión autenticada y usar query parametrizada (`$1`).
2. **[Crítico — Backend/Datos] Listado sin paginación + consulta N+1 + falta de índice** (`api/searchProducts.ts:L6-L10`)
   - **Problema:** `SELECT *` sin `LIMIT` ni paginación por cursor, seguido de un loop con `await db.query(...)` por producto. Además `ILIKE '%...%'` invalida índices B-Tree estándar.
   - **Solución:** Exigir paginación (`limit <= 50` + cursor), resolver categoría en el mismo `JOIN` o consulta `WHERE id = ANY($1)` en lote, y usar índice GIN (`pg_trgm`).
3. **[Crítico — Resiliencia/Observabilidad] API externa de IA sin timeout ni trazabilidad** (`api/searchProducts.ts:L12-L16`)
   - **Problema:** `fetch` sin `AbortSignal.timeout(...)` puede colgar workers indefinidamente y bloquea el catálogo esperando al LLM.
   - **Solución:** Agregar `AbortSignal.timeout(8000)`, instrumentar con span de **OpenTelemetry** (`traceId`, duración) y desacoplar el resumen IA por **streaming token a token (SSE)** en un endpoint dedicado.
4. **[Alto — Frontend] Búsqueda sin debounce, sin cancelación ni estados Loading/Error** (`ui/ProductSearch.tsx:L6-L9`)
   - **Problema:** Cada pulsación dispara un `fetch` (duplicación de solicitudes + *race conditions*) y faltan *skeleton* y manejo de error.
   - **Solución:** Aplicar **debounce (300 ms)** + `AbortController` para cancelar búsquedas previas y renderizar estados *Skeleton*, *Empty* y *Error recuperable*.
5. **[Alto — Frontend] Imágenes PNG sin WebP/AVIF, sin `loading="lazy"` ni dimensiones** (`ui/ProductSearch.tsx:L16`)
   - **Problema:** Carga PNGs completos en el arranque inicial causando alto peso y Layout Shift (CLS).
   - **Solución:** Servir desde **CDN** en formato **AVIF/WebP**, agregar `width`/`height` explícitos y `loading="lazy"` en las tarjetas de resultados.
6. **[Medio — Higiene] Dependencia sin usar (`lodash`) y escala de lista/caché** (`api/searchProducts.ts:L2`)
   - **Problema:** Import muerto de `lodash`.
   - **Solución:** Eliminar `lodash` (`knip`/`depcheck`). Si el listado crece > 100 tarjetas simultáneas, aplicar **virtualización**, activar **Brotli/Gzip** y cachear búsquedas frecuentes en **Redis** con TTL corto por `tenantId`.

---

### 3. Checklist Ejecutado
- **Verificado estáticamente:** Seguridad OWASP, N+1, paginación, timeout, debounce, imágenes WebP/AVIF/lazy, skeletons, streaming IA, dependencias sin usar.
- **Pendiente de verificación en runtime:** `EXPLAIN ANALYZE` sobre la búsqueda con `pg_trgm` y tests de integración probando rechazo 403 ante `tenantId` ajeno.
