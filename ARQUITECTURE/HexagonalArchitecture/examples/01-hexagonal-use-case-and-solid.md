# Ejemplo 01 — Migración a Arquitectura Hexagonal y SOLID en TypeScript

Caso práctico: pasar de un endpoint HTTP acoplado a Prisma, Stripe y envío síncrono de email a un diseño hexagonal limpio, aplicando **SRP, ISP, DIP** y evitando sobreingeniería.

---

## 1. Código Original (Antes — Acoplado)

```typescript
// Mal: controller acoplado a Prisma, Stripe, N+1 y SMTP síncrono
import { prisma } from "../db";
import Stripe from "stripe";
import nodemailer from "nodemailer";

const stripe = new Stripe(process.env.STRIPE_KEY!);

export async function checkoutController(req: any, res: any) {
  const { userId, items, paymentToken } = req.body;
  const user = await prisma.user.findUnique({ where: { id: userId } });
  if (!user) return res.status(404).json({ error: "User not found" });

  let total = 0;
  for (const item of items) {
    const product = await prisma.product.findUnique({ where: { id: item.productId } }); // N+1
    if (!product || product.stock < item.qty) {
      return res.status(400).json({ error: "Out of stock" });
    }
    total += product.priceCents * item.qty;
  }

  const charge = await stripe.charges.create({ amount: total, currency: "usd", source: paymentToken }); // Sin timeout
  const order = await prisma.order.create({
    data: { userId, totalCents: total, paymentId: charge.id },
  });

  // Bloquea respuesta HTTP esperando SMTP
  const mailer = nodemailer.createTransport({ host: process.env.SMTP_HOST });
  await mailer.sendMail({ to: user.email, subject: "Order confirmed", text: `Order ${order.id}` });

  return res.status(201).json(order);
}
```

---

## 2. Diseño Hexagonal Resultante (Después)

### A. Dominio puro (`domain/order.ts`) — *SRP*
```typescript
export interface PricedOrderItem {
  readonly productId: string;
  readonly quantity: number;
  readonly unitPriceCents: number;
  readonly availableStock: number;
}

export interface Order {
  readonly id: string;
  readonly userId: string;
  readonly totalCents: number;
  readonly paymentId: string;
}

export type CheckoutError =
  | { readonly code: "USER_NOT_FOUND" }
  | { readonly code: "EMPTY_CART" }
  | { readonly code: "INSUFFICIENT_STOCK"; readonly productId: string }
  | { readonly code: "PAYMENT_DECLINED"; readonly reason: string };

export type Result<T, E> =
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: E };

export function computeCheckoutTotalCents(
  items: readonly PricedOrderItem[],
): Result<number, CheckoutError> {
  if (items.length === 0) {
    return { ok: false, error: { code: "EMPTY_CART" } };
  }
  let totalCents = 0;
  for (const item of items) {
    if (item.quantity <= 0 || item.availableStock < item.quantity) {
      return { ok: false, error: { code: "INSUFFICIENT_STOCK", productId: item.productId } };
    }
    totalCents += item.unitPriceCents * item.quantity;
  }
  return { ok: true, value: totalCents };
}
```

### B. Puertos pequeños orientados al caso de uso (`application/ports.ts`) — *ISP + DIP*
```typescript
import type { Order, PricedOrderItem, Result } from "../domain/order";

export interface CheckoutCatalogReader {
  // Lectura en lote para evitar N+1
  findItemsForCheckout(
    requested: readonly { readonly productId: string; readonly quantity: number }[],
  ): Promise<readonly PricedOrderItem[]>;
  findUserEmail(userId: string): Promise<string | null>;
}

export interface PaymentCharger {
  charge(input: {
    readonly amountCents: number;
    readonly paymentToken: string;
    readonly idempotencyKey: string;
  }): Promise<Result<{ readonly paymentId: string }, { readonly reason: string }>>;
}

export interface OrderSaver {
  save(order: Omit<Order, "id">): Promise<Order>;
}

export interface OrderEmailQueue {
  enqueueOrderConfirmation(payload: { readonly email: string; readonly orderId: string }): Promise<void>;
}
```

### C. Caso de Uso (`application/submitOrderUseCase.ts`) — *Sin frameworks*
```typescript
import { computeCheckoutTotalCents, type CheckoutError, type Order, type Result } from "../domain/order";
import type { CheckoutCatalogReader, OrderEmailQueue, OrderSaver, PaymentCharger } from "./ports";

export interface SubmitOrderCommand {
  readonly userId: string;
  readonly items: readonly { readonly productId: string; readonly quantity: number }[];
  readonly paymentToken: string;
  readonly idempotencyKey: string;
}

export interface SubmitOrderDeps {
  readonly catalog: CheckoutCatalogReader;
  readonly payments: PaymentCharger;
  readonly orders: OrderSaver;
  readonly emailQueue: OrderEmailQueue;
}

export function createSubmitOrderUseCase(deps: SubmitOrderDeps) {
  return async function submitOrder(cmd: SubmitOrderCommand): Promise<Result<Order, CheckoutError>> {
    const [email, pricedItems] = await Promise.all([
      deps.catalog.findUserEmail(cmd.userId),
      deps.catalog.findItemsForCheckout(cmd.items),
    ]);

    if (!email) {
      return { ok: false, error: { code: "USER_NOT_FOUND" } };
    }

    const totalResult = computeCheckoutTotalCents(pricedItems);
    if (!totalResult.ok) {
      return totalResult;
    }

    const paymentResult = await deps.payments.charge({
      amountCents: totalResult.value,
      paymentToken: cmd.paymentToken,
      idempotencyKey: cmd.idempotencyKey,
    });
    if (!paymentResult.ok) {
      return { ok: false, error: { code: "PAYMENT_DECLINED", reason: paymentResult.error.reason } };
    }

    const savedOrder = await deps.orders.save({
      userId: cmd.userId,
      totalCents: totalResult.value,
      paymentId: paymentResult.value.paymentId,
    });

    await deps.emailQueue.enqueueOrderConfirmation({ email, orderId: savedOrder.id });
    return { ok: true, value: savedOrder };
  };
}
```

### D. *Composition Root* (`compositionRoot.ts`) — *Mark Seemann*
```typescript
import { createSubmitOrderUseCase } from "./application/submitOrderUseCase";
import { createPrismaCatalogReader, createPrismaOrderSaver } from "./infrastructure/persistence/prismaOrders";
import { createStripePaymentCharger } from "./infrastructure/gateways/stripePaymentCharger";
import { createBullEmailQueue } from "./infrastructure/gateways/bullEmailQueue";

export interface PrismaClientLike {
  readonly product: { findMany(args: unknown): Promise<unknown[]> };
  readonly user: { findUnique(args: unknown): Promise<{ email: string } | null> };
  readonly order: { create(args: unknown): Promise<Order> };
}

export interface StripeClientLike {
  readonly charges: { create(params: unknown): Promise<{ id: string }> };
}

export interface QueueClientLike {
  add(name: string, payload: unknown): Promise<unknown>;
}

export interface InfraClients {
  readonly prisma: PrismaClientLike;
  readonly stripe: StripeClientLike;
  readonly queue: QueueClientLike;
}

export function buildCheckoutHandler(deps: InfraClients) {
  const submitOrder = createSubmitOrderUseCase({
    catalog: createPrismaCatalogReader(deps.prisma),
    orders: createPrismaOrderSaver(deps.prisma),
    payments: createStripePaymentCharger(deps.stripe),
    emailQueue: createBullEmailQueue(deps.queue),
  });

  return { submitOrder };
}
```

---

## 3. Qué se decidió NO hacer (Anti-Sobreingeniería)
1. **No creamos clases `OrderEntity` con getters/setters vacíos ni `AbstractBaseUseCase`:** funciones fábrica y tipos inmutables bastan en TypeScript.
2. **No creamos un DTO espejo por cada capa:** `SubmitOrderCommand` y `Order` viajan limpios sin transformaciones inútiles.
3. **Resolvimos N+1 y bloqueo SMTP por diseño del puerto:** `findItemsForCheckout` fuerza lectura en lote y `OrderEmailQueue` saca el SMTP de la respuesta HTTP.
