# Estándares de Eficiencia en Backend, Datos e Infraestructura

Basado en *Designing Data-Intensive Applications* (Martin Kleppmann) y prácticas de sistemas distribuidos de alto rendimiento.

---

## 1. Consultas, Paginación, Índices y Pool de Conexiones

### Paginar todos los resultados de API
- **Regla:** Ningún endpoint de listado puede devolver una colección sin límite. Exige `limit` acotado en servidor (ej. máx. 100) y prefiere **paginación por cursor (*keyset*)** frente a `OFFSET` alto en tablas grandes.

```typescript
export function normalizePageSize(requested?: number, max = 100, fallback = 20): number {
  if (!requested || !Number.isFinite(requested) || requested < 1) return fallback;
  return Math.min(Math.trunc(requested), max);
}
```

### Arreglar consultas N+1
- **Síntoma:** Un `for` o `.map(async ...)` que ejecuta una query SQL/ORM por cada elemento devuelto por una consulta padre.
- **Solución:** Usar `JOIN` explícito, consulta en lote `WHERE id = ANY($1)` / `IN (...)` o `DataLoader` en GraphQL. Valida con conteo de queries en tests de integración.

```typescript
// Mal: 1 query padre + N queries en loop
for (const order of orders) {
  order.customer = await db.findCustomer(order.customerId);
}

// Bien: 2 queries constantes en total
const customerIds = [...new Set(orders.map((o) => o.customerId))];
const customers = await db.findCustomersByIds(customerIds);
const byId = new Map(customers.map((c) => [c.id, c]));
```

### Crear índices para las consultas más comunes
- **Regla:** Toda consulta frecuente por `WHERE`, `JOIN` u `ORDER BY` (especialmente claves foráneas y cursores de paginación) debe tener índice compuesto alineado al orden de filtrado y ordenamiento (`CREATE INDEX CONCURRENTLY ... ON orders (tenant_id, created_at DESC, id DESC)`).
- **Cuidado:** No indexes todas las columnas indiscriminadamente; cada índice penaliza escrituras (`INSERT`/`UPDATE`). Verifica con `EXPLAIN ANALYZE`.

### Usar un pool de conexiones (*Connection Pooling*)
- **Regla:** Crea **un único pool** en el arranque / *Composition Root* para BD y clientes HTTP (`keep-alive`); jamás abras y cierres una conexión por cada petición HTTP. En entornos *serverless*, usa un pooler externo (PgBouncer / RDS Proxy) y limita `max` conexiones por instancia.

---

## 2. Caché con Redis, Colas y Concurrencia

### Cachear con Redis los datos más consultados
- **Cuándo:** Catálogos, configuraciones, permisos o agregados de lectura intensiva y baja mutación.
- **Patrón seguro (*Cache-Aside*):**
  1. Leer de Redis; si acierta, devolver.
  2. Si falla, consultar BD, guardar en Redis con **TTL + jitter aleatorio** (evita expiración masiva simultánea / *cache stampede*) e **invalidar** al mutar el dato fuente.
  3. **Seguridad:** Incluye `tenantId`/`userId` en la clave si el dato no es público; nunca compartas caché privada entre usuarios.

### Mandar tareas lentas a una cola de trabajo
- **Regla:** Operaciones lentas o propensas a fallos externos (envío de emails, generación de PDFs, procesamiento de imágenes/video, webhooks, embeddings/IA batch) **no** deben bloquear la petición HTTP del usuario.
- **Implementación:** Encola un trabajo en BullMQ/SQS/RabbitMQ, responde `202 Accepted` (o guarda estado `PENDING`), y procesa en workers **idempotentes** con reintentos acotados y *Dead Letter Queue* (DLQ).

### Lanzar en paralelo peticiones independientes
- **Regla:** Nunca encadenes `await` secuenciales si las llamadas no dependen entre sí. Usa `Promise.all` (si todas son requeridas) o `Promise.allSettled` (si algunas son degradables). Para lotes dinámicos grandes, limita la concurrencia (ej. 5-10 a la vez) para no agotar el pool ni golpear rate limits.

---

## 3. Transporte, CDN, Streaming IA e Idempotencia

1. **Activar compresión Brotli o Gzip:**
   Comprime respuestas textuales (`application/json`, `text/html`, CSS, JS, SVG) con Brotli (preferido) o Gzip en el proxy/servidor. Excluye formatos ya comprimidos (AVIF, WebP, PNG, PDF, video) para no desperdiciar CPU.
2. **Servir archivos estáticos desde CDN:**
   Descarga al servidor de aplicaciones sirviendo bundles, avatares, descargas públicas e imágenes optimizadas desde CDN con cache headers largos e invalidación por versión/hash.
3. **Transmitir respuestas de IA token a token desde el backend:**
   Usa `text/event-stream` (SSE) o chunked stream haciendo flush por token, propagando `req.signal` / `AbortController` hacia el proveedor LLM para cortar la generación y ahorrar costos si el usuario cierra la pestaña.
4. **Evitar duplicación de solicitudes en backend (Idempotencia):**
   Mutaciones críticas (`POST /orders`, `POST /payments`) deben aceptar un `Idempotency-Key` almacenado en Redis/BD con TTL para devolver el mismo resultado ante reintentos de red o doble clic del cliente.
