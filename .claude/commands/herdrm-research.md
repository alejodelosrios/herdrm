---
description: Investiga un objetivo en herdrm y publica issues enriquecidos, ejecutables sin reinvestigar
argument-hint: <objetivo — bug, feature, módulo a auditar, o URL de referencia>
---

# /herdrm-research — investigación → issues enjambre-listos

Objetivo: **$ARGUMENTS**

Produce un **expediente**: informe de arquitecto + issues tan enriquecidos que un builder de modelo
externo los ejecuta sin volver a investigar. Es la precondición dura de `/herdrm-fleet`.

Este comando compone dos fuentes instaladas — úsalas, no las reimplementes:
- **`mattpocock:research`** — investigar contra **fuentes primarias** (docs oficiales, código fuente,
  specs, APIs de primera mano; nunca un resumen ajeno de ellas), siguiendo cada afirmación hasta la
  fuente que la posee. Lánzalo **en background** para no bloquearte mientras lee.
- **`mattpocock:to-tickets`** — el corte en tracer bullets, los **blocking edges**, el quiz al usuario
  y el trabajo por frontera. Adopta su proceso tal cual salvo la divergencia declarada abajo.

> ### Divergencia declarada respecto a `to-tickets`
> Pocock dice *"avoid specific file paths or code snippets — they go stale fast"*, porque asume
> ejecución con `/implement`: un agente capaz, contexto fresco, que explora el repo por su cuenta.
> **Aquí el ejecutor es un builder de modelo externo barato** (Kimi/MiMo) que, sin el snippet,
> alucina la API de SwiftUI y quema la ola. Así que en este repo **el `file:line` y el snippet son
> obligatorios**, y su staleness se paga con las dos contramedidas del contrato: el sello
> `## Verificación (<fecha>)` y la regla de que si el árbol se movió desde ese sello, el issue se
> **re-verifica antes de ejecutarse**, no se ejecuta a ciegas.
> El resto de `to-tickets` (rebanadas verticales, edges, prefactor primero, expand–contract) aplica
> sin cambios.

Tracker: **GitHub Issues de `alejodelosrios/herdrm`** (tu fork). Nunca abras issues en `missuo/herdrm`
— ahí no tienes triage. Plantillas y labels los define `/herdrm-issue`, única fuente de verdad.

## Paso 0 — Modo y alcance (antes de gastar un tool call)

Fija objetivo + alcance cerrado + **modo**. Si el objetivo no deja claro el modo, **pregunta**.

| Modo | Cuándo | Kit |
|---|---|---|
| **Interno** *(default)* | módulo a auditar, feature a descomponer, código muerto | grep dirigido · `CLAUDE.md` del autor · ledgers |
| **Bug** | algo está roto en la app | loop rojo (abajo) + kit interno · `mattpocock:diagnosing-bugs` |
| **Referencia externa** | "que se parezca a `<app/url>`" | detectar el motor ANTES de cargar skill; parámetros, no adjetivos |
| **Protocolo herdr** | toca el socket API, panes, eventos | `herdr api`, `herdr --skill`, el socket en vivo — **fuente primaria** |
| **Docs / API externa** | SwiftUI/SwiftTerm/Sparkle, versión de una API | MCP context7 — **nunca de memoria** |

**La premisa del humano es una HIPÓTESIS.** Si el objetivo afirma algo del código ("el bug está en
X", "esto ya existe"), verifícalo contra HEAD antes de construir encima. Si no se sostiene, va en la
PRIMERA línea del informe: eso vale más que cualquier hallazgo.

## Paso 1 — Verify-before-enrich (fuentes primarias)

Nada entra al expediente sin **`file:line` verificado en HEAD en esta corrida**. No hay CodeGraph en
este repo → grep dirigido y el mapa de `.claude/skills/herdrm-contrib/SKILL.md`.

- "puede pasar en más lugares" es **inaceptable**: enumera TODOS los sitios o declara el alcance cerrado.
- Swift: un símbolo se busca por declaración *y* por uso (`grep -rn 'showNewAgent' Sources/`); los
  `@State`/`@AppStorage` se leen junto a su `onAppear` o mientes sobre el valor inicial.
- Para el comportamiento de herdr, la fuente primaria es **el socket en vivo**, no la memoria ni el
  README: `herdr api`, y las notas ya verificadas del `CLAUDE.md` del autor (protocolo 19).
- Código muerto se propone **borrar**, no arreglar.
- Busca el hallazgo detrás del hallazgo: contrasta contra los patrones que el autor ya usa —
  `CLAUDE.md` es su brief; respétalo o justifica el desvío dentro del issue.

### Modo bug — el loop rojo antes de la hipótesis
Verificar que el código *dice* lo que crees no prueba que el defecto *ocurre*. Antes de enriquecer,
reprodúcelo en rojo: build Debug con la firma local, lanzar, ejecutar el gesto, capturar evidencia
(`Console.app`, `herdr pane read`, o un `print` temporal que se borra antes del issue). Sin rojo
reproducido el issue se marca `Requiere decisión` y lo dices.

## Paso 2 — Cruces obligatorios antes de escribir nada

1. **Issues ABIERTOS que toquen los mismos archivos** (`gh issue list --state open`). Dos entradas que
   nombren el mismo archivo son candidatas a duplicado **aunque los títulos no se parezcan**. Ante
   duplicado con fixes distintos, compara qué descubre cada uno del árbol real — no elijas por
   antigüedad ni por severidad declarada.
2. **Upstream**: `gh issue list --repo missuo/herdrm` y `gh pr list --repo missuo/herdrm`. Si missuo ya
   lo está haciendo, el issue **no se abre** — va a exclusiones. Es el cruce más barato que existe y
   es propio de contribuir a un repo ajeno.
3. **CHANGELOG `[Unreleased]`**: puede estar ya resuelto sin release.

## Paso 3 — Informe de arquitecto + quiz → DETENER

El usuario decide estructura y prioridades leyendo solo el informe:
- **Resultado primero** (una línea).
- Hallazgos **rankeados por severidad**, cada uno con su `file:line` verificado y la fuente que lo prueba.
- **Corte en tracer bullets** (reglas de `to-tickets`): cada issue una rebanada vertical
  demoable/verificable por sí sola, **dimensionada para caber en un contexto fresco**. La cantidad de
  **trabajo** decide 1 vs N issues, no la cantidad de hallazgos. El **prefactor va primero, como
  ticket propio** — "make the change easy, then make the easy change".
- **Blocking edges explícitos.** El paralelismo lo decide el **solapamiento de archivos**: file-sets
  disjuntos → corren en fleet; archivo compartido → edge de bloqueo. En este repo los hotspots son
  `Sources/HerdrM/ContentView.swift` (todos los sheets viven ahí) y `AppModel.swift` — dos issues que
  los toquen **no van en la misma ola**. Los edges cruzan también hacia issues ABIERTOS y sesiones
  activas del enjambre.
- **Wide refactor**: no lo fuerces en un tracer bullet → expand–contract (añade la forma nueva al
  lado de la vieja, migra en lotes por blast radius, contrae al final), cada lote su ticket.
- **Qué recomiendas EXCLUIR y por qué** (lente ponytail). Esta tabla **viaja al cuerpo del issue**, no
  se queda en el informe, o el builder la reabre y gasta la ola.

Luego **el quiz de `to-tickets`**, presentando el desglose numerado y preguntando explícitamente:
¿la granularidad es correcta (muy gruesa / muy fina)? ¿los blocking edges son los que de verdad
gatean? ¿algo se debe unir o partir? **Itera hasta que apruebe. No publiques sin aprobación.**

## Paso 4 — Publicar (en orden de dependencia)

Un issue por ticket, **nunca uno combinado**, publicados con los bloqueadores primero para que los
edges puedan citar `#números` reales. Usa la relación nativa de GitHub (sub-issues / "blocked by")
cuando exista; si no, la sección `## Blocked by`. Label **`ready-for-agent`** salvo instrucción
contraria: por construcción son agarrables por un agente. **No cierres ni modifiques el issue padre.**

Sobre las plantillas de `/herdrm-issue`, cada cuerpo lleva además:

- `## Verificación (<fecha>)` — lo confirmado contra HEAD con `file:line` frescos, los grep que
  cerraron el alcance, y la fuente primaria de cada afirmación externa.
- `## Fix propuesto` — **ejecutable tal cual** (ver la divergencia arriba): archivo:línea y el snippet
  cuando ahorre una decisión. Marca los valores que **se ven raros pero son deliberados** para que
  nadie los "corrija".
- `## Impacto en el PR upstream` — ¿missuo lo aceptaría? ¿toca `project.yml`/firma (entonces **no** va
  al PR)? ¿necesita entrada en `CHANGELOG [Unreleased]`?
- `## Ejecutabilidad (fleet/flow)` — `Auto-ejecutable` · `Gate de verificación manual` (todo lo visual
  de una app macOS lo es: alguien abre la app y mira) · `Requiere decisión`, más el grafo (`#A ∥ #B → #C`).
- `## Criterios de aceptación` — checkboxes **falsables**. "Se ve bien" no es criterio; **si no se
  puede fallar, no es criterio**. Aquí se traducen a: un grep con resultado esperado, `swift test`
  pasando, el build en verde, o **una observación concreta con el gesto exacto** ("con la app
  enfocada, ⌘N abre el sheet titulado 'New Agent'"). Los que exigen mirar la app **los marca el
  humano, nunca un agente**.
- `## Excluido` — la tabla de exclusiones.

Con los números reales en mano, edita los cuerpos publicados para que las referencias cruzadas usen
`#números` reales.

## Cierre
Reporta los issues publicados con su grafo, qué quedó excluido, y si el lote es apto para
`/herdrm-fleet` (≥2 auto-ejecutables con file-sets disjuntos) o va uno por uno con `/herdrm-flow`.
Se trabaja **la frontera**: cualquier ticket cuyos bloqueadores estén cerrados. Anota en
`.swarm/lessons-learned.md` lo que aprendiste del árbol y no del issue.
