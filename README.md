# skills-404-complement

Catálogo portable y versionado de **Agent Skills** (`SKILL.md`) para asistir a cualquier LLM (Antigravity, Kilo Code, Claude Code, Cursor, ChatGPT o Gemini) en arquitectura, *clean code*, diseño UI invariante y validación QA de producción.

---

## Catálogo de Skills

| Categoría | Skill | Versión | Estado | Propósito | Ubicación |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **ARQUITECTURE** | `clean-code` | `1.0.0` | Activa | Auditar *code smells*, organizar módulos y refactorizar de forma segura con TypeScript. | [ARQUITECTURE/CleanCode/SKILL.md](./ARQUITECTURE/CleanCode/SKILL.md) |
| **ARQUITECTURE** | `hexagonal-architecture` | `1.0.0` | Activa | Diseñar y migrar módulos con *Ports & Adapters* y principios SOLID. | [ARQUITECTURE/HexagonalArchitecture/SKILL.md](./ARQUITECTURE/HexagonalArchitecture/SKILL.md) |
| **QA-VALIDATOR** | `qa-validator` | `1.0.0` | Activa | Quality Gate de comportamiento, eficiencia FE/BE/Datos, seguridad OWASP, resiliencia y OpenTelemetry. | [QA-VALIDATOR/SKILL.md](./QA-VALIDATOR/SKILL.md) |
| **UI** | `ui-engineering-rules` | `1.0.0` | Activa | Invariantes universales de alineación, botones, tipografía, formularios, contraste WCAG 2.2 AA y semántica DOM. | [UI/SKILL.md](./UI/SKILL.md) |
| **REQUIREMENTS** | *Por definir* | — | Reservada | Refinamiento de HDU, criterios de aceptación y modelado de casos borde. | [REQUIREMENTS/](./REQUIREMENTS/) |

Historial completo de versiones en [CHANGELOG.md](./CHANGELOG.md).

---

## Cómo se combinan las skills en el ciclo de vida

1. **Al construir una feature nueva:**
   - Usa **`hexagonal-architecture`** para separar dominio, casos de uso, puertos y adaptadores (aplicando SOLID).
   - Usa **`clean-code`** para escribir funciones claras, nombres con intención y manejo limpio de errores.
   - Usa **`ui-engineering-rules`** cuando la feature incluya interfaz visual (grilla 8px, targets, jerarquía de botones, tipografía, formularios y cero IDs/etiquetas duplicadas).
   - Usa **`qa-validator`** (modo preventivo) para asegurar que el endpoint, las queries y la carga de UI cumplan estándares de eficiencia, seguridad y resiliencia desde el diseño.
2. **Al refactorizar código existente:**
   - Usa **`clean-code`**, **`hexagonal-architecture`** o **`ui-engineering-rules`** según la capa que estés limpiando.
   - Pasa **`qa-validator`** al finalizar para verificar que el refactor no introdujo consultas N+1, pérdida de timeouts ni regresiones de rendimiento o seguridad.
3. **Antes de abrir PR / pasar a producción (Quality Gate):**
   - Ejecuta **`ui-engineering-rules`** (si hubo cambios visuales/DOM) + **`qa-validator`** para emitir veredicto (`PASS` / `PASS WITH WARNINGS` / `BLOCK`).

### Dónde vive cada tipo de regla (sin solapamientos)

| Tipo de regla | Skill responsable | Ejemplos concretos |
| :--- | :--- | :--- |
| **Legibilidad micro y refactor seguro** | `clean-code` | Nombrado, *guard clauses*, *split phase*, *deep modules*, *code smells*, *Two Hats*. |
| **Capas, límites y SOLID** | `hexagonal-architecture` | Regla de dependencia, puertos pequeños (ISP/DIP), decoradores (OCP/LSP), *Composition Root*. |
| **Visual / Experiencia** | `ui-engineering-rules` | Grilla 8px/4px, alineación óptica ±1px, radios concéntricos, targets 24/44/48px, 1 Primary CTA, tipografía `65ch` / `tabular-nums`, contraste WCAG 2.2 AA, cero `id`/etiquetas duplicadas. |
| **Eficiencia, seguridad, resiliencia y QA** | `qa-validator` | WebP/AVIF, lazy loading, skeletons, debounce, Optimistic UI, streaming IA, SSR/SSG, code splitting, virtualización, CDN, paginación, N+1, índices, pool de conexiones, Redis, colas, Brotli/Gzip, timeouts, OpenTelemetry, OWASP, limpieza de dependencias sin usar. |
| **Levantamiento de HDU** *(futura)* | `REQUIREMENTS` | Historias de usuario, criterios de aceptación y modelado de casos borde. |

---

## Cómo usar estas skills en Antigravity, Kilo Code y otros LLMs

Todas las skills siguen el estándar abierto de **Agent Skills** con *Progressive Disclosure* (`SKILL.md` conciso + carga selectiva de `references/` y `examples/`). Como cada herramienta escanea **un solo nivel de profundidad** buscando `<carpeta>/SKILL.md`, expón las 4 carpetas finales (`ARQUITECTURE/CleanCode`, `ARQUITECTURE/HexagonalArchitecture`, `QA-VALIDATOR` y `UI`).

### 1. Antigravity (global para todos tus proyectos)
Registra el repositorio clonado en `~/.gemini/config/skills.json` apuntando a `<RUTA_AL_REPO>/skills-404-complement`:

```json
{
  "entries": [
    {
      "path": "<RUTA_AL_REPO>/skills-404-complement",
      "exclude": ["ARQUITECTURE", "REQUIREMENTS", "_TEMPLATE"]
    },
    {
      "path": "<RUTA_AL_REPO>/skills-404-complement/ARQUITECTURE"
    }
  ]
}
```

### 2. Kilo Code (global para todos tus proyectos)
Crea enlaces simbólicos en `~/.kilocode/skills/` hacia el repositorio clonado:

```bash
mkdir -p ~/.kilocode/skills
ln -s <RUTA_AL_REPO>/skills-404-complement/ARQUITECTURE/CleanCode ~/.kilocode/skills/clean-code
ln -s <RUTA_AL_REPO>/skills-404-complement/ARQUITECTURE/HexagonalArchitecture ~/.kilocode/skills/hexagonal-architecture
ln -s <RUTA_AL_REPO>/skills-404-complement/QA-VALIDATOR ~/.kilocode/skills/qa-validator
ln -s <RUTA_AL_REPO>/skills-404-complement/UI ~/.kilocode/skills/ui-engineering-rules
```

### 3. Por proyecto (compartido para Antigravity, Kilo Code y otros agentes)
En la raíz de cualquier repositorio de software, enlaza o copia las 4 carpetas dentro de `.agents/skills/`:
- `.agents/skills/clean-code`
- `.agents/skills/hexagonal-architecture`
- `.agents/skills/qa-validator`
- `.agents/skills/ui-engineering-rules`

*(En Claude Code usa `.claude/skills/`; en Cursor referencia `@SKILL.md`; en ChatGPT o Gemini Web adjunta `SKILL.md` + solo el archivo puntual de `references/` que necesites.)*

---

## Estándares de Calidad y Control de Versiones

1. **SRP (Single Responsibility):** Cada skill y cada archivo de `references/` tiene una única responsabilidad clara.
2. **LSP (Liskov Substitution):** Todos los `SKILL.md` comparten exactamente las mismas **6 secciones H2**, escala de severidades (`Crítico`, `Alto`, `Medio`, `Bajo`) y veredictos (`PASS`, `PASS WITH WARNINGS`, `BLOCK`).
3. **ISP (Interface Segregation):** Carga selectiva de `references/` según la superficie tocada.
4. **DIP / Cero Duplicación:** Cada regla tiene un único dueño y se enlaza sin duplicar contenido.
5. **Higiene técnica estricta:** Cero emojis, cero `any` en código limpio y comentarios breves (≤ 1 línea) solo para explicar el *por qué*.
6. **Versionado SemVer:** Cada `SKILL.md` declara `metadata.version` en su frontmatter YAML y cada release del repositorio se etiqueta con tags Git (`v1.0.0`) sincronizados con [CHANGELOG.md](./CHANGELOG.md).

---

## Cómo crear una nueva skill

1. Crea una carpeta dentro de la categoría correspondiente.
2. Copia la plantilla base desde [./_TEMPLATE/SKILL_TEMPLATE.md](./_TEMPLATE/SKILL_TEMPLATE.md) como `SKILL.md`.
3. Completa `metadata.version: "1.0.0"`, `name` en kebab-case y `description` en tercera persona.
4. Separa la teoría densa en `references/` (un tema por archivo) y agrega ejemplos Antes/Después en `examples/`.
5. Registra la nueva skill en este `README.md` y en [CHANGELOG.md](./CHANGELOG.md).
