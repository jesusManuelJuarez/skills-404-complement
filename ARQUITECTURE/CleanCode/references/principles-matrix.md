# Matriz Operativa de Principios — Clean Code & Software Design

Guía de referencia rápida con reglas accionables, micro-ejemplos en TypeScript y excepciones prácticas.

---

## 1. Nombrado y Vocabulario del Dominio
*Referentes: Robert C. Martin (Clean Code), John Ousterhout (A Philosophy of Software Design).*

| Regla | Síntoma a evitar | Criterio operativo |
| :--- | :--- | :--- |
| **Nombres que revelan intención** | `d`, `list1`, `handle(x)` | El nombre debe responder *qué es*, *por qué existe* y *cómo se usa* sin leer la implementación. |
| **Evitar palabras ruido** | `userData`, `orderInfo`, `AuthManager`, `DateHelper`, `Utils` | Si un sufijo no aporta significado (`Data`, `Info`, `Object`, `Manager`, `Helper`), elimínalo o usa el concepto de dominio real (`InvoiceCalculator`, `SessionStore`). |
| **Vocabulario consistente (1 concepto = 1 palabra)** | Mezclar `fetchUser`, `getClient`, `retrieveAccount` para la misma acción | Elige un verbo por operación en todo el módulo (`find*` cuando puede ser `undefined`, `get*` cuando lanza o garantiza valor, `create*`, `calculate*`). |
| **Booleanos como predicados claros** | `flag`, `status`, `notDisabled` | Prefija con `is`, `has`, `can`, `should` y evita negaciones dobles (`isEnabled` mejor que `isNotDisabled`). |
| **Simetría en pares** | `open/stop`, `add/erase` | Usa pares naturales: `open/close`, `begin/end`, `create/delete`, `encode/decode`. |
| **Longitud proporcional al alcance** | Variables locales de 35 caracteres; globales de 2 letras | Variables de bucle o lambda muy acotadas pueden ser cortas (`item`, `idx`); funciones públicas y módulos deben ser explícitos. |

```typescript
// Mal: ruido, abreviaturas y booleano ambiguo
function proc(d: { flg: boolean; amt: number }[]): number {
  return d.filter((x) => !x.flg).reduce((a, b) => a + b.amt, 0);
}

// Bien: vocabulario de dominio e intención inmediata
interface InvoiceLine {
  readonly isVoided: boolean;
  readonly amountCents: number;
}

function sumActiveInvoiceLines(lines: readonly InvoiceLine[]): number {
  return lines
    .filter((line) => !line.isVoided)
    .reduce((total, line) => total + line.amountCents, 0);
}
```

**Excepción:** No renombres términos de un contrato público o esquema de base de datos sin verificar impacto en serialización, APIs o consumidores externos.

---

## 2. Funciones y Flujo de Control
*Referentes: Robert C. Martin, Martin Fowler, Hunt & Thomas.*

| Regla | Criterio operativo | Excepción / Límite |
| :--- | :--- | :--- |
| **SLAP (Single Level of Abstraction Principle)** | Una función orquestadora llama pasos de alto nivel (`validate`, `calculate`, `persist`), no mezcla SQL crudo con reglas de negocio en el mismo bloque. | Scripts muy simples de < 15 líneas sin bifurcaciones no necesitan subdivisión artificial. |
| **CQS (Command-Query Separation)** | Una función o bien **consulta** (devuelve valor sin efectos secundarios) o bien **ejecuta un comando** (muta estado / hace I/O). | Patrones tipo `pop()` de pila o `Map.getOrCreate` donde la atomicidad es el objetivo. |
| **Guard Clauses y anidamiento ≤ 2** | Valida precondiciones y casos inválidos al inicio con `return` o error temprano; deja el camino feliz al nivel base. | Cuando una tabla de decisión o `switch` exhaustivo sobre un discriminante es más legible. |
| **Máximo 3 parámetros** | Si una función recibe 4+ parámetros o varios del mismo tipo (`string, string, string`), agrúpalos en un *Parameter Object* con nombre de dominio. | Constructores de Value Objects simples o callbacks impuestos por una librería. |
| **Prohibido *Flag Arguments* booleanos** | `exportReport(data, true)` oculta qué hace `true`. Separa en funciones explícitas (`exportCsvReport` / `exportPdfReport`) o usa un objeto de opciones con tipo unión. | Solo aceptable cuando el booleano se reenvía directamente a un componente UI/configuración declarativa (`disabled: boolean`). |
| **Separar cálculo puro de I/O** | Extrae las reglas de negocio a funciones puras deterministas; deja BD, HTTP, reloj (`Date.now`) y logs en los extremos. | Ninguna en lógica de dominio: separar pureza de I/O multiplica la testabilidad sin mocks pesados. |

```typescript
// Mal: flag argument + anidamiento profundo + I/O mezclado
async function publishArticle(id: string, isDraft: boolean, notify: boolean) {
  const article = await repo.find(id);
  if (article) {
    if (!isDraft) {
      if (article.body.length > 0) {
        article.status = "PUBLISHED";
        await repo.save(article);
        if (notify) await mailer.send(article.authorEmail);
      }
    }
  }
}

// Bien: guard clauses y flujo plano
async function publishArticle(articleId: string, options: { notifyAuthor: boolean }): Promise<void> {
  const article = await repo.findById(articleId);
  if (!article) throw new ArticleNotFoundError(articleId);
  if (article.body.trim().length === 0) throw new EmptyArticleError(articleId);

  const published = markAsPublished(article);
  await repo.save(published);

  if (options.notifyAuthor) {
    await mailer.notifyPublication(published.authorEmail, published.title);
  }
}
```

---

## 3. Módulos, Clases y Cohesión
*Referentes: John Ousterhout, Sandi Metz, Robert C. Martin, Hunt & Thomas.*

| Regla | Criterio operativo | Excepción / Límite |
| :--- | :--- | :--- |
| **Deep Modules (Ousterhout)** | Busca módulos con **interfaz simple y mucha funcionalidad oculta** dentro. Evita *shallow modules* cuya firma es casi tan compleja como su cuerpo. | Adaptadores muy finos exigidos por un framework o puerto explícito. |
| **Information Hiding** | Oculta detalles de almacenamiento, caché, formato de red o algoritmo interno. Nadie fuera del módulo debe conocer cómo representa sus datos internos. | DTOs de transferencia puros en fronteras HTTP/DB. |
| **Tell, Don't Ask / Ley de Demeter** | En lugar de encadenar `order.getCustomer().getAddress().getCountry()`, pide al objeto dueño de los datos la operación (`order.getShippingCountry()`). | Estructuras de datos puras (DTOs, ASTs, configuraciones) que no encapsulan comportamiento. |
| **SRP por actor / razón de cambio** | Agrupa código que cambia por la misma razón o para el mismo actor de negocio; separa lo que cambia por motivos distintos (reglas fiscales vs. formato de email). | Prototipos desechables o módulos triviales de una sola responsabilidad. |
| **Composición sobre herencia** | Prefiere componer funciones/colaboradores pequeños antes que jerarquías de clases de más de 2 niveles. | Cuando un framework base exige extender una clase específica. |
| **DRY de conocimiento (Regla de Tres)** | Elimina duplicación de **reglas de negocio** y algoritmos estables. Si dos bloques coinciden hoy por casualidad pero pertenecen a procesos distintos, **no los unas todavía** (Sandi Metz). | Constantes o contratos compartidos críticos (ej. fórmula regulatoria) se unifican desde el primer uso. |

---

## 4. Manejo de Errores y Fronteras (*Boundaries*)
*Referentes: Robert C. Martin, John Ousterhout, Hunt & Thomas.*

1. ***Define Errors Out of Existence* (Ousterhout):** Antes de lanzar una excepción por un caso trivial, pregúntate si la semántica normal puede absorberlo sin riesgo (ej. `removeTag(tag)` sobre un elemento sin ese tag puede ser un *no-op* idempotente en lugar de fallar).
2. ***Fail Fast* en fronteras (Hunt & Thomas):** Valida entradas externas (HTTP, JSON, env vars, archivos) en la frontera del sistema con esquemas explícitos; dentro del núcleo asume tipos válidos.
3. **Errores de dominio tipados o patrón `Result`:**
   - Usa **excepciones de dominio** para fallos excepcionales o ruptura de invariantes.
   - Usa un tipo **`Result<T, E>`** (unión discriminada) para fallos esperables de negocio donde el llamador debe decidir explícitamente qué hacer (ej. saldo insuficiente, cupón expirado).
4. **Prohibido `catch` silencioso o pérdida de contexto:** Nunca hagas `catch (e) {}` ni `catch (e) { return null }` sin registrar contexto o re-lanzar un error envuelto (`new PaymentGatewayError("...", { cause: error })`).
5. **Envuelve SDKs y APIs de terceros (*Wrap Boundaries*):** Ningún módulo de dominio debería importar directamente tipos o códigos de error crudos de Stripe, Axios, Prisma o AWS. Crea un adaptador/wrapper que traduzca a tipos y errores propios.

```typescript
type CouponError = { readonly code: "EXPIRED" | "MIN_AMOUNT_NOT_MET"; readonly message: string };
type Result<T, E> = { readonly ok: true; readonly value: T } | { readonly ok: false; readonly error: E };

function applyCoupon(subtotalCents: number, coupon: Coupon, now: Date): Result<number, CouponError> {
  if (coupon.expiresAt < now) {
    return { ok: false, error: { code: "EXPIRED", message: "El cupón ha expirado." } };
  }
  if (subtotalCents < coupon.minSubtotalCents) {
    return { ok: false, error: { code: "MIN_AMOUNT_NOT_MET", message: "Subtotal insuficiente." } };
  }
  return { ok: true, value: Math.max(0, subtotalCents - coupon.discountCents) };
}
```

---

## 5. Comentarios Útiles vs. Ruido
*Referentes: Robert C. Martin, John Ousterhout.*

- **Regla de formato:** Si comentas, hazlo breve (máx. 1 línea), sin emojis y solo para el *por qué* o una invariante no evidente.
- **Elimina comentarios que narran el *qué*:**
  `// Incrementar contador en 1` o `// Verificar si el usuario está activo` → si hace falta explicar *qué* hace un bloque, extrae una función o mejora los nombres.
- **Elimina código comentado y logs de historial:** Git ya guarda el historial; el código comentado solo pudre y confunde.
- **Escribe comentarios que expliquen el *por qué*, invariantes o advertencias:**
  - Decisiones no obvias y *trade-offs* (*"Se ordena en memoria porque la API externa no soporta sorting compuesto; máx. 200 items"*).
  - Restricciones externas o bugs de terceros (*"Workaround: la pasarela retorna 200 con body vacío cuando..."*).
  - JSDoc/TSDoc conciso en **interfaces públicas y módulos exportados** describiendo contrato, unidades (`amountCents`) y errores posibles.
