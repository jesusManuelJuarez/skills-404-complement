# Comportamiento Correcto, Testing y Criterios de Quality Gate

Basado en *Unit Testing: Principles, Practices, and Patterns* (Vladimir Khorikov) y validación funcional orientada a producción.

---

## 1. Validación de Comportamiento y Casos Borde

Antes de aprobar una feature o refactor, verifica estos invariantes frecuentes donde suelen esconderse bugs:

- **Dinero y aritmética:** Cálculos monetarios en enteros (centavos) o decimal exacto; nunca flotantes binarios sin redondeo controlado.
- **Fechas y zonas horarias:** Almacenar y transmitir en UTC / ISO-8601; formatear a zona local solo en la capa de presentación.
- **Colecciones y vacíos:** Comportamiento explícito con arrays vacíos `[]`, cadenas en blanco `"   "`, `0`, valores negativos y duplicados.
- **Concurrencia y doble envío:** Qué pasa si el usuario hace doble clic o llegan 2 requests simultáneos sobre el mismo recurso (transacciones, locks optimistas o `Idempotency-Key`).
- **Contratos API:** Códigos HTTP semánticos (`400/422` validación, `401` no autenticado, `403` sin permiso, `404` no encontrado, `409` conflicto, `429` rate limit, `202` trabajo encolado) y compatibilidad hacia atrás en respuestas existentes.

---

## 2. Estados Completos de UI

Toda pantalla que consume datos asíncronos debe contemplar los 5 estados (sus reglas visuales, contraste y foco viven en [UI Engineering Rules](../../UI/SKILL.md)):

1. **Loading:** *Skeleton* con geometría estable (sin CLS).
2. **Empty:** Estado vacío claro con llamado a la acción útil.
3. **Error:** Mensaje comprensible + acción de reintento (sin stack traces).
4. **Success:** Datos validados y formateados.
5. **Optimistic / Pending:** Feedback instantáneo con *rollback* seguro ante fallo.

---

## 3. Estrategia de Pruebas (Khorikov)

| Tipo de prueba | Qué debe testear | Qué evitar |
| :--- | :--- | :--- |
| **Unitarias (Núcleo / Dominio)** | Reglas de negocio puras, validaciones, transiciones de estado y casos borde. Rápidas, deterministas, sin mocks de BD/red. | Testear getters triviales o verificar qué método interno privado fue llamado (*brittle tests*). |
| **Integración (Adaptadores / API)** | Contratos SQL reales (índices, ausencia de N+1), serialización, autorización (IDOR), timeouts con servidor falso local y colas. | Mockear la base de datos dentro de un test cuyo objetivo es probar el repositorio SQL. |
| **E2E (Flujos Críticos)** | Pocos tests cubriendo los flujos críticos de negocio (registro/login, checkout, búsqueda principal) sobre la app real. | Cubrir cada variación menor con E2E lentos y frágiles. |

---

## 4. Criterios de Decisión del Quality Gate

- **`BLOCK` (Hallazgos `Crítico` — no mergear hasta corregir):**
  - Vulnerabilidad de seguridad (IDOR, SQLi, XSS, secreto expuesto, falta de validación en frontera).
  - Riesgo severo de caída o corrupción (N+1 evidente en endpoint principal, listado sin paginación/límite, API externa sin timeout, mutación financiera sin idempotencia/transacción, `catch` vacío).
  - Regresión funcional o tests fallando en lógica crítica.

- **`PASS WITH WARNINGS` (Hallazgos `Alto` o `Medio` no bloqueantes según escala actual):**
  - Falta de optimizaciones recomendadas según tráfico actual (ej. Redis aún no necesario en MVP interno, virtualización pendiente para lista < 100 items, deuda menor de cobertura en flujo secundario).

- **`PASS` (Solo hallazgos `Bajo` o ninguno):**
  - Comportamiento, casos borde, seguridad, resiliencia y eficiencia cubiertos según la criticidad del módulo y verificados tras el último cambio.
