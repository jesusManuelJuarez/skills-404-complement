# Ejemplo 03 — Salida Modelo en Modo Auditoría + Mentoría

Este ejemplo muestra exactamente cómo debe responder un LLM cuando el desarrollador activa la skill `clean-code` en **Modo Auditoría** (diagnosticar antes de reescribir).

---

## Prompt del desarrollador

> "Revisa `sessionHelper.ts` usando la skill de Clean Code. Dime qué problemas ves y cómo lo organizarías antes de tocar código."

```typescript
// sessionHelper.ts
import axios from "axios";

export async function handleUser(d: any, force: boolean) {
  // verificar si hay token
  if (d && d.token) {
    try {
      const res = await axios.post("https://auth.internal/verify", { t: d.token, f: force });
      if (res.data.ok === true) {
        d.lastSeen = Date.now();
        return res.data.user;
      } else {
        return null;
      }
    } catch (e) {
      return null;
    }
  }
  return null;
}
```

---

## Respuesta Modelo Esperada del LLM

### 1. Resumen Ejecutivo
- **Estado general:** Deuda alta para ser un archivo pequeño (18 líneas con alto riesgo de bugs silenciosos).
- **Cuello de botella principal:** La función mezcla validación de sesión, llamada HTTP directa con Axios, mutación de parámetro de entrada (`d.lastSeen = ...`) y un `catch` silencioso que confunde token inválido con caída del servicio de autenticación.

---

### 2. Hallazgos Priorizados

- **[Crítico] `catch (e)` silencioso que enmascara fallos de infraestructura** (`sessionHelper.ts:L12-L14`)
  - **Code smell:** *Swallowed Exception* / manejo ambiguo de errores.
  - **Principio / Referente:** *Fail Fast* y errores explícitos — Hunt & Thomas (*The Pragmatic Programmer*), Robert C. Martin (*Clean Code*).
  - **Impacto:** Si `auth.internal` está caído o hay un timeout, la función retorna `null` igual que si el token fuera inválido. El sistema expulsará usuarios legítimos sin dejar rastro en logs ni distinguir fallo de red vs. sesión expirada.
  - **Mejora sugerida:** Diferenciar un rechazo de autenticación (`{ status: "INVALID" }`) de una falla técnica (lanzar error contextualizado `AuthServiceUnavailableError` con `{ cause: e }`).

- **[Crítico] Acoplamiento directo a librería externa y URL hardcodeada** (`sessionHelper.ts:L7`)
  - **Code smell:** *Leaky External Boundary*.
  - **Principio / Referente:** *Wrap Boundaries* (Martin) & *Information Hiding* (Ousterhout).
  - **Impacto:** Imposible testear la lógica de sesión sin interceptar Axios y la URL está fija en el código.
  - **Mejora sugerida:** Extraer un cliente/adaptador de autenticación inyectable que reciba `baseUrl` por configuración y devuelva tipos propios de dominio.

- **[Alto] Mutación lateral del objeto de entrada y violación de CQS** (`sessionHelper.ts:L9`)
  - **Code smell:** Efecto secundario oculto (*Side Effect* en función que parece consulta).
  - **Principio / Referente:** *Command-Query Separation* — Robert C. Martin.
  - **Impacto:** `handleUser` promete verificar/obtener usuario pero muta `d.lastSeen` silenciosamente.
  - **Mejora sugerida:** Devolver un resultado inmutable con la nueva marca temporal o delegar explícitamente la actualización de `lastSeen` al módulo dueño de la sesión.

- **[Medio] *Flag Argument* (`force`) y parámetro `d: any`** (`sessionHelper.ts:L4`)
  - **Code smell:** *Flag Argument* (Fowler) + ausencia de contrato de tipos.
  - **Principio / Referente:** Nombres con intención y firmas claras — Martin / Fowler.
  - **Impacto:** `handleUser(user, true)` no comunica qué fuerza `true` ni qué estructura mínima necesita `d`.
  - **Mejora sugerida:** Tipar la entrada (`{ readonly token: string }`) y reemplazar `force: boolean` por un objeto de opciones explícito (`{ bypassCache?: boolean }`) o dos métodos si tienen semántica distinta.

- **[Bajo] Nombre genérico (`sessionHelper.ts`, `handleUser`, `d`) y comentario narrativo** (`sessionHelper.ts:L4-L5`)
  - **Code smell:** Palabras ruido (`Helper`, `handle`) + *Narrative Comment* (`// verificar si hay token`).
  - **Principio / Referente:** Nombrado con intención — Robert C. Martin.
  - **Mejora sugerida:** Renombrar a `authSessionClient.ts` y `verifySessionToken(...)`, eliminando el comentario redundante.

---

### 3. Plan de Refactor Incremental

1. **Red de seguridad:** Confirmar quiénes llaman a `handleUser` y cubrir con 3 tests de caracterización (token válido, token rechazado, error de red).
2. **Aislar frontera:** Crear `AuthSessionClient` con configuración de URL y traducción explícita de errores HTTP/red.
3. **Limpiar contrato:** Tipar entrada/salida con un tipo discriminado (`SessionVerificationResult`), eliminar mutación oculta de `d.lastSeen` y renombrar el archivo/función.
4. **Verificar:** Correr suite de tests y `tsc --noEmit`.

> **Nota de prudencia:** No recomiendo crear interfaces o abstracciones extra más allá de `AuthSessionClient` mientras exista un único proveedor de autenticación.
> ¿Quieres que revise primero los archivos que llaman a `handleUser` o que aplique ya este refactor paso a paso?
