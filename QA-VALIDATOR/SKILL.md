---
name: qa-validator
description: >-
  Valida y guía la implementación de estándares de producción en comportamiento,
  eficiencia (frontend, backend y datos), seguridad y resiliencia/observabilidad
  (Kleppmann, Nygard, OWASP, Web.dev Core Web Vitals, Khorikov, OpenTelemetry).
  Usar durante la construcción de features nuevas, durante refactorizaciones o como
  Quality Gate en auditorías de PR y pre-producción. NO usar para limpieza micro de
  nombres o funciones (clean-code), diseño de capas y puertos (hexagonal-architecture)
  ni reglas visuales de alineación/botones/semántica DOM (ui-engineering-rules).
metadata:
  version: "1.0.0"
  category: qa
---

# QA Validator — Estándares de Producción, Eficiencia, Seguridad y Resiliencia

Skill transversal de validación y construcción preventiva. Asegura que el software sea **correcto, rápido, seguro, resiliente y observable** bajo condiciones reales de producción.

---

## 1. Marco de Referentes

| Referente / Estándar | Aporte operativo en esta skill |
| :--- | :--- |
| **Martin Kleppmann** (*DDIA*) | Paginación por cursor, índices, eliminación de N+1, pooling, Redis, colas e idempotencia. |
| **Michael Nygard** (*Release It!*) | Timeouts obligatorios, retries con backoff/jitter, *circuit breakers* y degradación elegante. |
| **Web.dev Core Web Vitals** | WebP/AVIF, `loading="lazy"`, *skeletons*, *debounce*, *Optimistic UI*, streaming IA, SSR/SSG, *code splitting*, virtualización y CDN. |
| **OWASP Top 10 / ASVS** | Validación en frontera, SQL parametrizado, anti-IDOR, XSS/CSRF, secretos, rate limit y limpieza de dependencias sin usar. |
| **Vladimir Khorikov** (*Unit Testing*) | Pruebas sobre comportamiento observable, casos borde y balance unitarias/integración/E2E. |
| **OpenTelemetry** | Propagación `traceparent`/`traceId`, spans en BD/Redis/APIs, métricas RED y logs sin PII. |

---

## 2. Modos de Operación

1. **Durante construcción (Preventivo):** Verificar qué reglas de rendimiento, caché, paginación y seguridad aplican antes de cerrar una feature.
2. **Durante refactorización (Regresión Cero):** Confirmar que el refactor no introdujo N+1, pérdida de timeouts, exposición de datos ni degradación de UX.
3. **Quality Gate / Auditoría Pre-PR:** Auditar el cambio completo y emitir veredicto (`PASS` | `PASS WITH WARNINGS` | `BLOCK`).

### Activación Selectiva (*Progressive Disclosure*)
| Superficie tocada | Archivo a cargar en `./references/` |
| :--- | :--- |
| **UI, imágenes, búsquedas, listas, bundle JS, streaming IA** | [performance-frontend.md](./references/performance-frontend.md) |
| **APIs, SQL/ORM, índices, pool, Redis, colas, Brotli/Gzip** | [performance-backend-data.md](./references/performance-backend-data.md) |
| **Auth, inputs externos, IDOR, SQLi, dependencias sin usar** | [security-owasp.md](./references/security-owasp.md) |
| **Timeouts, retries, degradación, trazas OpenTelemetry** | [resilience-and-observability.md](./references/resilience-and-observability.md) |
| **Lógica de negocio, casos borde, contratos API y tests** | [behavior-and-testing.md](./references/behavior-and-testing.md) |

---

## 3. Protocolo de Ejecución

1. **Clasificar superficie:** Identifica si el cambio toca Frontend, Backend/Datos, Seguridad, Resiliencia o Lógica/Tests y carga solo las referencias necesarias.
2. **Validar comportamiento y casos borde primero:** Revisa invariantes (dinero, fechas, vacíos, concurrencia) y tests observables.
3. **Auditar eficiencia en red, carga y datos:** Verifica puntos críticos de Frontend y Backend/Datos según la escala del proyecto.
4. **Auditar seguridad, resiliencia y observabilidad:** Comprueba validación en frontera, autorización por recurso, timeouts, dependencias limpias y trazas OpenTelemetry.
5. **Emitir veredicto:** Clasifica hallazgos en escala única (`Crítico`, `Alto`, `Medio`, `Bajo`) y entrega acciones concretas.

---

## 4. Guardrails

- **Pragmatismo costo/beneficio:** No bloquees un MVP interno de bajo tráfico por no tener Redis o CDN multi-región; márcalo como `Medio` (`PASS WITH WARNINGS`).
- **Bloqueantes (`Crítico`) desde el día 1:** SQLi, IDOR, secretos expuestos, N+1 evidente en endpoint principal, listados sin paginación/límite, APIs externas sin timeout o `catch` silenciosos.
- **No romper a11y ni SEO al optimizar:** Nunca apliques `loading="lazy"` a la imagen hero (LCP), preserva teclado en listas virtualizadas y evita *flicker* de hidratación.
- **Higiene técnica estricta:** Cero `any` en código propuesto, cero emojis y comentarios breves (≤ 1 línea) solo para el *por qué*.

---

## 5. Mapa de Referencias y Sinergia

| Archivo | Propósito específico | Enlace |
| :--- | :--- | :--- |
| **Performance Frontend** | WebP/AVIF, lazy loading, skeletons, debounce, deduplicación, Optimistic UI, streaming IA, SSR/SSG, terceros diferidos, code splitting, virtualización, CDN. | [performance-frontend.md](./references/performance-frontend.md) |
| **Performance Backend & Datos** | Paginación, N+1, índices, pooling, Redis, colas, `Promise.all`, Brotli/Gzip, CDN, streaming SSE, idempotencia. | [performance-backend-data.md](./references/performance-backend-data.md) |
| **Seguridad OWASP** | Dependencias sin usar, validación, SQLi, anti-IDOR, XSS/CSRF, secretos, rate limit y CORS. | [security-owasp.md](./references/security-owasp.md) |
| **Resiliencia y Observabilidad** | Timeouts, retries/backoff, circuit breaker y trazado con OpenTelemetry. | [resilience-and-observability.md](./references/resilience-and-observability.md) |
| **Comportamiento & Testing** | Casos borde, contratos API, estrategia de pruebas y criterios de veredicto. | [behavior-and-testing.md](./references/behavior-and-testing.md) |
| **Ejemplo de Auditoría** | Auditoría full-stack real mostrando veredicto `BLOCK` y plan de corrección. | [01-qa-production-audit-sample.md](./examples/01-qa-production-audit-sample.md) |

### Sinergia con otras skills
- Para invariantes visuales, alineación, dimensiones de botones, tipografía, contraste y cero etiquetas/IDs duplicados, combina con [UI Engineering Rules](../UI/SKILL.md).
- Para arquitectura de capas/SOLID y legibilidad micro, apoya con [Hexagonal Architecture](../ARQUITECTURE/HexagonalArchitecture/SKILL.md) y [Clean Code](../ARQUITECTURE/CleanCode/SKILL.md).

---

## 6. Contrato de Salida

- **Veredicto:** `PASS` | `PASS WITH WARNINGS` | `BLOCK` (con justificación de 1-2 líneas).
- **Hallazgos Priorizados:** Ordenados de `Crítico` a `Bajo`, indicando ubicación, categoría, riesgo real y solución concreta.
- **Checklist Ejecutado:** Puntos verificados y qué quedó pendiente de probar en runtime (`tests`, `EXPLAIN ANALYZE`, build).
