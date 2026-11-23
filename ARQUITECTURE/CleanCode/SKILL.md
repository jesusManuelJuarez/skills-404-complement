---
name: clean-code
description: >-
  Audita, organiza y refactoriza código fuente aplicando principios de Clean Code,
  Refactoring seguro y Diseño de Software (Robert C. Martin, Martin Fowler,
  John Ousterhout, Hunt & Thomas, Sandi Metz). Usar cuando el desarrollador pida
  revisar calidad de código, detectar code smells, reducir complejidad cognitiva,
  refactorizar funciones/clases/módulos, mejorar nombrado o manejo de errores, o
  escribir código mantenible desde cero. NO usar para definir arquitectura macro de
  capas/puertos y adaptadores, diseño visual UI, levantamiento de requisitos o planes QA.
metadata:
  version: "1.0.0"
  category: architecture
---

# Clean Code & Refactoring Seguro

Skill operativa para diagnosticar *code smells*, organizar módulos cohesivos y refactorizar código sin romper comportamiento ni caer en sobreingeniería.

---

## 1. Marco de Referentes

| Referente | Obra | Aporte operativo en esta skill |
| :--- | :--- | :--- |
| **Robert C. Martin** | *Clean Code* | Nombrado con intención, SLAP, CQS, separación de errores y fronteras (*boundaries*). |
| **Martin Fowler** | *Refactoring (2nd Ed.)* | Catálogo de *code smells*, regla *Two Hats* y mecánicas seguras paso a paso. |
| **John Ousterhout** | *A Philosophy of Software Design* | Módulos profundos (*deep modules*), *information hiding* y reducción de complejidad cognitiva. |
| **Hunt & Thomas** | *The Pragmatic Programmer* | DRY de conocimiento, ortogonalidad, *fail fast* y reversibilidad. |
| **Sandi Metz** | *POODR* / *"The Wrong Abstraction"* | Preferir duplicación temporal antes que una abstracción acoplada y llena de flags. |

### Resolución de tensiones entre principios
1. **Funciones pequeñas (Martin) vs. Módulos profundos (Ousterhout):** Divide cuando separes responsabilidades o niveles de abstracción y reduzcas carga cognitiva. No extraigas *shallow functions* de 2 líneas que solo reenvían llamadas.
2. **DRY (Hunt & Thomas) vs. Wrong Abstraction (Metz):** Aplica DRY sobre conocimiento de negocio compartido (*Rule of Three*). Si dos flujos evolucionan por razones distintas, mantenlos separados antes que acoplarlos con flags booleanos.

---

## 2. Modos de Operación

1. **Auditoría / Code Review:** Diagnosticar y priorizar hallazgos por severidad (`Crítico`, `Alto`, `Medio`, `Bajo`) sin reescribir todo el módulo hasta confirmar alcance.
2. **Refactorización Guiada:** Aplicar mecánicas pequeñas y seguras preservando comportamiento observable y contratos públicos.
3. **Diseño / Código Nuevo:** Escribir código limpio desde cero con tipos estrictos, *guard clauses*, pureza separada de I/O y fronteras aisladas.
4. **Mentoría:** Conectar cada cambio con el *code smell*, el principio y el autor, explicando también qué se decidió **no** abstraer.

---

## 3. Protocolo de Ejecución

1. **Contexto y red de seguridad:** Lee el archivo y sus consumidores cercanos. Si vas a refactorizar lógica crítica sin tests, propón o escribe primero *characterization tests*.
2. **Diagnóstico por impacto estructural:** Prioriza acoplamiento de fronteras, mezcla de I/O con dominio, anidamiento profundo y *primitive obsession* antes que estilo menor.
3. **Intervención atómica (*Two Hats*):** Separa agregar comportamiento de refactorizar; aplica una mecánica concreta por paso ([refactoring-mechanics.md](./references/refactoring-mechanics.md)).
4. **Verificación:** Ejecuta tests, linter y `tsc`/build disponibles; audita con [checklist-and-heuristics.md](./references/checklist-and-heuristics.md) que la complejidad cognitiva haya bajado.
5. **Entrega:** Responde según el *Contrato de Salida* reportando qué quedó verificado.

---

## 4. Guardrails

- **Prohibido mezclar features y refactor** en el mismo paso lógico (*Two Hats*).
- **Prohibido crear abstracciones especulativas** (*interfaces* con 1 implementación sin necesidad de testeo/desacoplamiento, fábricas genéricas o jerarquías profundas — *YAGNI*).
- **Prohibido romper contratos públicos o cambiar lógica de negocio silenciosamente** ni usar `any` en código nuevo o refactorizado.
- **Boy Scout Rule acotada:** Mejora el código que tocas y su vecindad inmediata; no lances limpiezas masivas fuera de alcance sin confirmación.
- **Comentarios breves, sin emojis y solo de *por qué*:** Borra comentarios que narran el *qué*; conserva o escribe únicamente comentarios cortos (≤ 1 línea) que expliquen una invariante o decisión no obvia.

---

## 5. Mapa de Referencias y Sinergia

| Archivo | Cuándo leerlo | Enlace |
| :--- | :--- | :--- |
| **Matriz de principios** | Al diseñar código nuevo o fundamentar reglas de nombrado, funciones, cohesión y errores. | [principles-matrix.md](./references/principles-matrix.md) |
| **Catálogo de Code Smells** | Al realizar una auditoría o priorizar deuda técnica por severidad. | [code-smells-catalog.md](./references/code-smells-catalog.md) |
| **Mecánicas de Refactor** | Antes de ejecutar un refactor complejo o código legacy sin tests. | [refactoring-mechanics.md](./references/refactoring-mechanics.md) |
| **Checklist y Heurísticas** | Antes de entregar código refactorizado o generar un reporte de auditoría. | [checklist-and-heuristics.md](./references/checklist-and-heuristics.md) |
| **Ejemplo 01 (Funciones)** | Para calibrar *guard clauses*, *split phase* y tipos en TypeScript. | [01-refactoring-conditionals-and-functions.md](./examples/01-refactoring-conditionals-and-functions.md) |
| **Ejemplo 02 (Módulos y DRY)** | Para calibrar *deep modules*, aislamiento de SDKs y evitar *wrong abstractions*. | [02-deep-modules-cohesion-and-boundaries.md](./examples/02-deep-modules-cohesion-and-boundaries.md) |
| **Ejemplo 03 (Auditoría)** | Para ver el formato exacto de respuesta en Modo Auditoría + Mentoría. | [03-audit-output-sample.md](./examples/03-audit-output-sample.md) |

### Sinergia con otras skills
- Para arquitectura macro de capas, puertos/adaptadores y principios SOLID, combina con [Hexagonal Architecture](../HexagonalArchitecture/SKILL.md).
- Para validar eficiencia (N+1, Redis, paginación, colas, WebP/AVIF, debounce, streaming), seguridad (OWASP) y resiliencia/OpenTelemetry, ejecuta [QA Validator](../../QA-VALIDATOR/SKILL.md).

---

## 6. Contrato de Salida

- **En Auditoría:** Veredicto (`PASS` | `PASS WITH WARNINGS` | `BLOCK`) → Resumen Ejecutivo → Hallazgos priorizados (`Crítico`, `Alto`, `Medio`, `Bajo`) con ubicación, *smell*, principio y mejora → Plan incremental.
- **En Refactorización / Diseño:** Cambios e invariantes preservadas (incluyendo qué no se abstrajo) → Código TypeScript estricto sin `any` → Resultado de verificación ejecutada.
