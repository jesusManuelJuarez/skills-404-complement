# SOLID y la Regla de Dependencia en TypeScript

Guía operativa para aplicar **SOLID** sin caer en jerarquías innecesarias ni abstracciones especulativas.

---

## Regla de Dependencia (*Dependency Rule* — Robert C. Martin)

Las dependencias de código fuente **solo pueden apuntar hacia adentro**:
`Infraestructura (Adapters / Frameworks)` → `Aplicación (Use Cases / Ports)` → `Dominio (Entities / Value Objects / Policies)`.

- El **Dominio** no importa nada de Aplicación ni de Infraestructura.
- La **Aplicación** importa del Dominio y define **Puertos** (interfaces/contratos); nunca importa clientes HTTP, ORMs ni frameworks web.
- La **Infraestructura** implementa esos puertos y traduce entre el mundo externo y el lenguaje del dominio.

---

## 1. S — *Single Responsibility Principle* (SRP)
> *"Un módulo debe tener una, y solo una, razón para cambiar (un actor de negocio)."*

- **Síntoma de violación:** Un servicio o clase cambia tanto cuando el equipo de Finanzas ajusta el cálculo de impuestos como cuando Infraestructura cambia de proveedor SMTP o el equipo API cambia el formato JSON.
- **Criterio práctico:** Separa **cálculo/regla de dominio**, **orquestación de caso de uso** y **formato/transporte**.
- **Límite anti-sobreingeniería:** No significa "una función por archivo". Funciones altamente cohesionadas que sirven al mismo actor y cambian juntas pertenecen al mismo módulo.

```typescript
// Mal: mezcla regla de negocio + SQL + HTTP
async function createInvoice(req: HttpReq, res: HttpRes, db: Pool) {
  const tax = req.body.amount * 0.21;
  await db.query("INSERT INTO invoices ...", [req.body.amount, tax]);
  res.status(201).json({ total: req.body.amount + tax });
}

// Bien: dominio puro separado de infraestructura
function calculateInvoiceTotals(netCents: number, vatRate: number) {
  const taxCents = Math.round(netCents * vatRate);
  return { netCents, taxCents, totalCents: netCents + taxCents };
}
```

---

## 2. O — *Open/Closed Principle* (OCP)
> *"Abierto a extensión, cerrado a modificación en el núcleo estable."*

- **Síntoma de violación:** Para agregar un nuevo proveedor de pagos, canal de notificación o estrategia de caché, hay que editar el núcleo del caso de uso agregando más `if/else` y nuevas dependencias directas.
- **Criterio práctico:** Extiende mediante **puertos**, **composición de funciones** o **decoradores** (por ejemplo, envolver un `ProductCatalogReader` con un `CachedCatalogReader` en Redis sin tocar el caso de uso).
- **Límite anti-sobreingeniería:** No crees interfaces de extensión anticipadas para algo que solo tiene 1 variante estable (*YAGNI*). Aplica OCP en puntos reales de variación (proveedores externos, reglas múltiples, decoradores transversales).

---

## 3. L — *Liskov Substitution Principle* (LSP)
> *"Cualquier adaptador que implemente un puerto debe honrar su contrato completo sin sorprender al caso de uso."*

- **Síntoma de violación:** El adaptador en memoria devuelve `null` si no existe el registro, el de Postgres lanza `PrismaClientKnownRequestError`, y el de caché ignora la paginación o devuelve objetos mutables compartidos.
- **Criterio práctico:** Todos los adaptadores de un mismo puerto deben:
  - Respetar las mismas precondiciones y postcondiciones.
  - Traducir sus fallos técnicos a los **mismos errores o tipos `Result`** definidos por el puerto.
  - Nunca lanzar `throw new Error("Not implemented")` en métodos heredados.

```typescript
// Mismo contrato de null y errores en todos los adaptadores
export interface OrderReader {
  findById(orderId: string): Promise<Order | null>;
}
```

---

## 4. I — *Interface Segregation Principle* (ISP)
> *"Ningún caso de uso debe depender de métodos que no utiliza."*

- **Síntoma de violación:** Un puerto `UserRepository` con 18 métodos (`create`, `update`, `delete`, `findByEmail`, `listPaginated`, `exportCsv`, `countActive`...) inyectado en un caso de uso que solo necesita leer el email del usuario.
- **Criterio práctico:** Define **puertos pequeños orientados al consumidor** (1 a 3 métodos por puerto). Un mismo adaptador de BD puede implementar múltiples puertos si conviene, o existir adaptadores enfocados por capacidad (`OrderWriter`, `OrderPaginatedReader`).
- **Beneficio directo:** Tests unitarios mínimos (solo creas un fake con 1 método) y bajo acoplamiento.

```typescript
// Mal: puerto obeso
interface OrderRepository {
  save(order: Order): Promise<void>;
  findById(id: string): Promise<Order | null>;
  listByCustomer(customerId: string, cursor?: string): Promise<Page<Order>>;
  markAsRefunded(id: string): Promise<void>;
  getMonthlyRevenueReport(year: number): Promise<RevenueRow[]>;
}

// Bien: puertos segregados por caso de uso
export interface OrderSaver {
  save(order: Order): Promise<void>;
}

export interface CustomerOrderReader {
  listByCustomer(query: { customerId: string; cursor?: string; limit: number }): Promise<OrderPage>;
}
```

---

## 5. D — *Dependency Inversion Principle* (DIP)
> *"Los módulos de alto nivel no dependen de los de bajo nivel; ambos dependen de abstracciones definidas por el alto nivel."*

- **Síntoma de violación:** `SubmitOrderUseCase` importa `Stripe` o `PrismaClient` directamente, o define su interfaz con tipos importados de `@prisma/client`.
- **Criterio práctico:**
  - El **caso de uso** define el puerto en términos de su propio dominio (`PaymentGateway.charge(ChargeCommand)`).
  - El **adaptador** (`StripePaymentAdapter`) importa ese puerto y lo implementa.
  - El **Composition Root** conecta ambos al arrancar la app.
- **Límite anti-sobreingeniería:** No necesitas DIP para utilidades puras y estables sin I/O (como cálculos de fechas o formateo simple de strings). Resérvalo para **fronteras con efectos secundarios, I/O o dependencias volátiles**.
