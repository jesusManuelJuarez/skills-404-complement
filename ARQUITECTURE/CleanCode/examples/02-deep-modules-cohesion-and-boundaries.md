# Ejemplo 02 — Módulos Profundos, Fronteras y *The Wrong Abstraction*

Dos casos prácticos en TypeScript que ilustran las decisiones de diseño de **John Ousterhout** (*Deep Modules* & *Information Hiding*), **Robert C. Martin** (*Boundaries*) y **Sandi Metz** (*The Wrong Abstraction*).

---

## Caso A — De *Shallow Wrappers* y fuga de SDK a un *Deep Module*

### Antes: Fuga del proveedor externo y wrappers triviales

```typescript
// stripeClient.ts
export class StripeWrapper {
  constructor(private readonly sdk: StripeSdk) {}

  async createCharge(amount: number, curr: string, tok: string) {
    return this.sdk.charges.create({ amount, currency: curr, source: tok });
  }
}

// checkoutService.ts
export async function chargeCheckout(wrapper: StripeWrapper, cents: number, token: string) {
  try {
    const raw = await wrapper.createCharge(cents, "usd", token);
    if (raw.status !== "succeeded") {
      throw new Error(`Stripe failure: ${raw.failure_code ?? "unknown"}`);
    }
    return { chargeId: raw.id, receiptUrl: raw.receipt_url };
  } catch (err: any) {
    if (err?.raw?.code === "card_declined") {
      return { declined: true, reason: err.raw.decline_code };
    }
    throw err;
  }
}
```

**Problemas:**
- `StripeWrapper` es un *shallow module*: su firma expone casi lo mismo que el SDK (`curr`, `tok`) y devuelve el objeto crudo de Stripe.
- `checkoutService.ts` conoce `err.raw.code === "card_declined"` y `raw.failure_code` (**Leaky Boundary**). Cambiar de pasarela obliga a tocar la lógica de checkout.

### Después: *Deep Module* que oculta complejidad y traduce errores en la frontera

```typescript
// paymentGateway.ts
export interface ChargeRequest {
  readonly amountCents: number;
  readonly currency: "USD" | "EUR";
  readonly paymentToken: string;
  readonly idempotencyKey: string;
}

export type ChargeOutcome =
  | { readonly status: "APPROVED"; readonly transactionId: string }
  | { readonly status: "DECLINED"; readonly reason: "INSUFFICIENT_FUNDS" | "CARD_DECLINED" | "EXPIRED" };

export class PaymentGatewayUnavailableError extends Error {
  constructor(message: string, options?: { cause?: unknown }) {
    super(message, options);
    this.name = "PaymentGatewayUnavailableError";
  }
}

export interface PaymentGateway {
  charge(request: ChargeRequest): Promise<ChargeOutcome>;
}

// stripePaymentGateway.ts
export function createStripePaymentGateway(sdk: StripeSdk): PaymentGateway {
  return {
    async charge(request) {
      if (request.amountCents <= 0) {
        throw new RangeError("Charge amount must be positive.");
      }
      try {
        const response = await sdk.charges.create(
          {
            amount: request.amountCents,
            currency: request.currency.toLowerCase(),
            source: request.paymentToken,
          },
          { idempotencyKey: request.idempotencyKey },
        );

        if (response.status === "succeeded") {
          return { status: "APPROVED", transactionId: response.id };
        }
        return { status: "DECLINED", reason: mapDeclineReason(response.failure_code) };
      } catch (error) {
        const declineCode = extractStripeDeclineCode(error);
        if (declineCode) {
          return { status: "DECLINED", reason: mapDeclineReason(declineCode) };
        }
        throw new PaymentGatewayUnavailableError("Payment provider request failed", { cause: error });
      }
    },
  };
}
```

**Por qué funciona mejor:**
- El consumidor (`checkoutService`) solo conoce `PaymentGateway.charge(...)` y `ChargeOutcome`.
- El mapeo de códigos de Stripe, minúsculas de moneda, `idempotencyKey` y traducción de excepciones quedan ocultos dentro del módulo (*Information Hiding*).

---

## Caso B — Deshacer *The Wrong Abstraction* (Sandi Metz)

### Antes: Dos procesos con distinta razón de cambio forzados en una sola función

```typescript
interface DocumentInput {
  readonly customerType: "B2B" | "B2C";
  readonly netAmountCents: number;
  readonly taxId?: string;
  readonly loyaltyPoints?: number;
}

// Mal: abstracción forzada con flags condicionales
export function generateBillingDocument(input: DocumentInput, includeCreditTerms: boolean) {
  const vatRate = input.customerType === "B2B" ? 0.21 : 0.1;
  const taxCents = Math.round(input.netAmountCents * vatRate);

  if (input.customerType === "B2B") {
    if (!input.taxId) throw new Error("B2B requires taxId");
    return {
      kind: "INVOICE",
      taxId: input.taxId,
      netCents: input.netAmountCents,
      taxCents,
      totalCents: input.netAmountCents + taxCents,
      paymentTermsDays: includeCreditTerms ? 30 : 0,
    };
  }

  return {
    kind: "RECEIPT",
    netCents: input.netAmountCents,
    taxCents,
    totalCents: input.netAmountCents + taxCents,
    earnedPoints: input.loyaltyPoints ?? Math.floor(input.netAmountCents / 100),
  };
}
```

**Problema:** Factura B2B (regulación fiscal, `taxId`, términos de crédito) y Recibo B2C (puntos de fidelidad, consumidor final) evolucionan por **actores y razones distintas**. Compartir una función mediante `customerType` y `includeCreditTerms` acopla ambos flujos.

### Después: Desacoplar los flujos y compartir solo el cálculo verdaderamente común

```typescript
function computeTaxBreakdown(netAmountCents: number, vatRate: number) {
  const taxCents = Math.round(netAmountCents * vatRate);
  return {
    netCents: netAmountCents,
    taxCents,
    totalCents: netAmountCents + taxCents,
  };
}

export function createB2BInvoice(input: {
  readonly taxId: string;
  readonly netAmountCents: number;
  readonly paymentTermsDays: 0 | 30 | 60;
}) {
  if (!input.taxId.trim()) {
    throw new Error("B2B invoice requires a valid taxId");
  }
  return {
    kind: "INVOICE" as const,
    taxId: input.taxId,
    paymentTermsDays: input.paymentTermsDays,
    ...computeTaxBreakdown(input.netAmountCents, 0.21),
  };
}

export function createB2CReceipt(input: {
  readonly netAmountCents: number;
  readonly bonusPoints?: number;
}) {
  const earnedPoints = input.bonusPoints ?? Math.floor(input.netAmountCents / 100);
  return {
    kind: "RECEIPT" as const,
    earnedPoints,
    ...computeTaxBreakdown(input.netAmountCents, 0.1),
  };
}
```

**Lección clave:** *Duplication is far cheaper than the wrong abstraction* (Sandi Metz). Separar ambos contratos eliminó flags opcionales peligrosos (`taxId?`, `includeCreditTerms`) y dio tipos exactos a cada caso.
