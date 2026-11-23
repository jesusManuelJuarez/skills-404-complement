# Guía de Puertos, Adaptadores y *Composition Root*

Referencia práctica para organizar carpetas, contratos y ubicar correctamente las optimizaciones de producción dentro de una arquitectura hexagonal.

---

## 1. Anatomía de los Anillos

```text
src/modules/<contexto>/
├── domain/                   # Entidades, Value Objects, Políticas puras, Errores de dominio
├── application/              # Casos de uso + Puertos (driving/driven) + DTOs de entrada/salida
├── infrastructure/
│   ├── http/                 # Adaptadores primarios (controllers, validación de schema, auth)
│   ├── workers/              # Adaptadores primarios (consumidores de cola / cron)
│   ├── persistence/          # Adaptadores secundarios (SQL/ORM, repositorios, caché Redis)
│   └── gateways/             # Adaptadores secundarios (APIs externas, colas, mailer, OTel)
└── compositionRoot.ts        # Único punto donde se instancian e inyectan adaptadores concretos
```

### Puertos Primarios (*Driving Ports*) vs. Puertos Secundarios (*Driven Ports*)
- **Driving (entrada):** Definen *qué ofrece* la aplicación (ej. el caso de uso `SubmitOrder` o su interfaz de comando/respuesta). Los llaman controladores HTTP, CLI, workers o tests.
- **Driven (salida):** Definen *qué necesita* la aplicación del exterior (`OrderSaver`, `PaymentGateway`, `JobQueue`). Los implementan adaptadores de infraestructura.

---

## 2. Dónde Viven las Reglas de Eficiencia, Seguridad y Resiliencia

La Arquitectura Hexagonal **no** prohíbe Redis, colas, índices SQL ni OpenTelemetry: define en qué anillo vive cada preocupación (los detalles de implementación de cada regla viven en [QA Validator](../../../QA-VALIDATOR/SKILL.md)):

| Preocupación técnica | Dónde vive en Hexagonal | Regla de diseño |
| :--- | :--- | :--- |
| **Validación de entrada y Auth/IDOR** | Adaptador Primario (HTTP/GraphQL) + verificación de propiedad en Aplicación | Valida formato/tipos en HTTP; verifica propiedad de recurso (`ownerId === actorId`) en el caso de uso o política de dominio. |
| **Paginación y prevención de N+1** | Contrato en Puerto de Lectura + implementación en Adaptador de Persistencia | El puerto exige `{ cursor, limit }` (nunca `findAll()` ilimitado). El adaptador resuelve en lote sin N+1. |
| **Índices SQL y Pool de Conexiones** | Adaptador de Persistencia + `compositionRoot.ts` | El pool se crea una vez en el Composition Root; los índices viven junto al esquema de persistencia. |
| **Caché con Redis** | Adaptador Decorador sobre un Puerto de Lectura | Implementa el mismo puerto (OCP/LSP), ej. `CachedCatalogReader(redis, sqlReader)` sin que el caso de uso conozca Redis. |
| **Tareas lentas en Cola de Trabajo** | Puerto `BackgroundJobQueue` + Adaptador Worker | El caso de uso encola el trabajo y responde rápido; el worker invoca el caso de uso asíncrono. |
| **Timeouts, Retries y Circuit Breaker** | Adaptador Secundario (HTTP/SDK Gateway) | Configura timeouts y traduce fallos de red/5xx a errores de dominio antes de salir del adaptador. |
| **Compresión Brotli/Gzip, CDN y Rate Limit** | Adaptador Primario / Gateway HTTP / Edge | Políticas transversales de transporte; nunca dentro del dominio. |
| **Rastreo con OpenTelemetry** | Decorador o instrumentación en Adaptadores y *Composition Root* | Envuelve casos de uso o adaptadores con spans sin contaminar el dominio ni registrar PII. |
| **Streaming IA token a token** | Puerto `AiTextStreamer` (`AsyncIterable<string>`) + Adaptador SSE HTTP | El caso de uso expone el stream; el adaptador HTTP hace flush SSE y propaga cancelación. |

---

## 3. Reglas de los Contratos entre Capas

1. **Mapeo en la frontera del adaptador:**
   El adaptador de BD transforma `DbRow` → `DomainEntity` al leer, y `DomainEntity` → `DbRecord` al escribir. Nunca devuelvas filas crudas de ORM desde un repositorio.
2. **Transacciones sin filtrar el ORM:**
   Si un caso de uso requiere atomicidad entre varios cambios, usa un puerto `UnitOfWork` / `TransactionRunner` (`runInTransaction(async (txPorts) => ...)`), no pases `prismaClient` ni `sqlTransaction` explícitamente al dominio.
3. **Paralelismo explícito en Aplicación o Adaptadores:**
   Cuando un caso de uso necesita dos lecturas independientes (ej. perfil de usuario + saldo de créditos), usa `Promise.all([userReader.get(...), creditReader.get(...)])` sobre los puertos.
4. **Composition Root libre de lógica de negocio:**
   `compositionRoot.ts` solo lee configuración validada, inicializa pools/clientes y arma el grafo de dependencias.
