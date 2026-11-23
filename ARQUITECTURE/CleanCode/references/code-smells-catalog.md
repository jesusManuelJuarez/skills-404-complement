# Catálogo de Code Smells y Priorización

Basado en *Refactoring (2nd Ed.)* de Martin Fowler y ampliado con criterios de complejidad de John Ousterhout (*A Philosophy of Software Design*).

---

## Priorización por Severidad (para Auditorías)

Antes de listar todos los detalles, clasifica siempre los hallazgos según su impacto real en mantenibilidad y riesgo de bugs:

- **Crítico:** Riesgo directo de bug silencioso, corrupción de datos, `catch` vacío que oculta fallos, mezcla de reglas de negocio críticas con I/O sin tests, o fuga masiva de contratos externos en el dominio.
- **Alto:** Complejidad cognitiva elevada (anidamiento > 3 niveles, funciones largas multi-responsabilidad, *God Class* / *Divergent Change*, *Primitive Obsession* en dinero/fechas/estados).
- **Medio:** Duplicación de conocimiento de negocio, *Feature Envy*, *Flag Arguments*, *Data Clumps*, *Shallow Wrappers* que añaden indirección sin ocultar complejidad.
- **Bajo:** Nombres mejorables, comentarios narrativos redundantes, código muerto menor o inconsistencias estilísticas locales.

---

## 1. Bloaters (Código que creció demasiado)

| Smell | Síntoma observable | Riesgo | Refactorización recomendada | Cuándo NO tocarlo |
| :--- | :--- | :--- | :--- | :--- |
| **Long Function** | Función > 40-50 líneas con múltiples bloques separados por comentarios o líneas en blanco. | Difícil de leer, testear y modificar sin romper efectos colaterales. | *Split Phase*, *Extract Function*, *Decompose Conditional*. | Si es una tabla declarativa lineal (mapeo 1:1 o configuración) sin lógica ramificada. |
| **Large Class / God Module** | Archivo/clase con cientos de líneas, > 7 dependencias o métodos que tocan grupos de campos disjuntos. | Alto acoplamiento; cualquier cambio exige entender todo el archivo. | *Extract Class*, agrupar por actor/razón de cambio (SRP). | Si representa una máquina de estados altamente cohesionada donde dividirla generaría *Feature Envy*. |
| **Long Parameter List** | Función con 4+ parámetros o múltiples flags/primitivos seguidos. | Llamadas propensas a error posicional y firmas frágiles. | *Introduce Parameter Object*, *Preserve Whole Object*. | Funciones de utilidad matemática pura o firmas impuestas por un framework. |
| **Primitive Obsession** | Usar `string`/`number` sueltos para dinero (`number` sin moneda/unidad), emails, IDs distintos o estados mágicos (`status === 2`). | Bugs por intercambio de unidades (centavos vs. pesos) o strings inválidos dispersos. | *Replace Primitive with Object*, tipos unión discriminados o *Branded Types*. | Valores primitivos internos de vida efímera sin invariantes propias. |
| **Data Clumps** | Los mismos 3-4 campos (`startDate, endDate` o `lat, lng`) viajan juntos por múltiples funciones. | Duplicación de validaciones y parámetros repetitivos. | *Extract Class* / *Introduce Parameter Object* (`DateRange`, `Coordinates`). | Si aparecen juntos solo en 1 o 2 sitios triviales. |

---

## 2. Control-Flow & OO Abusers (Complejidad de flujo y diseño)

| Smell | Síntoma observable | Riesgo | Refactorización recomendada | Cuándo NO tocarlo |
| :--- | :--- | :--- | :--- | :--- |
| **Deep Nesting** | `if` dentro de `if` dentro de `for`, con `else` lejanos. | Carga cognitiva exponencial; casos borde olvidados. | *Replace Nested Conditional with Guard Clauses*. | Nunca es deseable más de 2-3 niveles; siempre conviene aplanar. |
| **Repeated Switches / If-Else Chains** | El mismo `switch (paymentType)` aparece en 3+ archivos distintos. | Olvidar actualizar un caso al agregar un nuevo tipo. | *Replace Conditional with Polymorphism* o mapa de estrategias (*Strategy Map*). | Un único `switch` exhaustivo centralizado (en TypeScript con `never` check) es simple y preferible a 5 clases. |
| **Flag Argument** | Parámetro booleano que cambia radicalmente lo que hace la función. | El llamador no entiende qué hace `save(order, true)` sin leer la definición. | *Remove Flag Argument* (dividir en 2 funciones con nombre explícito). | Opciones declarativas simples que no bifurcan algoritmos distintos. |
| **Refused Bequest** | Subclase que hereda métodos que no usa o lanza `"Not implemented"`. | Rompe Liskov (LSP) y engaña a quien lee la jerarquía. | *Replace Subclass with Delegate* (composición sobre herencia). | N/A — suele ser señal clara de herencia mal modelada. |

---

## 3. Change Preventers (Fricción al evolucionar)

| Smell | Síntoma observable | Riesgo | Refactorización recomendada | Cuándo NO tocarlo |
| :--- | :--- | :--- | :--- | :--- |
| **Divergent Change** | Un mismo módulo se edita por motivos sin relación (ej. cada vez que cambia el esquema SQL *y* cada vez que cambia el formato PDF). | Regresiones cruzadas entre responsabilidades ajenas. | *Split Phase* / *Extract Module* separando por razón de cambio. | Prototipos tempranos donde aún no está claro el eje de cambio. |
| **Shotgun Surgery** | Agregar un campo o regla obliga a editar 6-10 archivos pequeños dispersos. | Cambios incompletos y alto costo de mantenimiento. | *Move Function*, *Inline Class* / consolidar en un **Deep Module** (Ousterhout). | Separación legítima entre capas de frontera y núcleo en sistemas grandes. |
| **Duplicated Business Knowledge** | Misma regla de cálculo/validación copiada en 3+ lugares (no solo sintaxis parecida). | Corregir un bug en un sitio y dejarlo vivo en otros dos. | *Extract Function* compartida tras verificar que comparten razón de cambio. | Si solo se parecen en forma pero pertenecen a dominios distintos (*Wrong Abstraction* — Metz). |

---

## 4. Dispensables (Ruido y sobreingeniería)

| Smell | Síntoma observable | Riesgo | Refactorización recomendada | Cuándo NO tocarlo |
| :--- | :--- | :--- | :--- | :--- |
| **Shallow Module / Middle Man** | Clase o función cuya única misión es delegar a otra sin ocultar complejidad ni traducir abstracción. | Indirección excesiva (*Pass-Through Methods*, Ousterhout); el lector salta sin aprender nada. | *Inline Function* / *Remove Middle Man*. | Si es un adaptador explícito que aísla un SDK externo o una frontera arquitectónica acordada. |
| **Speculative Generality** | Interfaces con 1 implementación, genéricos crípticos o hooks `"por si algún día..."`. | Complejidad accidental sin retorno real (*YAGNI*). | *Collapse Hierarchy* / *Inline* hasta que exista un segundo caso real. | Librerías públicas diseñadas para extensibilidad de terceros. |
| **Dead Code & Commented Code** | Funciones sin llamadores, imports sin uso, bloques comentados viejos. | Confunde al lector y al LLM sobre qué código está vivo. | Eliminar directamente (el historial vive en Git). | Exports públicos de una librería que forman parte de su API documentada. |
| **Narrative Comments** | Comentarios que parafrasean la línea siguiente (`// loop over users`). | Ruido visual que desincentiva leer comentarios realmente importantes. | Borrar comentario; si el código no se entiende solo, renombrar o extraer función. | Comentarios que explican el **por qué**, invariantes o workarounds externos. |

---

## 5. Couplers (Acoplamiento excesivo)

| Smell | Síntoma observable | Riesgo | Refactorización recomendada | Cuándo NO tocarlo |
| :--- | :--- | :--- | :--- | :--- |
| **Feature Envy** | Una función accede a 4+ propiedades/métodos de otro objeto para calcular algo sobre él. | Lógica dispersa lejos de los datos que la sostienen. | *Move Function* hacia el módulo/tipo dueño de esos datos. | Estrategias o formateadores externos que intencionalmente leen un modelo puro para renderizarlo. |
| **Message Chains (Ley de Demeter)** | `a.getB().getC().getD().doSomething()`. | Cualquier cambio intermedio rompe a todos los llamadores. | *Hide Delegate* (exponer método semántico en el objeto inmediato). | Fluent builders (`query.where(...).limit(...)`) o acceso a propiedades de un DTO plano. |
| **Leaky External Boundary** | Tipos de Axios, Prisma, Stripe o códigos HTTP crudos propagados por la lógica de negocio. | Acoplamiento total al proveedor; imposible testear sin mockear infraestructura. | *Wrap External Boundary* con tipos y errores propios del dominio. | Scripts internos triviales de automatización o glue code sin lógica de negocio. |
