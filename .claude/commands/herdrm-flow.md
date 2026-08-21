---
description: Orquesta UN issue de herdrm de punta a punta — apply vigilado, gate local, PR limpio a upstream
argument-hint: <número de issue>
---

# /herdrm-flow — un issue, de punta a punta

Issue: **$ARGUMENTS**

Eres el **PM**. Topología estrella: los workers son subagentes o procesos, **nunca se hablan entre
sí**, todo pasa por ti. Verificas el `git diff` real; **jamás** el reporte de un builder.

## Paso 0 — Router (fail-safe)
**Un número de issue SIEMPRE es del fork `alejodelosrios/herdrm`, nunca de upstream.** Léelo con
`gh issue view <n> --repo alejodelosrios/herdrm`. Upstream solo se consulta como referencia
(duplicados, PRs abiertos) y para abrir el PR final.
Si te pasaron **≥2 issues**, PARA y redirige a `/herdrm-fleet`. Este comando corre uno.
Lee el issue completo (body + comentarios). Si no tiene `ready-for-agent` ni cumple el contrato de
enriquecimiento, PARA y manda a `/herdrm-research` — no lo enriquezcas al vuelo.

## Paso 1 — Alcance y corte YAGNI → GATE HUMANO
Resume: qué se construye, file-set exacto, qué queda EXCLUIDO, criterios y cuáles son `manual-gate`.
Cruza contra `gh pr list --repo missuo/herdrm` por si upstream ya lo está haciendo.
Escribe la especificación en `.specs/<slug>.md` copiada de `.specs/TEMPLATE.md`: **proposal**
(problema + propuesta + EXCLUIDO), **design** (file-set exacto y prohibiciones), **spec Gherkin**
(un `Scenario` por criterio, con su verificación y cuáles son `manual-gate`) y **DoD**. `.specs/`
está gitignored: es andamiaje, no viaja al repo. Sin esta especificación no se lanza ningún builder.

**DETENTE** por aprobación de la especificación. (En fleet este gate lo aprueba el orquestador contra el issue.)

## Paso 2 — SIEMPRE: marcar el issue en curso
También si te saltaste el aislamiento porque venías de un fleet con rama ya creada.
`gh issue edit` puede devolver 0 sin mutar → **relee y confirma**.

## Paso 3 — Aislar: SIEMPRE en tu propio worktree

**Nunca `git switch` en el checkout principal.** No es tuyo: otra sesión puede estar dentro, y un
`switch` le cambia el árbol bajo los pies **sin error para ninguna de las dos**. Medido el
2026-08-19: dos sesiones en `~/Sites/herdrm`, una se llevó el checkout a la rama de la otra a mitad
de faena. Reincidencia el 2026-08-21 en el #8: el PM hizo `switch` en el principal teniendo otra
sesión viva sobre el repo — salió bien por suerte, no por método.

Censo antes de tocar nada — barato y obligatorio:

```bash
git worktree list                      # ¿quién tiene qué?
git status --short                     # ¿árbol ajeno sucio? entonces NO es tuyo
git reflog -5 --format='%gd %gs'       # ¿alguien movió HEAD hace minutos?
```
Y `ListAgents`: una sesión **busy** sobre este repo es una sesión que está escribiendo.

Aísla con worktree, no con switch:

```bash
git fetch origin
git worktree add ../herdrm-<slug> -b <feat|fix>/<slug> develop
cd ../herdrm-<slug>
xcodegen generate     # OBLIGATORIO: el worktree no trae .xcodeproj
```

El worktree NO trae `HerdrM.xcodeproj` ni `Sources/HerdrM/Info.plist` (gitignored, los genera
xcodegen) → **corre `xcodegen generate` antes de cualquier build** o el gate revienta. `.specs/`
tampoco viaja al worktree (gitignored) → copia `.specs/<slug>.md` al worktree antes del Apply. No hay
`.env` que copiar; el cert vive en el keychain y está disponible en cualquier worktree.

En fleet el worktree lo crea el orquestador (`herdr worktree add`) y te lo pasa hecho: no crees otro.

## Paso 4 — Apply: builders en background, NUNCA en foreground

El brief **es** `.specs/<slug>.md` — nada más. No le pases el issue crudo ni contexto extra: el
margen de acción de un modelo barato es exactamente lo que no está escrito. Copia:
`cp .specs/<slug>.md .swarm/brief-<slug>.md` y añade al final la línea:
`Si algo no está en esta especificación, NO lo hagas: para y reporta.`
Lanza el builder **en background** y **vigílalo**. Un `opencode run` en foreground es ciego: headless
no reporta `agent_status`, y te quedas media hora mirando la nada.

```bash
cd <worktree>
touch .swarm/apply-<slug>.start
opencode run --agent swift-builder --model opencode-go/kimi-k2.7-code \
  "$(cat .swarm/brief-<slug>.md)" > .swarm/apply-<slug>.log 2>&1
```
…lanzado con `run_in_background: true`, y **en el mismo turno** arma el watcher con `Monitor`:

```bash
LOG=.swarm/apply-<slug>.log; last=0; silent=0
while true; do
  sz=$(wc -c < "$LOG" 2>/dev/null || echo 0)
  touched=$(find Sources Packages -name '*.swift' -newer "$LOG.start" 2>/dev/null | wc -l | tr -d ' ')
  if [ "$sz" -gt "$last" ]; then echo "progreso: ${sz}B log · ${touched} .swift tocados"; last=$sz; silent=0
  else silent=$((silent+30)); fi
  grep -qiE 'error:|rate.?limit|requires explicit opt in|unauthor' "$LOG" && { echo "FALLO: $(grep -m1 -iE 'error:|rate.?limit|opt in' "$LOG")"; break; }
  pgrep -f 'opencode run' >/dev/null || { echo "FIN: el proceso salió"; break; }
  [ "$silent" -ge 240 ] && { echo "HOMBRE-MUERTO: 4min sin progreso, matando"; pkill -f 'opencode run'; break; }
  sleep 30
done
```

Reglas del watcher: va **por nivel, no por flanco** (un builder que pasa de un estado a otro entre
sondeos no genera transición y te deja ciego), cubre **fallo además de progreso** (el silencio no es
éxito), y lleva **interruptor de hombre muerto** a los 4 min sin ninguna de las tres señales (log,
archivos, proceso vivo).

Fan-out solo con **file-sets disjuntos**. `ContentView.swift` y `AppModel.swift` son hotspots: los
edita **un solo** builder, o tú. OpenCode no bloquea archivos entre procesos → el segundo pisa al
primero **sin error**.

## Paso 5 — Gate de compilación INMEDIATO (la mitigación del modelo externo)

Tras cada Apply, antes de cualquier otra fase:
```bash
xcodegen generate && xcodebuild -project HerdrM.xcodeproj -scheme HerdrM -configuration Debug \
  -derivedDataPath build build -skipPackagePluginValidation \
  CODE_SIGN_IDENTITY="Apple Development" DEVELOPMENT_TEAM=D2HZ8U62PA 2>&1 | grep -E 'error:|BUILD'
```
- No compila → devuélvele **el error textual** al builder. **Máximo 2 reintentos.**
- 3er fallo → **fallback al subagente Claude `swift-builder`** (Sonnet). Anota el fallo en
  `.swarm/lessons-learned.md` con el modelo y la clase de error: eso es el benchmark real del motor.
- Nunca `CODE_SIGN_IDENTITY="-"`: compila pero rompe UserNotifications en silencio.

## Paso 6 — QA (gate mecánico, sin mandato abierto)
`cd Packages/HerdrKit && swift test`. Si tocó `SSHTunnel`/`Device`/`SSHCredentialStore`, además
`HERDRM_E2E_SSH_TARGET=kupavo@srv1759591 swift test --filter RemoteSSHTests`.
Los suites Local* necesitan un **herdr corriendo**. QA recortado al gate: sin exploración libre — un
QA con mandato abierto costó 235k tokens con rendimiento cero.

## Paso 7 — Auditoría adversaria → binario
Lanza el agente `auditor` (Opus) con el `git diff` completo **y `.specs/<slug>.md`**: audita contra
la especificación, no contra su criterio. Diff fuera del file-set de §2 = DENEGADO automático. Veredicto **APROBADO/DENEGADO**, máx 2
iteraciones. **Reposo sin veredicto = FALLO, no aprobado**: un subagente que muere mudo pidiendo un
permiso no es una aprobación.

## Paso 8 — Gate local (no hay CI de GitHub, por diseño)
Las tres en verde, medidas por ti, no reportadas por nadie: `xcodegen` · build Debug · `swift test`.
Sin las tres no se mergea a `develop`. Esta es la regla que sustituye al branch protection.

## Paso 9 — Cerrar criterios con un NÚMERO
`grep -cE '^- \[ \]'` sobre body **y comentarios**. Cada casilla restante justificada por escrito en
el issue. **Publicar una tabla en prosa NO sustituye marcar las casillas** — es el fallo que se comete
creyendo cumplir la regla. Los `manual-gate` quedan en blanco a propósito y dices quién los cierra.

## Paso 10 — Merge a develop
`git switch develop && git merge --no-ff <rama>`. Es tu rama de integración: no requiere PR.

## Paso 11 — Publish: el PR limpio a upstream → GATE HUMANO
`.claude/`, `.swarm/` **y `.opencode/`** viven commiteados en `main`/`develop`, así que **un PR
cortado de `develop` arrastraría el enjambre** — medido: 720 líneas de enjambre por 2 de código real.
Los tres directorios se excluyen; olvidar `.opencode/` fue un bug real de este diseño, cazado por el
censo. La rama de PR se genera:

```bash
git fetch upstream
git switch -c pr/<slug> upstream/main
git diff develop~1..develop -- . ':(exclude).claude' ':(exclude).swarm' ':(exclude).opencode' | git apply
git add -A && git commit   # Conventional Commits, minúsculas, scope opcional
# censo de contaminacion: DEBE dar 0
git diff --name-only upstream/main | grep -cE '^\.claude/|^\.swarm/|^\.opencode/'
```
Añade el bullet a `CHANGELOG.md` bajo `## [Unreleased]` — el CI de release lo extrae o truena; no
inventes un heading de versión, solo el maintainer taggea.
Antes de abrir el PR, dos `grep` que missuo tuvo que arreglar a mano en el #8 — comentarios en
español y el marcador de la sesión colados en el fuente de upstream:

```bash
# comentarios no ingleses en el codigo que viaja: DEBE dar 0
git diff upstream/main --  'Sources/*' 'Packages/*' | grep '^+' | grep -nE '[áéíóúñ¿¡]|// (el|la|los|las|que|si|para|no) ' | wc -l
# nombres en clave de sesion o modo: DEBE dar 0
git diff upstream/main -- 'Sources/*' 'Packages/*' | grep '^+' | grep -cE 'ponytail|swarm|enjambre'
```

**DETENTE** por aprobación del humano antes de `gh pr create --repo missuo/herdrm --base main`.

## Paso 11.5 — Reconciliar la especificación con lo que de verdad se construyó

**Antes de cerrar, la `.specs/<slug>.md` tiene que describir el código que existe, no el que
imaginaste en el Paso 1.** Un flujo largo cambia de diseño a mitad de vuelo: el auditor tumba
un mecanismo, el gate manual descubre que un atajo no llega, un hallazgo obliga a invertir un
criterio. Si eso no vuelve a la spec, queda un documento que miente — y en una serie por fases
la spec de la fase 1 es el punto de partida de la 2.

Recorre el diff final contra §2 y §3 y pregunta, decisión por decisión:

- ¿Algún **mecanismo** de §2 se cambió por otro? (ejemplo real: `@AppStorage` en un
  `ObservableObject` → `@Published` con `didSet`; cuatro ítems de menú con atajo dinámico →
  ocho fijos con `.disabled` por eje). Reescribe el bullet con el mecanismo nuevo **y por qué
  el anterior no servía** — ese "por qué" es lo que evita que el siguiente lo reintente.
- ¿Alguna instrucción tuya resultó **estar mal**? Dilo en la spec marcándolo como corrección,
  no la borres en silencio. (Ejemplo real: "aplica `.opacity` solo cuando hay split" era un
  error: el `if` crea la rama de `_ConditionalContent` que destruye la identidad de la vista.)
- ¿Cambió algún **criterio** por decisión del humano en el gate manual? Invierte el `Scenario`
  y déjalo anotado como desvío aprobado, con fecha.
- ¿Aparecieron **manual-gates nuevos** que no estaban en §3? Añádelos: son los que la próxima
  fase va a heredar.
- ¿Hay algo en el EXCLUIDO que ya **no** aplique, o algo nuevo que deba excluirse?

Lo mismo con el **issue** (que sí es público) y con `CLAUDE.md` si el cambio movió algo que
describe. Y si lo aprendido es del enjambre y no del producto —un watcher con un patrón que
da falsos positivos, un agente mal configurado, una receta de firma caducada— va a
`.swarm/lessons-learned.md` y al comando correspondiente, no a la spec.

Regla práctica: si al leer la spec un builder nuevo escribiría algo distinto de lo que hay
en el repo, la spec está desactualizada y el paso no está hecho.

### Y en el mismo momento: veredicto de auditoría sobre el código FINAL

**No se cierra nada sin un veredicto de auditoría del código que se va a mergear.** No vale el
veredicto de una ronda anterior: si después de auditar entró un solo commit —incluido el
arreglo del propio hallazgo del auditor, o cualquier cosa salida del gate manual— ese código
**no está auditado** y hay que volver a pasarlo. Medido en el #9: la primera ronda dio DENEGADO
y después entraron dos commits de código nuevo (el eje como focused value con ocho ítems de
menú, y el foco tras el cierre del sheet) que nadie había revisado.

Va aquí, junto a la reconciliación de la spec, y por el mismo motivo: son las dos cosas que se
desincronizan cuando el flujo se alarga. El orden es: reconcilia la spec → lanza la auditoría
final contra esa spec ya corregida → solo con **APROBADO** sigues al merge.

Checklist del cierre, las tres a la vez:
- [ ] `.specs/<slug>.md` describe el código que existe
- [ ] veredicto **APROBADO** sobre el diff final completo (`git diff develop..HEAD`), no sobre
      un estado intermedio
- [ ] los `manual-gate` cerrados por el humano, sobre un binario **verificado** como el nuevo

Si el veredicto es DENEGADO, arreglas y **vuelves a auditar**: el ciclo no se cierra con "ya lo
arreglé". Y recuerda que reposo sin veredicto no es aprobación — ve a buscar el resultado.

## Paso 12 — Cierre
Comentario en el issue con el PR, qué se marcó con qué evidencia, y qué queda sin marcar y quién lo
cierra. La rama remota `pr/*` **sobrevive** mientras el PR esté abierto. Cuando missuo mergee (squash):
`git rebase upstream/main` y **el squash gana** — tu commit local se descarta, no se preserva.
Deja el repo limpio: `git branch -d` de las ramas mergeadas (nunca `-D` a ciegas).
Anota en `.swarm/lessons-learned.md`.
