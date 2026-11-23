# Ejemplo 02 — Adaptadores Resilientes, Decorador de Caché (OCP/LSP) y Tests en Memoria

Cómo implementar timeouts, OpenTelemetry, paginación por cursor y caché Redis dentro de adaptadores hexagonales sin contaminar el caso de uso, más cómo testear el núcleo en milisegundos.

---

## 1. Adaptador HTTP Secundario con Timeout y OpenTelemetry

```typescript
// infrastructure/gateways/httpRiskScoringAdapter.ts
export interface RiskAssessmentPort {
  evaluateRisk(userId: string, amountCents: number): Promise<"LOW" | "HIGH">;
}

export class ExternalRiskServiceError extends Error {
  constructor(message: string, options?: { cause?: unknown }) {
    super(message, options);
    this.name = "ExternalRiskServiceError";
  }
}

interface TracerSpan {
  setAttribute(key: string, value: string | number | boolean): void;
  recordException(err: unknown): void;
  end(): void;
}
interface Tracer {
  startActiveSpan<T>(name: string, fn: (span: TracerSpan) => Promise<T>): Promise<T>;
}

export function createHttpRiskScoringAdapter(config: {
  readonly baseUrl: string;
  readonly timeoutMs: number;
  readonly tracer: Tracer;
}): RiskAssessmentPort {
  return {
    async evaluateRisk(userId, amountCents) {
      return config.tracer.startActiveSpan("adapter.risk_scoring.evaluate", async (span) => {
        span.setAttribute("risk.amount_cents", amountCents);
        try {
          const response = await fetch(`${config.baseUrl}/v1/risk`, {
            method: "POST",
            headers: { "content-type": "application/json" },
            body: JSON.stringify({ userId, amountCents }),
            signal: AbortSignal.timeout(config.timeoutMs),
          });

          if (!response.ok) {
            throw new ExternalRiskServiceError(`Risk API responded with HTTP ${response.status}`);
          }
          const payload = (await response.json()) as { score: number };
          return payload.score > 70 ? "HIGH" : "LOW";
        } catch (error) {
          span.recordException(error);
          if (error instanceof ExternalRiskServiceError) throw error;
          throw new ExternalRiskServiceError("Risk API request failed or timed out", { cause: error });
        } finally {
          span.end();
        }
      });
    },
  };
}
```

---

## 2. Puerto Paginado + Decorador Redis (*OCP* & *LSP*)

El caso de uso solo ve `ProductCatalogPort`. Podemos envolver el adaptador SQL con un decorador Redis sin tocar el caso de uso ni el repositorio SQL.

```typescript
export interface ProductSummary {
  readonly id: string;
  readonly name: string;
  readonly priceCents: number;
}

export interface ProductPage {
  readonly items: readonly ProductSummary[];
  readonly nextCursor: string | null;
}

export interface ProductCatalogPort {
  listByCategory(query: {
    readonly categoryId: string;
    readonly cursor?: string;
    readonly limit: number;
  }): Promise<ProductPage>;
}

interface RedisLike {
  get(key: string): Promise<string | null>;
  set(key: string, value: string, options: { EX: number }): Promise<void>;
}

export function createCachedProductCatalog(
  inner: ProductCatalogPort,
  redis: RedisLike,
  ttlSeconds = 60,
): ProductCatalogPort {
  return {
    async listByCategory(query) {
      const safeLimit = Math.min(Math.max(query.limit, 1), 100);
      const cacheKey = `catalog:${query.categoryId}:${query.cursor ?? "first"}:${safeLimit}`;

      const cached = await redis.get(cacheKey);
      if (cached) {
        return JSON.parse(cached) as ProductPage;
      }

      const freshPage = await inner.listByCategory({ ...query, limit: safeLimit });
      // Jitter anti-cache-stampede
      const jitter = Math.floor(Math.random() * 10);
      await redis.set(cacheKey, JSON.stringify(freshPage), { EX: ttlSeconds + jitter });
      return freshPage;
    },
  };
}
```

---

## 3. Test del Caso de Uso con *In-Memory Fakes* (Sin BD ni Red)

```typescript
import assert from "node:assert/strict";

async function testSubmitOrderInsufficientStock() {
  const savedOrders: unknown[] = [];
  const queuedEmails: unknown[] = [];

  const submitOrder = createSubmitOrderUseCase({
    catalog: {
      async findUserEmail() {
        return "ana@example.com";
      },
      async findItemsForCheckout() {
        return [{ productId: "sku-1", quantity: 5, unitPriceCents: 1000, availableStock: 2 }];
      },
    },
    payments: {
      async charge() {
        throw new Error("Should not charge when stock is insufficient");
      },
    },
    orders: {
      async save(order) {
        savedOrders.push(order);
        return { id: "ord-1", ...order };
      },
    },
    emailQueue: {
      async enqueueOrderConfirmation(payload) {
        queuedEmails.push(payload);
      },
    },
  });

  const result = await submitOrder({
    userId: "usr-1",
    items: [{ productId: "sku-1", quantity: 5 }],
    paymentToken: "tok_test",
    idempotencyKey: "idem-1",
  });

  assert.deepEqual(result, {
    ok: false,
    error: { code: "INSUFFICIENT_STOCK", productId: "sku-1" },
  });
  assert.equal(savedOrders.length, 0);
  assert.equal(queuedEmails.length, 0);
}
```
