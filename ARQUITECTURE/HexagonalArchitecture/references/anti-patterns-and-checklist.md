# Anti-Patrones y Checklist de Auditoría Hexagonal

---

## Anti-Patrones Frecuentes (Qué Detectar y Evitar)

| Anti-patrón | Síntoma observable | Impacto | Corrección |
| :--- | :--- | :--- | :--- |
| **Hexagonal Ceremony** | 5 interfaces, 4 mappers y 3 DTOs idénticos para un `GET /health` o CRUD trivial de catálogo sin reglas. | Fricción absurda y código inflado (*Speculative Generality*). | Usar un módulo simple directo; reservar Hexagonal para módulos con lógica o integraciones reales. |
| **ORM Entity as Domain** | Clases/entidades de dominio llenas de decoradores `@Entity()`, `@Column()` o tipos `Prisma.OrderInclude`. | El dominio no puede compilar ni testearse sin el ORM. | Separar el modelo de dominio puro del esquema de persistencia en `infrastructure/persistence`. |
| **SDK-Shaped Ports** | Puerto `DatabasePort` con `query(sql)` o `StripePort` con `createPaymentIntent(stripeParams)`. | Falsa abstracción (DIP roto); cambiar de librería rompe toda la aplicación. | Diseñar el puerto desde la necesidad del caso de uso y en lenguaje del dominio. |
| **Anemic Domain + Fat Use Case** | Todos los cálculos, invariantes y transiciones de estado viven en un caso de uso de 200 líneas manipulando objetos pasivos. | Reglas duplicadas entre distintos casos de uso. | Mover invariantes y transiciones de estado a entidades/Value Objects o funciones puras de dominio. |
| **Service Locator** | Llamar `Container.get(OrderRepo)` dentro de una entidad o caso de uso. | Dependencias ocultas; tests frágiles y difíciles de aislar. | Inyectar dependencias explícitamente en el constructor o función fábrica desde `compositionRoot.ts`. |
| **Over-Mocking en Tests** | Tests de caso de uso con 10 `vi.fn()` que verifican el orden exacto de llamadas internas. | Tests acoplados a la implementación que fallan en cada refactor. | Usar *in-memory fakes* reales del puerto y verificar el resultado observable/estado final. |

---

## Checklist de Auditoría Arquitectónica

### Dependencias y Fronteras
- [ ] ¿`domain/` está 100% libre de imports de `application/`, `infrastructure/`, frameworks web, ORMs y SDKs?
- [ ] ¿`application/` solo importa de `domain/` y define sus propios puertos?
- [ ] ¿Existe un único *Composition Root* que conecta implementaciones reales con casos de uso?
- [ ] ¿Ningún adaptador expone excepciones crudas de HTTP/SQL/SDK hacia el caso de uso?

### SOLID Aplicado
- [ ] **SRP:** ¿Cada módulo separa cálculo/reglas de dominio, orquestación y transporte/persistencia?
- [ ] **ISP:** ¿Los puertos tienen solo los métodos que cada consumidor realmente usa (1-3 métodos típicos)?
- [ ] **LSP:** ¿Los adaptadores reales, decoradores (ej. caché) y *fakes* de test cumplen exactamente el mismo contrato de retorno y errores?
- [ ] **DIP / OCP:** ¿Pueden cambiarse BD, pasarela de pago o caché sin modificar la lógica del caso de uso?

### Pragmatismo y Operación
- [ ] ¿La complejidad de capas está justificada por la riqueza del dominio o número de integraciones?
- [ ] ¿Los puertos de listado exigen paginación/límites (sin `findAll()` abiertos)?
- [ ] ¿Se evitó crear interfaces redundantes o DTOs espejo sin transformación real?
