# Ejemplo 01 — Refactorización UI: Antes y Después (React TSX + CSS)

Tarjeta de suscripción editable dentro de una lista. El ejemplo original concentra los errores reales más frecuentes en interfaces generadas sin reglas estrictas.

---

## 1. Código Original (Antes)

```tsx
// Mal: sin semántica, id duplicado y radios/targets rotos
export function SubscriptionCards({ items, onSave, onRemove }: any) {
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: "13px" }}>
      {items.map((item: any) => (
        <div
          key={item.id}
          style={{ borderRadius: "12px", padding: "19px", border: "1px solid #ddd" }}
        >
          <div style={{ display: "flex", justifyContent: "space-between" }}>
            <h4 style={{ textAlign: "center" }}>{item.planName}</h4>
            <div onClick={() => onRemove(item.id)} style={{ width: 16, height: 16 }}>
              &times;
            </div>
          </div>

          <p style={{ textAlign: "center", color: "#aaa" }}>{item.description}</p>
          <div style={{ fontSize: 20 }}>${item.price}</div>

          <input
            id="billing-email"
            placeholder="Email de facturación"
            defaultValue={item.email}
            style={{ fontSize: "12px", borderRadius: "14px", padding: "5px" }}
          />

          <div style={{ display: "flex", gap: "4px", marginTop: "11px" }}>
            <button style={{ background: "#2563eb", color: "#fff" }}>Cancelar</button>
            <button style={{ background: "#2563eb", color: "#fff" }} onClick={() => onSave(item.id)}>
              Aceptar
            </button>
          </div>
        </div>
      ))}
    </div>
  );
}
```

---

## 2. Código Refactorizado (Después)

```tsx
// SubscriptionCards.tsx
import { useId, useState, type FormEvent } from "react";
import "./SubscriptionCards.css";

export interface SubscriptionItem {
  readonly id: string;
  readonly planName: string;
  readonly description: string;
  readonly monthlyPriceFormatted: string;
  readonly billingEmail: string;
}

interface SubscriptionCardsProps {
  readonly items: readonly SubscriptionItem[];
  readonly onSaveEmail: (id: string, nextEmail: string) => Promise<void>;
  readonly onRemove: (id: string) => void;
}

export function SubscriptionCards({ items, onSaveEmail, onRemove }: SubscriptionCardsProps) {
  return (
    <ul className="sub-list">
      {items.map((item) => (
        <li key={item.id}>
          <SubscriptionCard item={item} onSaveEmail={onSaveEmail} onRemove={onRemove} />
        </li>
      ))}
    </ul>
  );
}

function SubscriptionCard({
  item,
  onSaveEmail,
  onRemove,
}: {
  readonly item: SubscriptionItem;
  readonly onSaveEmail: (id: string, nextEmail: string) => Promise<void>;
  readonly onRemove: (id: string) => void;
}) {
  // IDs únicos por instancia en la lista
  const emailInputId = useId();
  const errorId = useId();

  const [email, setEmail] = useState(item.billingEmail);
  const [error, setError] = useState<string | null>(null);
  const [isSaving, setIsSaving] = useState(false);

  async function handleSubmit(event: FormEvent<HTMLFormElement>) {
    event.preventDefault();
    const trimmed = email.trim();
    if (!trimmed.includes("@")) {
      setError("Ingresa un correo válido, por ejemplo facturacion@example.com.");
      return;
    }
    setError(null);
    setIsSaving(true);
    try {
      await onSaveEmail(item.id, trimmed);
    } finally {
      setIsSaving(false);
    }
  }

  return (
    <article className="sub-card" aria-labelledby={`plan-${item.id}`}>
      <header className="sub-card__header">
        <div className="sub-card__title-group">
          <h2 id={`plan-${item.id}`} className="sub-card__title">
            {item.planName}
          </h2>
          <span className="sub-card__price">{item.monthlyPriceFormatted}</span>
        </div>

        <button
          type="button"
          className="sub-btn sub-btn--icon"
          aria-label={`Eliminar suscripción ${item.planName}`}
          onClick={() => onRemove(item.id)}
        >
          <span aria-hidden="true">&times;</span>
        </button>
      </header>

      <p className="sub-card__desc">{item.description}</p>

      <form className="sub-card__form" onSubmit={handleSubmit} noValidate>
        <div className="sub-field">
          <label htmlFor={emailInputId} className="sub-field__label">
            Correo de facturación
          </label>
          <input
            id={emailInputId}
            type="email"
            className="sub-field__input"
            value={email}
            placeholder="facturacion@example.com"
            aria-invalid={error ? "true" : undefined}
            aria-describedby={error ? errorId : undefined}
            onChange={(e) => setEmail(e.target.value)}
          />
          {error && (
            <p id={errorId} className="sub-field__error" role="alert">
              {error}
            </p>
          )}
        </div>

        <div className="sub-card__actions">
          <button
            type="button"
            className="sub-btn sub-btn--secondary"
            onClick={() => {
              setEmail(item.billingEmail);
              setError(null);
            }}
          >
            Restablecer
          </button>
          <button type="submit" className="sub-btn sub-btn--primary" aria-busy={isSaving}>
            <span>{isSaving ? "Guardando..." : "Guardar correo"}</span>
          </button>
        </div>
      </form>
    </article>
  );
}
```

```css
/* SubscriptionCards.css */
.sub-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
  list-style: none;
  padding: 0;
  margin: 0;
}

.sub-card {
  /* R_inner <= R_outer (16px) - padding (16px) */
  border-radius: 16px;
  padding: 16px;
  border: 1px solid var(--border-subtle, #cbd5e1);
  background: var(--surface-card, #ffffff);
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.sub-card__header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
}

.sub-card__title-group {
  display: flex;
  flex-direction: column;
  gap: 4px;
  min-width: 0;
}

.sub-card__title {
  margin: 0;
  font-size: 1.125rem;
  font-weight: 600;
  line-height: 1.25;
  color: var(--text-primary, #0f172a);
}

.sub-card__price {
  font-size: 1rem;
  font-weight: 600;
  font-variant-numeric: tabular-nums;
  color: var(--text-primary, #0f172a);
}

.sub-card__desc {
  margin: 0;
  max-width: 65ch;
  font-size: 0.9375rem;
  line-height: 1.5;
  color: var(--text-secondary, #475569);
  text-align: start;
}

.sub-field {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.sub-field__label {
  font-size: 0.875rem;
  font-weight: 500;
  color: var(--text-primary, #0f172a);
}

.sub-field__input {
  /* >= 16px evita auto-zoom en iOS Safari */
  font-size: 1rem;
  line-height: 1.5;
  padding: 8px 12px;
  min-height: 44px;
  border-radius: 8px;
  border: 1px solid var(--border-input, #64748b);
}

.sub-field__error {
  margin: 0;
  font-size: 0.875rem;
  color: var(--text-danger, #b91c1c);
}

.sub-card__actions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  margin-top: 16px;
}

.sub-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-height: 44px;
  min-width: 44px;
  padding: 8px 16px;
  border-radius: 8px;
  font-size: 0.9375rem;
  font-weight: 500;
  cursor: pointer;
  transition: transform 150ms ease-out, background-color 150ms ease-out;
}

.sub-btn--icon {
  padding: 0;
  background: transparent;
  border: 1px solid transparent;
  color: var(--text-secondary, #475569);
}

.sub-btn--primary {
  background: var(--accent-primary, #1d4ed8);
  color: #ffffff;
  border: 1px solid transparent;
}

.sub-btn--secondary {
  background: transparent;
  color: var(--text-primary, #0f172a);
  border: 1px solid var(--border-input, #64748b);
}

.sub-btn:focus-visible,
.sub-field__input:focus-visible {
  outline: 2px solid var(--focus-ring, #1d4ed8);
  outline-offset: 2px;
}

.sub-btn:active {
  transform: scale(0.98);
}

@media (prefers-reduced-motion: reduce) {
  .sub-btn {
    transition: none;
  }
  .sub-btn:active {
    transform: none;
  }
}
```
