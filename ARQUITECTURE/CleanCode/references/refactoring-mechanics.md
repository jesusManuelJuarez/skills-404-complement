# Mecánicas Seguras de Refactorización

Basado en *Refactoring (2nd Ed.)* de Martin Fowler y *Working Effectively with Legacy Code* de Michael Feathers.

---

## Protocolo Previo de Seguridad (Antes de editar)

### 1. La regla de *Two Hats* (Fowler)
Nunca uses los dos sombreros al mismo tiempo:
- **Sombrero Refactor:** Mejoras estructura interna **sin cambiar comportamiento observable**; los tests existentes deben seguir pasando sin alterarse (salvo acoplamientos internos que estás limpiando explícitamente).
- **Sombrero Feature / Fix:** Agregas comportamiento o corriges un bug. Si el código está sucio, primero ponte el sombrero de refactor, limpia el camino, verifica tests y luego cambia al sombrero de feature.

### 2. Red de seguridad en código sin tests (*Characterization Tests* — Feathers)
Si vas a refactorizar lógica no trivial sin suite de tests:
1. No adivines qué hace cada rama rara: escribe primero **tests de caracterización** que capturen el comportamiento actual exacto (incluidos casos borde).
2. Ejecuta los tests en verde antes de tocar la estructura.
3. Aplica las mecánicas en pasos pequeños ejecutando tests tras cada paso.
4. Si encuentras un bug real durante la lectura, **anótalo aparte**; no lo corrijas silenciosamente en medio del refactor salvo acuerdo explícito.

---

## Las 8 Mecánicas Principales (en TypeScript)

### 1. *Replace Nested Conditional with Guard Clauses*
**Cuándo:** Anidamiento > 2 niveles con validaciones iniciales o casos borde al principio.
**Pasos:**
1. Identifica las condiciones de salida temprana o error.
2. Invierte la condición y retorna/lanza inmediatamente al inicio.
3. Desanida el camino feliz al nivel base y verifica tests.

```typescript
// Antes
function getPayAmount(emp: Employee): number {
  let result: number;
  if (emp.isSeparated) {
    result = separatedAmount();
  } else {
    if (emp.isRetired) result = retiredAmount();
    else result = normalPayAmount(emp);
  }
  return result;
}

// Después
function getPayAmount(emp: Employee): number {
  if (emp.isSeparated) return separatedAmount();
  if (emp.isRetired) return retiredAmount();
  return normalPayAmount(emp);
}
```

---

### 2. *Split Phase* (Validación / Cálculo Puro / Efectos I/O)
**Cuándo:** Una función mezcla parseo/validación de entrada, cálculo de reglas de negocio y persistencia/envío de emails/logs.
**Pasos:**
1. Identifica la frontera natural entre *calcular qué hacer* y *ejecutar los efectos*.
2. Extrae el cálculo a una función pura que reciba datos planos y devuelva un resultado o comando inmutable.
3. Haz que el orquestador haga: `leer/validar` → `llamar función pura` → `persistir/notificar`.
4. Agrega tests unitarios rápidos sobre la función pura (sin mocks).

---

### 3. *Extract Function* & *Decompose Conditional*
**Cuándo:** Una función mezcla niveles de abstracción o tiene una condición booleana compleja (`if (date >= start && date <= end && !user.isTrial && ...)`).
**Pasos:**
1. Extrae la condición compleja a un predicado con intención clara (`isEligibleForSummerDiscount(order, now)`).
2. Extrae bloques con propósito cohesivo (no de 1-2 líneas triviales que no aporten claridad).
3. Asegúrate de que el nombre describa **qué** logra el bloque, no **cómo** lo implementa.

---

### 4. *Introduce Parameter Object* & *Replace Primitive with Value Object*
**Cuándo:** 4+ parámetros o primitivos con reglas invariantes (`amount` en centavos vs. dólares, `currency`, `email`).
**Pasos:**
1. Agrupa parámetros que viajan juntos en una interfaz/type explícito (`DateRange`, `Money`).
2. Si el valor tiene invariantes (ej. `amount >= 0`, `Email` válido), encapsula su creación/validación en una fábrica o tipo dedicado (*Branded Type* o clase pequeña).
3. Migra los llamadores incrementalmente.

```typescript
type Cents = number & { readonly __brand: "Cents" };

function cents(value: number): Cents {
  if (!Number.isInteger(value) || value < 0) {
    throw new RangeError(`Invalid cents amount: ${value}`);
  }
  return value as Cents;
}
```

---

### 5. *Replace Conditional with Strategy Map / Polymorphism*
**Cuándo:** El mismo `switch` o `if/else if` sobre un tipo/estado aparece repetido en varios módulos o crece continuamente.
**Pasos:**
1. En TypeScript idiomático, prefiere primero un **mapa de estrategias tipado** (`Record<Kind, Handler>`) antes que crear una jerarquía de clases pesada.
2. Si el discriminante es un *Union Type*, garantiza exhaustividad en compilación con `assertNever(value)`.
3. Pasa a clases polimórficas solo cuando cada variante mantenga estado propio y múltiples métodos acoplados.

```typescript
type ShippingMethod = "STANDARD" | "EXPRESS" | "PICKUP";

const SHIPPING_COST_CENTS: Record<ShippingMethod, (weightKg: number) => number> = {
  STANDARD: (kg) => 500 + kg * 100,
  EXPRESS: (kg) => 1500 + kg * 250,
  PICKUP: () => 0,
};
```

---

### 6. *Move Function* & *Encapsulate Collection* (resolver *Feature Envy*)
**Cuándo:** Un módulo inspecciona internamente 4+ campos de otro objeto para calcular un derivado, o expone arrays mutables internos (`order.items.push(...)` desde fuera).
**Pasos:**
1. Mueve el cálculo junto a los datos que usa.
2. Expón colecciones como `readonly` (`readonly OrderItem[]`) y provee métodos explícitos con invariantes (`addItem`) si hay reglas que proteger.

---

### 7. *Introduce Special Case / Null Object*
**Cuándo:** Múltiples lugares repiten `if (customer === null)` devolviendo los mismos valores por defecto (`"Guest"`, tarifa `0`, sin notificaciones).
**Pasos:**
1. Crea un objeto o variante tipada explícita (`GuestCustomer` o `ANONYMOUS_USER`) que implemente la misma interfaz con comportamiento seguro por defecto.
2. Elimina los `if (!user)` repetitivos en los consumidores.

---

### 8. *Wrap External Boundary*
**Cuándo:** La lógica de negocio importa directamente clientes HTTP, SDKs de pagos, ORMs o `process.env`.
**Pasos:**
1. Define una interfaz pequeña orientada a lo que **tu aplicación necesita** (no un espejo de los 40 métodos del SDK).
2. Implementa el adaptador traduciendo requests/responses y convirtiendo errores técnicos externos en errores de dominio.
3. Inyecta el wrapper en el módulo de negocio; ahora es testeable en memoria con un doble simple.
