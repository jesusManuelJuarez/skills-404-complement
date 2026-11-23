---
name: hexagonal-architecture
description: >-
  Diseña, audita y migra módulos aplicando Arquitectura Hexagonal (Ports & Adapters)
  y principios SOLID (Alistair Cockburn, Robert C. Martin, Vaughn Vernon, Mark Seemann).
  Usar cuando el desarrollador pida estructurar capas, desacoplar lógica de negocio
  de frameworks/BD/APIs, aplicar SOLID, definir puertos y adaptadores o diseñar casos
  de uso testeables. NO usar para limpieza micro de funciones/nombres (clean-code),
  diseño visual (ui-engineering-rules) ni auditorías de rendimiento o seguridad (qa-validator).
metadata:
  version: "1.0.0"
  category: architecture
---

# Arquitectura Hexagonal (*Ports & Adapters*) & SOLID

Skill operativa para estructurar software donde las reglas de negocio permanecen aisladas de frameworks, bases de datos y proveedores externos, sin caer en ceremonia innecesaria.

---

## 1. Marco de Referentes

| Referente | Obra | Aporte operativo en esta skill |
| :--- | :--- | :--- |
| **Alistair Cockburn** | *Hexagonal Architecture (Ports & Adapters)* | Separar núcleo de aplicación de actores primarios y secundarios mediante puertos. |
| **Robert C. Martin** | *Clean Architecture* & *SOLID* | Regla de Dependencia hacia adentro y principios SRP, OCP, LSP, ISP, DIP. |
| **Vaughn Vernon** | *Implementing Domain-Driven Design* | Modelado de dominio rico, puertos con lenguaje de negocio y aislamiento de infraestructura. |
| **Mark Seemann** | *Dependency Injection Principles* | *Composition Root* único, inyección por constructor/función y evitación de *Service Locator*. |

### Cuándo SÍ y cuándo NO usar Arquitectura Hexagonal
- **SÍ aplicar cuando:** existe lógica de negocio no trivial, múltiples canales de entrada (HTTP + cola + CLI), integraciones externas reemplazables o vida larga que exige tests rápidos sin infraestructura.
- **NO aplicar cuando:** es un script puntual, prototipo desechable o CRUD 1:1 sin reglas de negocio (un módulo simple con fronteras limpias basta).

---

## 2. Modos de Operación

1. **Diseño de Módulo / Feature:** Definir dominio, casos de uso, puertos mínimos (ISP/DIP) y adaptadores desde cero.
2. **Migración / Refactorización:** Extraer lógica de negocio fuera de controllers HTTP, ORMs o SDKs en pasos incrementales seguros.
3. **Auditoría Arquitectónica & SOLID:** Detectar fugas de infraestructura, violaciones a la Regla de Dependencia, puertos obesos o *Hexagonal Ceremony*.
4. **Mentoría:** Explicar qué vive en cada anillo y qué principio SOLID justifica la decisión.

---

## 3. Protocolo de Ejecución

1. **Identificar el núcleo de dominio y sus actores:** Separa invariantes de negocio de detalles de transporte (HTTP/CLI) y almacenamiento (SQL/Redis/APIs).
2. **Definir casos de uso y puertos mínimos (ISP + DIP):** Modela puertos pequeños orientados al consumidor (`PaymentCharger`, `OrderCatalogReader`), no espejos de ORMs o SDKs.
3. **Ubicar infraestructura en adaptadores:** Paginación, prevención de N+1, caché Redis, colas, timeouts, pools y OpenTelemetry viven en adaptadores cumpliendo el puerto.
4. **Ensamblar en el *Composition Root*:** Instancia e inyecta adaptadores reales en un único punto de entrada; prohíbe `new ConcreteAdapter()` dentro del núcleo.
5. **Validar con tests en memoria:** Prueba casos de uso con *fakes* en memoria (sin BD ni red) y adaptadores con integración acotada.

---

## 4. Guardrails

- **Prohibida la *Hexagonal Ceremony* en CRUDs triviales:** Si un endpoint solo lee y devuelve una fila sin reglas de negocio, no crees 6 carpetas y 4 interfaces espejo.
- **Prohibidos puertos que copian SDKs/ORMs:** Un puerto con firmas específicas de Prisma/Stripe filtra infraestructura al núcleo.
- **Prohibidos DTOs redundantes sin transformación:** No dupliques modelos con campos idénticos entre cada capa si no protegen una frontera real.
- **Prohibido importar frameworks en Domain/Application:** Ni `express`, `next`, `prisma`, `axios`, `ioredis` ni `process.env` dentro del núcleo.
- **Higiene técnica estricta:** Cero `any` en diseños/refactors nuevos, cero emojis y comentarios breves (≤ 1 línea) solo para el *por qué*.

---

## 5. Mapa de Referencias y Sinergia

| Recurso | Cuándo leerlo | Enlace |
| :--- | :--- | :--- |
| **SOLID y Regla de Dependencia** | Al diseñar contratos, dividir interfaces o auditar acoplamientos. | [solid-and-dependency-rule.md](./references/solid-and-dependency-rule.md) |
| **Guía de Puertos y Adaptadores** | Para ubicar Dominio, Use Cases, Redis, Colas, BD y OpenTelemetry. | [ports-and-adapters-guide.md](./references/ports-and-adapters-guide.md) |
| **Anti-patrones y Checklist** | Antes de aprobar un diseño o cerrar una auditoría arquitectónica. | [anti-patterns-and-checklist.md](./references/anti-patterns-and-checklist.md) |
| **Ejemplo 01 (Caso de Uso + SOLID)** | Migración completa de controller acoplado a Hexagonal en TypeScript. | [01-hexagonal-use-case-and-solid.md](./examples/01-hexagonal-use-case-and-solid.md) |
| **Ejemplo 02 (Resiliencia + Tests)** | Adaptador HTTP con timeout/OTel, caché decorador y fakes en memoria. | [02-adapter-resilience-and-testing.md](./examples/02-adapter-resilience-and-testing.md) |

### Sinergia con otras skills
- Para legibilidad interna, nombrado y *code smells* de funciones, combina con [Clean Code](../CleanCode/SKILL.md).
- Para validar eficiencia (N+1, Redis, paginación, colas), seguridad (OWASP) y resiliencia/observabilidad en adaptadores y UI, ejecuta [QA Validator](../../QA-VALIDATOR/SKILL.md).

---

## 6. Contrato de Salida

- **En Auditoría:** Veredicto (`PASS` | `PASS WITH WARNINGS` | `BLOCK`) → Violaciones a la Regla de Dependencia y SOLID priorizadas (`Crítico`, `Alto`, `Medio`, `Bajo`) con ubicación, riesgo y refactor incremental.
- **En Diseño / Migración:** Mapa de dependencias y carpetas → Puertos y tipos de dominio sin `any` → Casos de uso, adaptadores y *Composition Root* → Decisiones de **qué no se abstrajo**.
