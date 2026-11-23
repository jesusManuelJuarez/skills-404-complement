# Checklist de Calidad y Heurísticas Anti-Sobreingeniería

Usa esta referencia antes de cerrar cualquier auditoría o refactorización.

---

## Test Rápido de Carga Cognitiva (5 Preguntas)

Antes de dar por terminado un cambio, responde honestamente:

1. **Lectura lineal:** ¿Puede otro desarrollador entender qué hace este módulo leyendo de arriba hacia abajo sin saltar por más de 2-3 archivos?
2. **Estado y mutación:** ¿Se redujo la cantidad de estado mutable compartido y efectos secundarios ocultos?
3. **Flujo de control:** ¿El camino feliz está al nivel base (sin anidamiento profundo ni `else` innecesarios)?
4. **Fronteras limpias:** ¿Los datos externos se validan en la entrada y los errores se traducen en la frontera?
5. **Menos es más:** ¿El código resultante es **más simple de entender y modificar** que antes, no solo "más elegante" en abstracto?

---

## Checklist Detallado por Categoría

### Nombrado y Claridad
- [ ] Los nombres revelan intención de dominio y evitan ruido (`data`, `info`, `helper`, `manager`, `utils`).
- [ ] El vocabulario es consistente (un mismo verbo para la misma semántica en todo el módulo).
- [ ] Las unidades y formatos no obvios están explícitos en el nombre o el tipo (`timeoutMs`, `amountCents`, `createdAtIso`).
- [ ] Los booleanos usan predicados positivos claros (`isActive`, `hasPermission`, `canRetry`).

### Funciones y Flujo
- [ ] Cada función opera en un único nivel de abstracción (SLAP).
- [ ] Las precondiciones usan *guard clauses* con retorno o error temprano (anidamiento ≤ 2 niveles).
- [ ] Ninguna función tiene más de 3 parámetros sueltos ni *flag arguments* booleanos opacos.
- [ ] La lógica de negocio pura está separada de efectos secundarios (I/O, reloj, red, BD).

### Cohesión, Módulos y Límites
- [ ] Los módulos son profundos (*deep modules*): interfaz simple que oculta complejidad real, sin *shallow wrappers* gratuitos.
- [ ] No hay *Feature Envy* ni cadenas largas tipo `a.b.c.d` sobre objetos con comportamiento.
- [ ] No se forzó una abstracción compartida entre flujos con razones de cambio distintas (*Wrong Abstraction*).
- [ ] Los SDKs, APIs HTTP y variables de entorno están aislados detrás de fronteras propias.

### Errores, Comentarios y Seguridad del Refactor
- [ ] Ningún `catch` traga errores silenciosamente ni destruye la causa original (`{ cause: err }`).
- [ ] Comentarios en código son breves (≤ 1 línea), sin emojis y explican solo el *por qué* no obvio.
- [ ] Los casos borde existentes (nulos, vacíos, límites numéricos) se preservaron intactos.
- [ ] Si se modificó código crítico sin tests previos, se propusieron o añadieron *characterization tests*.
- [ ] Se ejecutaron tests/typecheck/build disponibles tras el último cambio (o se reportó explícitamente qué no pudo correrse).

---

## Auditoría Anti-Sobreingeniería (Señales de Alarma)

Detente y simplifica si detectas alguna de estas señales en tu propio refactor:

| Señal de alarma | Diagnóstico | Acción correctiva |
| :--- | :--- | :--- |
| Creaste una `interface FooService` con una única implementación que nadie mockea ni sustituye. | *Speculative Generality* | Elimina la interfaz; exporta la función o clase concreta hasta que exista un segundo caso real. |
| Extrajiste 6 funciones de 2 líneas para reemplazar una función lineal de 25 líneas. | *Shallow Functions* (Ousterhout) | Reúne los pasos triviales dentro de una función cohesiva clara. |
| Agregaste un parámetro `mode: "A" \| "B"` o `isSpecial: boolean` lleno de `if` internos para "reutilizar" código. | *The Wrong Abstraction* (Metz) | Separa en dos funciones simples que compartan solo el núcleo realmente común. |
| Introdujiste un patrón complejo (Factory + Strategy + Decorator) para 2 condicionales simples. | Sobreingeniería de patrones | Usa un `Record<Key, Handler>` o un `switch` exhaustivo simple. |

---

## Plantilla de Reporte en Modo Auditoría

```markdown
### Veredicto: PASS | PASS WITH WARNINGS | BLOCK

### 1. Resumen Ejecutivo
- **Estado general:** [Saludable / Deuda moderada / Deuda alta]
- **Cuello de botella principal:** [1-2 líneas sobre el mayor problema estructural]

### 2. Hallazgos Priorizados
- **[Crítico/Alto/Medio/Bajo] Título breve** (`archivo.ts:L12-L45`)
  - **Code smell:** [Nombre del smell]
  - **Principio / Referente:** [Principio — Autor]
  - **Impacto:** [Por qué duele al mantener o testear]
  - **Mejora sugerida:** [Acción concreta]

### 3. Plan de Refactor Incremental
1. [Red de seguridad / tests si faltan]
2. [Cambio estructural mayor]
3. [Limpieza local de funciones/tipos]
4. [Verificación]

> ¿Quieres que aplique los pasos 1 y 2 ahora?
```
