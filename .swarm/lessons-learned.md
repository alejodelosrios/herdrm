# Lecciones aprendidas — enjambre herdrm

Escritor unico: el orquestador.

## 2026-08-19
- Los builders de `opencode run` en foreground son ciegos: headless no reporta `agent_status` y la sesion se queda colgada sin senal. Siempre background + watcher.
- `deepseek-v4-pro` y `-flash` de OpenCode Go estan tras un opt-in de hosting en China, a nivel cuenta. Descartados para no contaminar proyectos propietarios.
- El cableado de un agente opencode se verifica leyendo la linea de estado (`> agente · modelo`), no el auto-reporte del modelo. Verificado: swift-builder → kimi-k2.7-code, docs-writer → mimo-v2.5.
- El patron de atajos del autor (hidden Button + .keyboardShortcut en .background) NO sirve para teclas
  que el menu principal ya tiene: AppKit resuelve los key equivalents del menu ANTES del
  performKeyEquivalent de las vistas. `WindowGroup` regala File > New Window con Cmd-N.
- `SwiftTerm/Mac/MacTerminalView.swift:1567` consume todo evento con `.command` en keyDown sin
  propagar. Cualquier atajo nuevo se verifica CON EL FOCO DENTRO DEL TERMINAL, que es el estado
  normal de la app; probarlo con el sidebar enfocado da un falso verde.
- `focusedSceneValue` != `focusedValue`. El segundo exige que ESA vista tenga el foco; en esta app
  lo tiene el terminal, asi que rompe el caso principal.

## 2026-08-19 — issue #1 (⌘N / ⇧⌘N): el auditor murió mudo

**Fallo del enjambre, no del código.** El subagente `auditor` (Opus) entró en reposo dos veces
(`idle_notification`, `idleReason: available`) sin emitir veredicto, incluso tras un SendMessage
exigiéndolo en formato fijo y ofreciéndole declarar si le faltaba un permiso. Nunca contestó.

Aplicada la regla del flujo — **reposo sin veredicto = FALLO, no aprobado** — el PM (Opus) hizo la
auditoría adversaria él mismo en lugar de mergear con un silencio interpretado como visto bueno.
Esa regla se ganó el sueldo: sin ella, el merge habría pasado con cero revisión.

Pendiente de diseño: el `auditor` no tiene forma de reportar "me bloqueó un permiso". Vale la pena
que su prompt exija imprimir el veredicto como PRIMERA línea de su salida, antes de cualquier
lectura de archivos, y actualizarlo al final — así un reposo prematuro deja rastro utilizable.

**Del código, sin incidencias:** build Debug en verde y 36/36 tests al primer intento, con el fix
del issue aplicado tal cual venía especificado (el research dejó el diff al carácter). Cero
reintentos de builder — no se usó builder externo: tres ediciones exactas no pagan el arranque de
un proceso en background, y `ContentView.swift` es hotspot de un solo escritor.

## 2026-08-19 — research de copy-on-select: el cruce upstream valía más que el grep

**El Paso 2 del research (cruces) fue lo que salvó el issue, no la lectura de código.**
`gh issue list --repo missuo/herdrm --state all` destapó dos cosas que ninguna cantidad de grep
habría dado:

1. **missuo/herdrm#2** («鼠标无法选中字符») y la mitad de copia de **#5** son exactamente el síntoma
   que el usuario reportó, y **ya estaban arregladas en v0.2.2** vía el toggle de *Mouse reporting*.
   O sea: no había bug que arreglar. Ir directo a construir habría producido un issue para un
   problema resuelto hace dos versiones.
2. El **último comentario de @boonguan en #5 pide copy-on-select** explícitamente y nadie contestó.
   Eso convierte la feature de "capricho local" en "demanda upstream sin dueño" — y es la cita más
   fuerte que puede llevar el PR.

**Corrección de un dato heredado:** el cuerpo del issue #1 afirmaba que missuo/herdrm#5 fue «cerrado
COMPLETED sin commit en main», y lo repetí. Falso: la mitad de copia/selección se arregló en 0.2.2
(está en el CHANGELOG citando «(#2, #5)») y la de pegado se cerró como no reproducible **con el
reportero confirmando que le funcionaba**. Lección: un research previo del propio enjambre es
contexto, no fuente primaria — el issue upstream con sus comentarios sí lo es. Leerlo cuesta un
tool call.

**Trampa técnica cazada en el enriquecimiento, no en el gate:** `UserDefaults.bool(forKey:)`
devuelve `false` cuando la clave no existe, y `@AppStorage("k") = true` **no escribe** el default en
`UserDefaults` hasta que el usuario mueve el control. Un builder que escriba lo obvio entrega una
feature que compila, pasa el gate y **nunca copia nada** hasta que tocas el Toggle dos veces. El
issue prescribe `object(forKey:) as? Bool ?? true`, lo marca como deliberado, y le pone dos
criterios falsables (un grep que exige la forma correcta y otro que prohíbe la incorrecta) más un
manual-gate de instalación limpia. Esta clase de bug — "verde y muerto" — es la que el modelo
externo produce con más gusto.

**De paso, del árbol y no del issue:** el arrastre normal NO selecciona en panes de agente porque
los TUI piden mouse reporting; solo Shift+arrastre. Cualquier criterio de aceptación sobre
selección que no diga "con Shift" es un falso negativo esperando ocurrir.

## Issue #2 — teardown de túneles SSH al salir (PR upstream #11)

- **`gh` resolvía el default repo a `missuo/herdrm`, no al fork.** `/herdrm-flow 2` leyó el
  issue #2 de *upstream* (uno cerrado, en chino, sobre selección de texto) y estuvo a punto de
  abortar el flujo por "ya resuelto". Pasar `--repo alejodelosrios/herdrm` **siempre**, o correr
  `gh repo set-default alejodelosrios/herdrm`. El síntoma es traicionero porque el comando no
  falla: devuelve un issue real y coherente, solo que del repo equivocado.

- **Un issue enriquecido puede traer el fix mal y el gate de compilación no lo nota.** El issue
  especificaba `(NSApp.delegate as? AppDelegate)?.model = model`, pero con
  `@NSApplicationDelegateAdaptor` SwiftUI instala su propio proxy como `NSApp.delegate`: el cast
  da `nil` siempre. Build verde, 36 tests verdes, **fix inerte**. Lo cazó el auditor levantando
  una app mínima con la misma forma, no leyendo el diff. Lección de mecánica: para bugs de ciclo
  de vida, ni el build ni la suite son gate — el gate es la medición del comportamiento. El
  `research` que enriquece un issue debería marcar el cableado de un delegate como *no verificado*
  en vez de darlo por bueno.

- **El control negativo vale su tiempo.** Medir el binario *sin* el fix (túnel pasando a `ppid=1`
  y sobreviviendo) es lo que convierte "no veo huérfanos" en prueba. Sin él, un teardown que no
  corre y una app que nunca creó el túnel se ven idénticos.

- **Los worktrees comparten bundle id.** `osascript -e 'tell application "herdrm" to quit'`
  alcanzó a la app del worktree vecino (`herdrm-copy-on-select`), la cerró, y dejó su túnel
  huérfano — basura que luego yo mismo conté como fuga propia. Un bucle de 3 ciclos midiendo
  `pgrep`/conteos globales dio resultados sucios y hubo que rehacerlo **dirigido por PID**:
  capturar el pid de la app, el de su túnel hijo, y verificar *ese*. En una máquina donde corren
  varios flows a la vez, todo conteo global de procesos es ruido.

- **`git apply` del diff de develop contra `upstream/main` conflictúa por contexto de otro issue.**
  El #1 (⌘N) ya estaba en develop y tocaba los mismos archivos. `git apply -3` resuelve y deja
  marcadores donde hay que decidir; la resolución correcta es quedarse solo con lo del issue en
  curso (fuera el bloque `FocusedValues` y la entrada de CHANGELOG del #1). Verificar después con
  `git diff upstream/main | grep -c '<marca del otro issue>'` → 0.

## Issue #4 — tipografía gruesa del terminal (research, 2026-08-20)

- **La premisa del humano apuntaba a la tipografía; el árbol dijo renderer.** El picker de fuente ya
  existía y no arreglaba nada porque el engrosamiento lo mete macOS al rasterizar (font smoothing /
  stem dilation), no la familia. Confirmarlo costó un grep en SwiftTerm y le ahorró al issue una
  propuesta inútil ("añadan más fuentes").

- **SwiftTerm v1.20.0 trae APIs que herdrm no usa.** `fontSmoothing` y `lineSpacing` son `@objc open`
  en `extension TerminalView` (`AppleTerminalView.swift:271` y `:279`) y estaban ahí desde antes.
  Antes de proponer un fork o un PR upstream conviene leer el `extension` completo del dependency:
  la mitad de lo que uno quiere "añadir" ya existe con default conservador.

- **Asimetría macOS/iOS en SwiftTerm:** el `TerminalView` de iOS tiene
  `setFonts(normal:bold:italic:boldItalic:)`; el de macOS **no**, y `fontSet` es `internal`. Cualquier
  feature que necesite controlar la cara *bold* en macOS es PR a SwiftTerm, no trabajo de herdrm.

- **Medir la tinta convierte "se ve grueso" en dato.** Rasterizar la misma cadena a 2x con
  `setShouldSmoothFonts(true/false)` y promediar el canal da −10.6 %. Un script de 25 líneas con
  `swift file.swift` (sin proyecto, sin target) es suficiente y evita discutir de gustos en el issue.

- **`NSFont.Weight.light.rawValue != -0.4`** (vale `-0.4000000059604645`). Un `Picker` con `.tag(-0.4)`
  compila, corre y nunca casa la selección: el clásico "verde y muerto". Igual de importante: aplicar
  un trait de peso por `fontDescriptor` a Menlo/Monaco/Courier New es **no-op** — el peso solo se
  puede elegir en el mono del sistema.

- **El build local falla por firma, no por código.** `make build` usa la identidad Developer ID de MOE
  AI LLC, ausente en esta máquina. Para verificar que un snippet compila:
  `CODE_SIGN_IDENTITY="-" CODE_SIGN_STYLE=Manual DEVELOPMENT_TEAM=""`. Sin eso, un research honesto
  concluye "no compila" cuando el problema es el llavero.

- **Aplicar el parche completo, compilarlo y revertir** (`git checkout Sources/HerdrM/`) es el modo
  barato de cumplir el contrato de "ejecutable tal cual" sin ensuciar `develop`. El diff verificado se
  guarda en el scratchpad y viaja al cuerpo del issue.

## Issue #5 — foco del terminal al saltar de agente (research, 2026-08-20)

- **La premisa culpaba a ⌘K; el árbol dijo `.id()`.** El sheet de búsqueda es otra ventana y **no**
  toca el `firstResponder` de la principal (medido). Lo que rompe el foco es
  `.id("attach-\(entry.id)")` en `ContentView.swift:222`: SwiftUI destruye el
  `LocalProcessTerminalView` y crea uno nuevo, y un NSView recién nacido nunca es first responder.
  Consecuencia práctica: el bug NO es de ⌘K — es de **toda** ruta que cambie de agente (sidebar,
  notificación, ⌘N, reselección implícita). Un issue redactado solo para ⌘K habría dejado el 80 % vivo.

- **`grep -rn "FirstResponder" Sources/HerdrM/` → 0.** Un grep negativo sobre toda la superficie vale
  como prueba de alcance cerrado y cuesta un tool call. Úsalo antes de escribir "podría pasar en más
  sitios".

- **El arnés mínimo es el loop rojo de las apps de escritorio.** 60 líneas de `swift focus.swift`
  (NSViewRepresentable + `.id()` + sheet, imprimiendo `window.firstResponder` en tres tiempos) dieron
  rojo *y* verde en 4 segundos, sin GUI que automatizar y sin instrumentar la app real. Es el mismo
  truco que usó el auditor en el #2 y ahora tiene dos casos: sirve para foco, ciclo de vida y
  cualquier cosa que dependa de en qué momento AppKit hace su parte.

- **Del árbol y no del issue:** `becomeFirstResponder` de SwiftTerm no solo habilita el teclado —
  llama `terminal.setTerminalFocus(true)` y `caretView.updateCursorStyle()`
  (`Mac/MacTerminalView.swift:1313-1321`). Sin foco, el TUI *cree* que no está enfocado y dibuja el
  cursor hueco. Eso convierte un criterio de aceptación vago ("se puede escribir") en uno observable:
  el cursor sólido.

- **Trabajo sin commitear = blocking edge real.** El #4 estaba aplicado en el working tree (no en
  HEAD) tocando `TerminalView.swift`. Los `file:line` del issue nuevo se sellaron contra HEAD y el
  cuerpo dice explícitamente «anclá por nombre de función, no por número de línea». Revisar
  `git status` **antes** de sellar la verificación, no después.

## Issue #4 (thin strokes) — especificación cerrada estilo OpenSpec

- **`.specs/<slug>.md` es ahora el brief.** proposal → design con file-set cerrado → Gherkin → DoD
  con invariantes de `grep`. El builder barato (kimi-k2.7-code) aplicó los 3 archivos y el CHANGELOG
  **a la primera, sin reintento de compilación**. Es el primer apply del enjambre con 0 iteraciones
  de gate: el margen de acción cerrado es lo que lo compró.
- **`.specs/` va a `.git/info/exclude`, NO a `.gitignore`.** El `.gitignore` viaja al upstream;
  meterle andamiaje del enjambre es la misma contaminación que el censo de `.claude/`/`.swarm/`
  existe para cazar. `.git/info/exclude` es local y no aparece en ningún diff.
- **`opencode run --agent swift-builder` no encuentra el agente**: los subagentes viven en
  `.claude/agents/`, y opencode los busca en `.opencode/agent/`. Cae silenciosamente al agente
  `build` por defecto — funcionó porque la spec era el contrato, no el prompt del agente. Si el
  agente importara, este fallo sería invisible hasta el gate.
- **El builder omite lo que no es verificable por `grep`.** Se saltó los comentarios `///` que
  explican los cinco valores deliberados y el `.help()` del Picker: nada de eso estaba en el DoD.
  Los repuso el PM. Lección: lo que quieras que sobreviva, ponlo como invariante medible.
- **Un número de issue es del fork.** `gh repo set-default alejodelosrios/herdrm` fijado; upstream
  #4 es otro issue (tema del terminal, ya cerrado por missuo) y leerlo costó una vuelta entera.
- **El auditor entregó la auditoría pero no la envió** (se quedó idle sin `SendMessage`). La regla
  «reposo sin veredicto = FALLO» funcionó: se le reclamó y entregó APROBADO. No asumir que idle
  es aprobación sigue siendo la regla correcta.

## Sincronizar el fork tras un merge upstream (2026-08-20)

`git rebase upstream/main` NO sirve aquí. missuo mergea con **squash**: los commits que él
aceptó existen upstream con otro SHA y otro contenido, así que rebasar los reaplica y revienta
en conflicto contra su propia versión ya mergeada.

Receta que funcionó (de 13 commits divergentes a 2, sin conflictos):

1. `git branch backup/develop-pre-sync develop` — red de seguridad antes de tocar nada.
2. `git switch -c develop-sync upstream/main`.
3. `git checkout backup/develop-pre-sync -- .claude .swarm` + **un solo commit**. El historial
   del enjambre no se cherry-pickea: se reordena mal contra la base nueva (commits sucesivos
   sobre `lessons-learned.md` chocan entre sí) y a nadie le sirve esa historia. Colápsalo.
4. `git cherry-pick -x <commit de la rama pr/*>` para el trabajo aún no mergeado. Se aplica
   limpio **porque la rama de PR ya estaba cortada de `upstream/main`** — otra razón para
   generarla siempre así y no desde `develop`.
5. Gate completo sobre el resultado (`xcodegen` · build · `swift test`) ANTES de empujar.
6. `git push --force-with-lease` en `develop` **y en `main`**: `main` también diverge si alguna
   vez le pusiste un commit propio.

Y al probar la app: `pkill -x HerdrM` **no mata nada**, el proceso se llama `herdrm` en
minúsculas. `open` sobre una instancia ya viva solo la trae al frente, así que se prueba un
binario de ayer creyendo que es el de ahora. Contrastar siempre `stat -f %Sm` del binario
contra `ps -o lstart=` del proceso; costó una ronda entera de "no están los controles".

## #5 — foco del terminal al cambiar de agente (2026-08-20)

- **El PM aplicó el fix a mano** (+9 líneas, un archivo): lanzar un builder externo con brief
  hubiera costado más que el diff. Regla: por debajo de ~20 líneas en un solo archivo, lo hace el PM.
- **El auditor cazó un fallo real del PM: el fix estaba sin commitear.** Build, tests y checks se
  midieron sobre el working tree, que no es lo que se mergea. Commitear ANTES de auditar; el auditor
  audita `git diff develop..rama`, no el disco.
- **Probar con un binario viejo.** El usuario reportó "no funciona" con la app abierta desde antes de
  compilar. Tras cada build, matar y relanzar `build/Build/Products/Debug/herdrm.app` — comparar
  `ls -l .../Contents/MacOS/herdrm` con la hora del build.
- **`gh pr list` puede venir cacheado:** mostraba el #27 abierto cuando ya estaba mergeado en
  `upstream/main`. Fiarse de `git log upstream/main` tras `git fetch upstream`.
- **manual-gate de notificación:** se puede provocar desde la propia sesión — `sleep` en background y
  luego AskUserQuestion deja el pane Blocked mientras el usuario manda la app al fondo.

## 2026-08-20 — research: orden de ⌘K
- El orden por urgencia ya existe como `AgentStatus.sortBucket` (HerdrKit/Models.swift) y el sidebar
  lo consume en `AppModel.visibleAgents`; SearchView es la única superficie que lista agentes sin él.
- "Agente necesita algo" ≡ `status == .blocked`; las notificaciones (notifyTransitions →
  NotificationManager) son consumidores del mismo campo, no una fuente aparte — para features nuevas
  se comparte `AgentStatus`, nunca el pipeline de notificaciones.
- El icono de la fila de ⌘K es BrandIcon (marca del agente), fácil de confundir con un glifo de
  estado; verificar con grep antes de asumir que el estado ya se muestra.

## #6 — orden de ⌘K por estado (2026-08-20)

- **El glifo compartido va en `Theme.swift`, no en archivo nuevo:** ya vivía ahí `statusColor(_:)`
  sobre el mismo enum. Un `struct` de 15 líneas no justifica un archivo.
- **`ContentView.swift` conserva su propio `statusGlyph` privado** (spinner 13×13, variante compacta).
  No se unificó: la spec prohibía tocar ese archivo. Candidato a issue aparte, no a arreglo al vuelo.
- **Notas de research sin commitear en el working tree** al arrancar el flow: commitearlas APARTE
  antes del cambio funcional, para que el commit de la feature quede limpio y el PR se pueda cortar
  de un solo commit (`git diff <sha>~1..<sha>`), no de `develop~1..develop`.
- **`git apply -3`** para la rama de PR: `upstream/main` no tenía el bloque `### Fixed` del #5, así que
  el contexto del CHANGELOG no casaba. El 3-way lo resuelve; el `git apply` a secas habría fallado.
- **manual-gate verificado en vivo poniendo el propio pane en blocked** (AskUserQuestion) y mirándolo
  desde otro agente: el flow puede generar su propio caso de prueba.

## 2026-08-20 — research: scroll de ⌘K
- SearchView usa ScrollView "a pelo": cualquier navegación por teclado nueva en sheets necesita
  ScrollViewReader + scrollTo(anchor: nil) o el resaltado se sale del viewport (~8 filas de 36px
  en maxHeight 320).
- anchor: nil es la pieza clave: no-op sobre filas visibles, así onHover (que también muta
  highlighted) no salta el scroll.
- Con PRs upstream en vuelo, "mismo archivo" genera un edge de PR (rebase sobre el PR anterior)
  aunque el código ya esté en develop y el issue sea ejecutable ya.

## 2026-08-20 — issue #7 (scroll de ⌘K → PR upstream #30)
- **Un diff de 6 líneas no merece builder externo.** Lancé el edit yo (gate humano aprobó la
  desviación): brief + watcher + hasta 2 reintentos cuestan más que el diff. El gate de compilación
  y la auditoría adversaria se mantuvieron íntegros — lo que se salta es el motor, no los gates.
- **`gh issue view` devuelve vacío en este entorno** (exit 0, cero bytes, con y sin sandbox).
  `gh issue list` y `gh issue view --json … -q` sí funcionan. Usar siempre la forma `--json`.
- **La etiqueta `in-progress` no existía en el fork.** El Paso 2 del flow asume que sí; hubo que
  `gh label create`. Ya existe para los siguientes issues.
- **`git apply` a secas sí sirve para `Sources/`, falla en `CHANGELOG.md`.** El código del fix no
  dependía del #6, pero el contexto del changelog en `upstream/main` no casaba (le faltan los
  bloques del #5 y #6). Patrón: aplicar `-- Sources` con `git apply` y **reescribir el bullet del
  changelog a mano** sobre la rama de PR. Más predecible que `-3` cuando el `### Fixed` ni existe.
- **Alcance añadido a mitad de flow (el usuario pidió el reset al reabrir):** el orden que funcionó
  fue implementar → build → que el usuario lo pruebe → **actualizar la spec Y el issue** (comentario
  con su propia casilla `manual-gate`) → segunda pasada del auditor sobre el DELTA, no sobre todo.
  Baratísima (19k tokens) y cazó que el bullet del CHANGELOG no mencionaba el comportamiento nuevo.
- **El auditor pidiendo por escrito lo que no puede verificar** (`scrollTo` dentro de `onAppear` es
  históricamente frágil en SwiftUI sin layout; descansa en el manual-gate) es la forma correcta de
  aprobar: dice de qué no responde en vez de darlo por bueno.
- **La rama de PR se rehace, no se enmienda,** cuando cambia el alcance: `git branch -D pr/<slug>`
  y recortar de nuevo desde `upstream/main` con el delta completo (`git diff <base-de-la-rama>..develop`,
  no `develop~1..develop`, que solo trae el último merge).

## 2026-08-21 — research: split de terminal (serie #8→#9→#10)
- `herdr agent attach` es SOLO para panes con agente: sobre un pane shell devuelve
  `agent_not_found` (probado contra el socket vivo, herdr 0.8.0). Un "terminal convencional"
  embebido tiene que ser proceso local/ssh de la app, no un pane de herdr.
- SwiftTerm sella `becomeFirstResponder`/`resignFirstResponder` como `public override`
  (MacTerminalView.swift:1313) — misma trampa que `keyDown`. Rastreo de foco: KVO sobre
  `NSWindow.firstResponder` (documentado KVO-observable), nunca subclasear.
- `HSplitView` de SwiftUI no expone el ratio programáticamente: cualquier split que necesite
  resize por teclado se construye a mano (GeometryReader + ratio en estado).
- Serie apilada upstream: fases = issues = PRs encadenados, cada cuerpo declara "builds on"
  y el orden de merge; `blocked` en las fases 2+ hasta que la anterior cierre.

## 2026-08-21 — issue #8 (split de terminal): el auditor murió mudo 4 veces

**Fallo del enjambre, no del código.** Cuatro subagentes de auditoría entraron en
`idle_notification` / `idleReason: available` sin emitir veredicto: 3× `auditor` (Opus) y
1× `general-purpose` (Opus). El builder de la misma sesión SÍ entregó su reporte, así que
no es el mecanismo de subagentes en general.
- Hipótesis probada y **REFUTADA**: que se bloqueaban en un prompt de permiso al leer el
  diff en `/private/tmp/.../scratchpad` (fuera de los working dirs de la sesión). Moví el
  diff a `.swarm/audit-split.diff`, dentro del repo, y el cuarto murió igual.
- Descartado también que sea la definición de `.claude/agents/auditor.md`: falló igual con
  `general-purpose`, que no la usa.
- El `SendMessage` de recuperación pidiendo solo el veredicto en prosa tampoco lo revive.
  Es el mismo síntoma del 2026-08-19 (issue #1), ahora con muestra de 4 y una hipótesis
  menos. **Pendiente**: probar el auditor en una sesión nueva de Claude Code.
- Consecuencia operativa: reposo sin veredicto NO es aprobación. Se reporta al humano que
  el gate no corrió y se para; no se mergea diciendo "auditado".

**El agente `swift-builder` de opencode nunca ha existido.** `swarm-manifest.json` declara
`"opencode": ["swift-builder", "docs-writer"]`, pero no hay `.opencode/agent/` ni en el repo
ni en `~/.config/opencode`. La primera línea de `.swarm/apply-terminal-thin-strokes.log` lo
dice: `! agent "swift-builder" not found. Falling back to default agent`, y corrió con
`> build · kimi-k2.7-code`. La entrada del 2026-08-19 que dice "Verificado: swift-builder →
kimi-k2.7-code" es FALSA por su propio criterio: la línea de estado dice `build`, no
`swift-builder`. El motor (`opencode-go` autenticado, kimi resuelve) nunca fue el problema;
faltaba el archivo del agente. Redactado en scratchpad, pendiente de aterrizar en
`.opencode/agent/swift-builder.md`.

**Defectos que el diff traía y que solo aparecen leyéndolo** (4, todos del builder Claude):
- `DragGesture` con `value.location` mide contra el origen de la vista del gesto, no del
  contenedor. Divisor de 7pt → el ratio arrancaba en ~0 y brincaba al clamp. Se arregla con
  `value.translation` (delta, independiente del espacio) sobre un ancla capturada al inicio.
  La API `.coordinateSpace(.named(_:))` NO sirve aquí: es macOS 15+ y el target es 14.0.
- Meter `colorScheme` en el `.id` de un terminal LOCAL lo mata al cambiar de tema. En el
  attach da igual (herdr guarda el pane server-side); un shell local no tiene a dónde volver.
- `NSCursor.push()` en `.onHover` sin `pop()` garantizado: si la vista desaparece con el
  puntero encima, `onHover(false)` nunca llega y el cursor de resize se queda pegado en toda
  la app. `.set()` no lleva pila y no puede desbalancearse.
- `static stored properties` no existen en tipos genéricos de Swift: un `static let` dentro
  de `SplitContainer<First, Second>` no compila (lo cazó el builder solo).

**⌘W no se puede interceptar desde `CommandGroup(replacing: .saveItem)`.** El "Close" del
menú File lo genera `WindowGroup` y no hay `CommandGroupPlacement` para él, así que el
resultado probable es ⌘W duplicado. No es verificable estáticamente: queda como el primer
manual-gate a probar. Si el split no se cierra, es el Close automático comiéndose el atajo.

**`.swarm/lessons-learned.md` se sobrescribió con Write en vez de anexar** y perdió el
encabezado y todas las entradas previas (283 líneas). Se recuperó con `git checkout --` +
re-anexar. Este archivo se ANEXA, nunca se reescribe.

## 2026-08-21 — issue #8, addendum: los auditores NO estaban mudos

**Corrección de la entrada anterior y de la del 2026-08-19.** Los subagentes de auditoría
terminaban bien: `stop_reason: end_turn` con el veredicto completo escrito en su transcript
(`~/.claude/projects/<proyecto>/<sesion>/subagents/agent-*.jsonl`). Lo que falla es la
ENTREGA: el texto final de un subagente en background **no se propaga al padre**; al padre
solo le llega el `idle_notification`. El builder sí entregó porque llamó `SendMessage`
explícitamente ("Reported to the team lead"); los auditores dejaron el veredicto como texto
de turno y ahí se quedó. Incluso la respuesta del auditor a un `SendMessage` de rescate
quedó sin entregar.
- **`TaskOutput` NO es la vía**: está deprecado y para agentes locales advierte de no leer
  su fichero de salida (symlink al transcript completo → desborda el contexto). Corrección
  de lo que escribí primero en esta misma entrada.
- **El arreglo que funciona, probado en la ronda final**: el prompt del auditor le ORDENA
  entregar el veredicto llamando `SendMessage` con `to: "main"`, avisándole de que un texto
  final de turno no se propaga al padre. Con esa línea, el veredicto llegó a la primera.
- Red de rescate si un subagente ya murió sin entregar: leer el último mensaje `assistant`
  de `~/.claude/projects/<proyecto>/<sesion>/subagents/agent-*.jsonl` filtrando por bloques
  de texto largos. Nunca leer el .jsonl completo.
- Regla de lectura: `idle` != mudo. Ir a buscar el resultado antes de concluir que no hay.
- Veredictos reales de esta corrida: DENEGADO / APROBADO / DENEGADO. Se estuvo a punto de
  mergear un bloqueante creyendo que "el gate no corrió".

**Defecto bloqueante que solo vio el auditor, y que la prueba manual NO delata.** Poner
`first()` en dos ramas de un condicional (`if let axis` / `else`) hace que SwiftUI destruya
y reconstruya el subárbol al conmutar: la identidad no sobrevive un cambio de rama de
`_ConditionalContent`, y `.id()` no ayuda porque está anidado dentro de la posición
estructural que desaparece. Resultado: dividir mataba el `AttachTerminalView` →
`terminate()` → moría el `ssh -tt` del agente y se relanzaba con `--takeover`.
- **Invisible a ojo**: herdr repinta el pane al reattachear, así que se ve igual. Manuel
  probó el split y dijo "quedó muy bien" con el defecto presente.
- **Verificación objetiva que sí lo caza**: `pgrep -f "agent attach"` antes y después de
  dividir. El PID viejo quedaba `<defunct>` (zombie, ni cosechado) y aparecía uno nuevo.
  Para cualquier cambio que toque la jerarquía del terminal, comparar PIDs es el gate, no
  mirar la pantalla.
- Arreglo: una sola posición estructural, eje vía `AnyLayout(HStackLayout)` /
  `AnyLayout(VStackLayout)`, que conmuta sin tocar la identidad de los hijos.

**`CommandGroup(replacing: .saveItem)` SÍ es el sitio correcto para el ⌘W.** Yo sospeché
que quedaría duplicado con el "Close" que genera `WindowGroup`; dos auditores lo refutaron:
`Close` vive en ese grupo junto a Save/Revert. Sospecha mía, infundada.
