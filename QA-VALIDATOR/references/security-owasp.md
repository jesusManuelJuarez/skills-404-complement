# Estándares de Seguridad OWASP e Higiene de Dependencias

Basado en **OWASP Top 10 / ASVS** y buenas prácticas de seguridad en APIs y aplicaciones web.

---

## 1. Higiene de Dependencias

- **Quitar dependencias sin usar:** Audita periódicamente con `knip`, `depcheck` y `npm audit` (o equivalente del ecosistema). Elimina librerías muertas, duplicadas (`moment` + `dayjs`) o imports residuales.
- **Impacto directo:** Reduce superficie de ataque (*supply chain*), evita CVEs innecesarios y achica el tamaño del bundle entregado al navegador.

---

## 2. Controles OWASP Obligatorios en Código y PRs

| Control | Qué verificar en fronteras y lógica | Riesgo que previene |
| :--- | :--- | :--- |
| **Validación estricta en fronteras** | Valida body, query, params, headers y archivos con esquemas explícitos (ej. Zod/Valibot) rechazando campos extra no esperados. | Inyección, *Mass Assignment* y corrupción de datos. |
| **Consultas parametrizadas** | Usa placeholders (`$1`, `?`) o Query Builders tipados; jamás concatenes input de usuario en SQL ni comandos shell. | SQL Injection / Command Injection. |
| **Autorización por recurso (Anti-IDOR)** | Verifica siempre que el actor autenticado tenga permiso sobre el `resourceId` o `tenantId` solicitado; nunca confíes en IDs enviados en body/query sin validar pertenencia. | *Broken Object Level Authorization (IDOR / BOLA)*. |
| **Protección contra XSS y CSRF** | Escapa HTML por defecto; sanitiza explícitamente antes de inyectar HTML crudo. Configura cookies de sesión con `HttpOnly`, `Secure`, `SameSite=Lax/Strict`. | Robo de sesión y ejecución de scripts en clientes. |
| **Secretos y manejo seguro de errores** | Secretos solo en variables de entorno o vault (jamás en repo ni expuestos al cliente `NEXT_PUBLIC_*`). En producción devuelve mensajes genéricos al cliente y registra el detalle solo en logs/OTel. | Fuga de credenciales y *Stack Trace Leakage*. |
| **Rate Limiting y CORS estricto** | Aplica rate limit en login, recuperación de clave, endpoints públicos y endpoints costosos de IA. Configura orígenes CORS explícitos (nunca `*` con credenciales). | Fuerza bruta, abuso de cuota LLM y DoS de aplicación. |
