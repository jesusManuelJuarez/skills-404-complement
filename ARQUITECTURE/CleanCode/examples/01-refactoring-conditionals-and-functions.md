# Ejemplo 01 — Refactorización de Condicionales, Funciones y Tipos en TypeScript

Caso práctico completo: cómo transformar una función de procesamiento de pedidos con anidamiento profundo, *flag arguments*, *primitive obsession* y mezcla de I/O en código limpio, testeable y sin sobreingeniería.

---

## 1. Código Original (Antes)

```typescript
// Código original con múltiples code smells
interface Db {
  saveOrder(order: any): Promise<void>;
}
interface Mailer {
  send(to: string, subject: string, body: string): Promise<void>;
}

export async function processOrder(
  user: any,
  items: any[],
  isVip: boolean,
  sendEmail: boolean,
  couponCode: string | null,
  db: Db,
  mailer: Mailer,
) {
  let total = 0;
  if (user) {
    if (user.active) {
      if (items && items.length > 0) {
        for (let i = 0; i < items.length; i++) {
          if (items[i].qty > 0 && items[i].price >= 0) {
            total += items[i].qty * items[i].price;
          } else {
            return { ok: false, err: "INVALID_ITEM" };
          }
        }
        if (isVip) {
          total = total * 0.85;
        } else {
          if (couponCode === "WELCOME10" && total >= 50) {
            total = total * 0.9;
          }
        }
        const order = {
          userId: user.id,
          total: Math.round(total * 100) / 100,
          status: "CREATED",
        };
        try {
          await db.saveOrder(order);
          if (sendEmail) {
            await mailer.send(user.email, "Order created", `Total: ${order.total}`);
          }
          return { ok: true, order };
        } catch (e) {
          return { ok: false, err: "ERR_500" };
        }
      } else {
        return { ok: false, err: "EMPTY_ITEMS" };
      }
    } else {
      return { ok: false, err: "USER_INACTIVE" };
    }
  } else {
    return { ok: false, err: "NO_USER" };
  }
}
```

---

## 2. Diagnóstico de *Code Smells*

1. **[Crítico] Mezcla de validación, cálculo de negocio y efectos secundarios (I/O) (`Split Phase` pendiente — Fowler):**
   No se puede testear el cálculo de descuentos sin mockear `db` y `mailer`.
2. **[Crítico] `catch (e)` genérico que traga el error original (Hunt & Thomas / Martin):**
   Devuelve `"ERR_500"` perdiendo la causa real y enmascarando fallos de red/BD vs. errores de dominio.
3. **[Alto] Anidamiento de 5 niveles (`Deep Nesting` — Martin / Fowler):**
   Los `else` están a 30 líneas de distancia de su `if`.
4. **[Alto] `any`, *Primitive Obsession* y dinero en punto flotante:**
   `user: any`, `items: any[]`, `price` flotante (`Math.round(total * 100) / 100`) y códigos de error sueltos.
5. **[Medio] *Long Parameter List* (7 parámetros) + *Flag Arguments* (`isVip`, `sendEmail`):**
   El estado VIP pertenece al usuario/tier o a una política explícita; `sendEmail` acopla la notificación al cálculo.

---

## 3. Plan de Refactor en Pasos Atómicos

1. Definir tipos de dominio estrictos (`OrderUser`, `OrderItem`, `OrderError`, `Result`) trabajando en **centavos enteros** (`Cents`).
2. Extraer la lógica pura a `calculateOrderQuote(...)` (*Split Phase*) usando **Guard Clauses** para aplanar el flujo.
3. Dejar `placeOrder(...)` como orquestador delgado en un único nivel de abstracción (*SLAP*), preservando la causa de error en fallos de infraestructura.

---

## 4. Código Refactorizado (Después)

```typescript
export type CustomerTier = "STANDARD" | "VIP";

export interface OrderUser {
  readonly id: string;
  readonly email: string;
  readonly isActive: boolean;
  readonly tier: CustomerTier;
}

export interface OrderItem {
  readonly sku: string;
  readonly quantity: number;
  readonly unitPriceCents: number;
}

export type OrderValidationError =
  | { readonly code: "USER_INACTIVE" }
  | { readonly code: "EMPTY_ITEMS" }
  | { readonly code: "INVALID_ITEM"; readonly sku: string };

export type Result<T, E> =
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: E };

export interface CreatedOrder {
  readonly userId: string;
  readonly totalCents: number;
  readonly status: "CREATED";
}

export interface OrderRepository {
  saveOrder(order: CreatedOrder): Promise<void>;
}

export interface OrderNotifier {
  sendOrderCreated(email: string, totalCents: number): Promise<void>;
}

export interface PlaceOrderDeps {
  readonly repository: OrderRepository;
  readonly notifier?: OrderNotifier;
}

const VIP_DISCOUNT_RATE = 0.15;
const WELCOME_COUPON_CODE = "WELCOME10";
const WELCOME_MIN_SUBTOTAL_CENTS = 5_000;
const WELCOME_DISCOUNT_RATE = 0.1;

export function calculateOrderTotalCents(
  user: OrderUser,
  items: readonly OrderItem[],
  couponCode: string | null,
): Result<number, OrderValidationError> {
  if (!user.isActive) {
    return { ok: false, error: { code: "USER_INACTIVE" } };
  }
  if (items.length === 0) {
    return { ok: false, error: { code: "EMPTY_ITEMS" } };
  }

  let subtotalCents = 0;
  for (const item of items) {
    if (item.quantity <= 0 || item.unitPriceCents < 0) {
      return { ok: false, error: { code: "INVALID_ITEM", sku: item.sku } };
    }
    subtotalCents += item.quantity * item.unitPriceCents;
  }

  const discountCents = calculateDiscountCents(subtotalCents, user.tier, couponCode);
  return { ok: true, value: subtotalCents - discountCents };
}

function calculateDiscountCents(
  subtotalCents: number,
  tier: CustomerTier,
  couponCode: string | null,
): number {
  if (tier === "VIP") {
    return Math.round(subtotalCents * VIP_DISCOUNT_RATE);
  }
  if (couponCode === WELCOME_COUPON_CODE && subtotalCents >= WELCOME_MIN_SUBTOTAL_CENTS) {
    return Math.round(subtotalCents * WELCOME_DISCOUNT_RATE);
  }
  return 0;
}

export async function placeOrder(
  input: {
    readonly user: OrderUser;
    readonly items: readonly OrderItem[];
    readonly couponCode: string | null;
  },
  deps: PlaceOrderDeps,
): Promise<Result<CreatedOrder, OrderValidationError>> {
  const totalResult = calculateOrderTotalCents(input.user, input.items, input.couponCode);
  if (!totalResult.ok) {
    return totalResult;
  }

  const order: CreatedOrder = {
    userId: input.user.id,
    totalCents: totalResult.value,
    status: "CREATED",
  };

  await deps.repository.saveOrder(order);
  await deps.notifier?.sendOrderCreated(input.user.email, order.totalCents);

  return { ok: true, value: order };
}
```

---

## 5. Qué decidimos NO abstraer (y por qué)

1. **No creamos un patrón `DiscountStrategy` / `DiscountFactory`:**
   Solo hay dos reglas de descuento simples y estables (`VIP` y `WELCOME10`). Crear 3 clases e interfaces ahora sería *Speculative Generality*. Si mañana llegan 5 tipos de cupones con reglas complejas, `calculateDiscountCents` ya está aislado y listo para evolucionar a un `Record<CouponKind, ...>`.
2. **No extrajimos una función `validateItems` de 4 líneas:**
   Sumar el subtotal mientras validas cada ítem evita recorrer dos veces el arreglo y mantiene la lectura lineal sin *shallow functions*.
3. **No convertimos `placeOrder` en una clase `OrderProcessorService`:**
   No hay estado interno que encapsular; funciones cohesivas con dependencias explícitas en `PlaceOrderDeps` son más simples y directas en TypeScript.
