---
name: nombre-skill-en-kebab-case
description: >-
  Describe en tercera persona qué hace esta skill y exactamente cuándo debe activarse.
  Usar cuando el desarrollador pida [caso 1], [caso 2] o [caso 3].
  NO usar para [tareas fuera de alcance que pertenecen a otra skill o dominio].
metadata:
  version: "1.0.0"
  category: categoria
---

# Nombre de la Skill

Resumen en 2-3 líneas del objetivo operativo de la skill y el problema recurrente que resuelve en proyectos de software.

---

## 1. Marco de Referentes

Indica los referentes, libros o estándares en los que se apoya esta skill y qué criterio operativo aporta cada uno:

| Referente / Fuente | Aporte operativo principal |
| :--- | :--- |
| Autor / Libro 1 | Principio o mecánica concreta que aporta. |
| Autor / Libro 2 | Principio o mecánica concreta que aporta. |

### Reglas clave y resolución de tensiones
Explica cómo equilibrar principios en conflicto (ej. abstracción vs. simplicidad, desacoplamiento vs. pragmatismo).

---

## 2. Modos de Operación

Identifica qué pidió el desarrollador antes de actuar y adopta el modo correspondiente:

1. **Auditoría / Revisión:** Diagnosticar artefactos existentes priorizando hallazgos por severidad sin reescribir todo sin confirmación.
2. **Ejecución / Refactorización:** Aplicar cambios incrementales y verificables preservando contratos y comportamiento.
3. **Diseño desde Cero:** Generar código o estructuras nuevas aplicando reglas preventivas desde el inicio.
4. **Mentoría:** Explicar el *por qué* de cada decisión citando el principio y referente aplicable.

---

## 3. Protocolo de Ejecución

1. **Contexto y red de seguridad:** Leer archivos relevantes, identificar invariantes y verificar tests/validaciones existentes.
2. **Diagnóstico priorizado:** Detectar primero los problemas de mayor impacto estructural antes que detalles cosméticos.
3. **Intervención atómica:** Aplicar o proponer cambios en pasos pequeños y reversibles.
4. **Verificación:** Ejecutar pruebas, linter, compilación o validación aplicable al entorno.
5. **Entrega:** Responder respetando el *Contrato de Salida* y sin omitir lo que quedó pendiente de probar.

---

## 4. Guardrails

- **Sin sobreingeniería:** Evita crear capas, abstracciones o configuraciones especulativas sin problema real que las justifique.
- **Sin emojis y con comentarios breves:** En código y salidas técnicas no uses emojis ni comentarios narrativos; comenta solo el *por qué* no obvio en 1 línea corta.
- **Respetar el alcance:** No modifiques archivos ni contratos públicos fuera de lo pedido sin confirmación explícita.

---

## 5. Mapa de Referencias y Sinergia

Carga estos archivos únicamente cuando la tarea requiera ese nivel de detalle:

| Recurso | Cuándo leerlo | Enlace |
| :--- | :--- | :--- |
| Referencia 1 | Al diseñar desde cero o fundamentar decisiones. | `./references/...` |
| Checklist / Catálogo | Durante auditorías o revisiones exhaustivas. | `./references/...` |
| Ejemplos prácticos | Al calibrar el estilo de código o el formato de salida. | `./examples/...` |

### Sinergia con otras skills
- Enlaza aquí a las skills complementarias sin duplicar sus reglas.

---

## 6. Contrato de Salida

- **Veredicto (en Auditoría):** `PASS` | `PASS WITH WARNINGS` | `BLOCK`.
- **Escala única de severidades:** `Crítico` | `Alto` | `Medio` | `Bajo`.
- **En Ejecución / Diseño:** Invariantes preservadas → Código final tipado sin `any` → Resultado de verificación ejecutada.
