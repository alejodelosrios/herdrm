---
description: Orquesta ≥2 issues de herdrm en paralelo, en worktrees de Herdr, por olas
argument-hint: <números de issue separados por espacio>
---

# /herdrm-fleet — paralelismo multi-issue

Issues: **$ARGUMENTS**

Eres el **orquestador**. Lanzas un `/herdrm-flow` por issue como **PM hijo** (Sonnet) en su worktree
de Herdr. Un solo issue → usa `/herdrm-flow`.

**Los issues son del fork `alejodelosrios/herdrm`.** Upstream (`missuo/herdrm`) es solo lectura de
referencia y destino del PR final.

## Precondición DURA
Todos los issues **completamente enriquecidos** y con `ready-for-agent`. Es lo que te habilita a
aprobar los gates de alcance de los hijos sin molestar al humano. Alguno a medio enriquecer → PARA y
manda ese a `/herdrm-research`. Sin esto, no puedes bajar los hijos a Sonnet.

## Reparto en olas — **máximo 3**
Dos límites se suman: los builders comparten el endpoint de OpenCode Go (rate-limit compartido; más
teams **no** es speedup lineal) y `xcodebuild` es pesado en CPU/disco — varios en paralelo en la
MacBook se estorban entre sí y también compiten por el caché de SwiftPM.
Reparte para que issues que toquen el **mismo archivo** caigan en **olas distintas**. Hotspots:
`Sources/HerdrM/ContentView.swift`, `Sources/HerdrM/AppModel.swift`. Cierra una ola antes de abrir la
siguiente.

## Lanzamiento — nunca fire-and-forget
Por cada issue: `herdr worktree add` desde `develop`, y en el pane del hijo:
1. `herdr pane run "$PANE" '/herdrm-flow <N>'`
2. **`herdr pane send-keys "$PANE" Enter`** — inyectar un slash-command abre el dropdown de
   autocompletado que **se traga el Enter**.
3. **`herdr pane read "$PANE"`** para verificar que ya no está colgado en el prompt. Reintenta el
   Enter si sigue ahí. Solo entonces pasas al siguiente team.

## En el PRIMER mensaje al hijo, revócale lo que su brief le permite
Su `/herdrm-flow` standalone le deja cosas que en fleet están **prohibidas**. Dilo explícito o hará
lo que dice su brief:
- **NO mergea a `develop`** (lo haces tú, tras verificar).
- **NO lanza al auditor** (lo lanzas tú, con el file-set de sus hermanos, para cazar cruces de frontera).
- **NO genera la rama `pr/*`** ni abre PR upstream.
- **NO escribe `.swarm/lessons-learned.md`** (escritor único: tú).
- Su gate de alcance te lo reporta **a ti**, no al humano: te manda `.specs/<slug>.md` y tú lo
  apruebas contra el issue. Sin especificación aprobada, no lanza builder.

## Vigilancia
Listener **persistente** toda la corrida que dispara cuando un hijo te **pregunta** — gate, menú o
idle en **cualquier** paso, no solo al abrir PR. Distingue "espera a un subagente" (activo, tiene
ticker) de "te espera a TI" (idle accionable). Va **por nivel** con contador de silencio de tres
fuentes (pane, archivos del hijo, `agent_status`) y lo mata a los ~4 min.
Vigila además **señal de producto** (commits en su rama, PRs): un hijo desviado se ve idéntico a uno
que va bien, y un gate que se dispara con **el reporte del hijo** se salta solo y en silencio.
Vigila también el avance de `develop`: un merge a mitad de vuelo colisiona con un team del mismo dominio.
No mates un QA lento por reloj — mátalo por **delta de progreso**.

## Auditoría de tablero al arrancar
En cuanto los N hijos pasen su fase de aislamiento, lee el estado de **todos** los issues del lote en
**una** llamada y compáralo con lo esperado. Si a alguno le falta la transición, **muévelo tú** en vez
de interrumpir a un hijo que ya está implementando.

## Permisos
El wall-clock se lo comen los prompts de shell, no el trabajo (~40 aprobaciones una por una fue la
mayor fuente de lentitud medida). Allowlistea en el worktree del hijo las clases de **solo lectura**
que sabes que se repiten; **lee una por una** las de escritura, muerte y merge.

## Cierre — por CADA team, no solo los huérfanos
Un hijo **no puede** cerrar el workspace donde vive ni matar su propio proceso → aunque borre su
worktree, Herdr conserva workspace + proceso idle zombie.
1. `herdr worktree remove --workspace <id> --force` por **cada** team.
2. `git worktree prune`, y **cuenta contra `git worktree list`**, no contra tu estado: el flow crea
   worktrees efímeros que no pasan por ti. El criterio de borrado es **pertenencia al lote**, nunca
   "no es el principal".
3. `herdr workspace list` para confirmar que no quedó ninguno.
4. Repo principal impecable: `git switch develop && git pull --ff-only`, `git branch -d` de las ramas
   del lote (seguro: solo borra mergeadas), árbol limpio.
5. Las ramas `pr/*` con PR abierto en upstream **sobreviven** — se borran cuando missuo mergee.
