# Estándares de Resiliencia y Observabilidad (OpenTelemetry)

Basado en *Release It!* (Michael Nygard) y especificaciones de **OpenTelemetry**.

---

## 1. Resiliencia en Integraciones Externas

### Timeout obligatorio en toda API externa y consulta
- **Regla:** Ninguna llamada de red (`fetch`, Axios, SDKs, Redis, BD) puede quedar sin límite de tiempo. Sin timeout, un proveedor lento agota conexiones y workers de todo el backend (*cascading failure*).
- **Implementación:** Usa `AbortSignal.timeout(ms)` (ej. 3 000–8 000 ms en APIs síncronas; límites dedicados en streaming) y configura `statement_timeout` / `connectTimeout` en el pool de BD.

### Reintentos seguros y degradación elegante
- **Retries:** Reintenta solo errores transitorios (timeouts, 429, 502/503/504) en operaciones **idempotentes**, máximo 2-3 intentos con *exponential backoff + jitter*. Nunca reintentes sobre `400/401/403/404`.
- **Degradación:** Si un servicio secundario (recomendaciones, enriquecimiento no crítico) falla o abre un *Circuit Breaker*, devuelve una respuesta degradada segura en lugar de tumbar la operación principal.

---

## 2. Observabilidad y Rastreo con OpenTelemetry

- **Regla:** Instrumenta la aplicación para detectar peticiones lentas y fallos en producción de extremo a extremo:
  1. **Propaga `traceparent` / `traceId`** entre frontend/API gateway, servicios y colas de trabajo.
  2. **Envuelve operaciones clave con spans:** consultas BD, operaciones Redis, llamadas a APIs externas y casos de uso críticos.
  3. **Registra métricas RED:** *Rate* (rps), *Errors* (% 5xx/fallos) y *Duration* (latencia p50, p95, p99).
  4. **Logs estructurados en JSON correlacionados con `traceId`:** Prohibido registrar contraseñas, tokens, cookies, tarjetas o PII en atributos de spans o logs.

```typescript
interface SpanLike {
  setAttribute(key: string, value: string | number): void;
  end(): void;
}

interface TracerLike {
  startActiveSpan<R>(name: string, fn: (span: SpanLike) => Promise<R>): Promise<R>;
}

async function tracedOperation<T>(
  tracer: TracerLike,
  operationName: string,
  fn: () => Promise<T>,
): Promise<T> {
  return tracer.startActiveSpan(operationName, async (span) => {
    const startedAt = performance.now();
    try {
      const result = await fn();
      span.setAttribute("duration_ms", Math.round(performance.now() - startedAt));
      return result;
    } finally {
      span.end();
    }
  });
}
```
