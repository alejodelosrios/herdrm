---
description: Sincroniza el fork (main + develop) con missuo/herdrm — detecta qué ya se mergeó upstream, corre el gate local, y publica con force-with-lease
argument-hint: (sin argumentos)
---

# /herdrm-sync

Trae `upstream/main` al fork sin perder el enjambre (`.claude/`, `.swarm/`) ni trabajo de
`develop` que todavía no llegó a missuo. Ejecuta esto, en orden, sin saltarte el gate.

**No sirve `git rebase upstream/main`.** missuo mergea con **squash**: tus commits ya
aceptados existen upstream con otro SHA y otro contenido; rebasar los reaplica y revienta
contra su propia versión ya mergeada. El único camino es reconstruir `develop` desde
`upstream/main` y solo traer encima lo que de verdad falta.

## 0. Árbol limpio (precondición)

```sh
git status --porcelain    # debe salir vacío
```

El paso 3 hace `git switch`, y git lo aborta si hay cambios sin commitear en archivos que
difieren entre ramas — pasa siempre que vienes de editar `.claude/` o `.swarm/`. Commitea o
haz stash antes de empezar; no arranques la sync con el árbol sucio.

## 1. Fetch y diagnóstico

```sh
git fetch upstream --tags && git fetch origin
git log --oneline main..upstream/main    # lo nuevo que trae upstream
git log --oneline upstream/main..main    # DEBE salir vacío
```

Si `upstream/main..main` no sale vacío, `main` tiene un commit propio que nunca se mandó
a upstream — **detente y pregunta al dueño** antes de tocar nada; no es el caso normal.

## 2. Censo de qué ya está mergeado

```sh
gh pr list --repo missuo/herdrm --author alejodelosrios --state all \
  --json number,title,state,headRefName,mergedAt
```

Cruza contra las ramas locales `pr/*` (`git branch --list 'pr/*'`). Toda rama con PR
`MERGED` no necesita nada más — bórrala si ya no sirve de referencia.

Después, censa `develop` en busca de commits que **no** estén en ninguna rama `pr/*` ni en
`upstream/main` — trabajo hecho después de la última sync que aún no se propuso arriba:

```sh
git log --oneline main..develop   # candidatos; filtra los "docs(swarm): ..." (no aplican)
```

Para cada candidato, **no asumas que falta** — puede que missuo ya lo haya implementado
por su cuenta (pasó con el fix de scroll del ⌘K: llegó con otro autor, mismo código). Verifica
línea por línea contra `upstream/main` antes de decidir que hay que cherry-pickearlo:

```sh
git show upstream/main:<archivo> | grep -n -A3 -B3 '<ancla del cambio>'
```

## 2b. Cerrar los issues del fork que upstream ya absorbió

Un PR mergeado arriba deja su issue del fork abierto: nadie lo cierra desde upstream, porque
missuo no ve tus issues. Este es el momento de barrerlos — si no, `in-progress` se acumula y
`/herdrm-flow` arranca sobre issues que ya están hechos.

```sh
gh issue list --repo alejodelosrios/herdrm --state open \
  --json number,title,labels --jq '.[] | "\(.number)\t\(.title)"'
```

Para cada issue abierto, decide con evidencia, **no por el título**:

1. ¿Hay un PR tuyo MERGED que lo implementaba? (censo del paso 2). Si sí → cerrar.
2. Si no hubo PR, ¿lo implementó missuo por su cuenta? Verifica el código en
   `upstream/main`, no el mensaje del commit:
   ```sh
   git show upstream/main:<archivo> | grep -n '<ancla del criterio>'
   ```
   Si el comportamiento está arriba, el issue está resuelto aunque el código no sea tuyo → cerrar
   citando el commit de missuo.
3. Si solo está **parcialmente** arriba, NO lo cierres: comenta qué parte quedó fuera y deja
   el issue abierto con esa parte como alcance restante.

Al cerrar, deja la trazabilidad y quita las etiquetas de trabajo en curso:

```sh
gh issue close <n> --repo alejodelosrios/herdrm \
  --comment "Mergeado en upstream: <url del PR> → \`<sha>\`. Cerrado en la sync del $(date +%Y-%m-%d)."
gh issue edit <n> --repo alejodelosrios/herdrm --remove-label in-progress
```

`gh issue edit`/`close` pueden devolver 0 sin mutar → **relee y confirma** con
`gh issue view <n> --json state,labels`. Es la misma trampa que el paso 2 de `/herdrm-flow`.

Ojo con las series por fases: cerrar la fase 1 no cierra las siguientes, y si missuo cambió la
arquitectura al mergear (pasó con #8: añadió terminales independientes encima del split), revisa
que las fases pendientes sigan teniendo sentido antes de dejarlas abiertas tal cual.

## 3. Reconstruir develop

```sh
BK=backup/develop-pre-sync-$(date +%Y%m%d-%H%M)   # con hora: dos syncs el mismo dia no colisionan
git branch "$BK" develop
git switch -c develop-sync upstream/main
git checkout "$BK" -- .claude .swarm .opencode
git add .claude .swarm .opencode
git commit -m "chore(swarm): restore agent harness after upstream sync"
```

**Los tres directorios, siempre: `.claude`, `.swarm` y `.opencode`.** Olvidar `.opencode` borra
los agentes de opencode sin decir nada, porque `upstream/main` no tiene ese directorio y `develop`
se reconstruye desde ahí. Comprobación previa de que no falta ninguno:

```sh
git ls-tree -r develop --name-only | grep -oE '^\.(claude|swarm|opencode)/' | sort -u
```

El historial del enjambre **no se cherry-pickea** (commits sucesivos sobre
`.swarm/lessons-learned.md` chocan entre sí contra la base nueva) — se colapsa en este único
commit.

Si el paso 2 encontró trabajo genuino pendiente:

```sh
git cherry-pick -x <sha>   # aplica limpio si la rama pr/* de origen se cortó de upstream/main
```

Si hay conflicto en `CHANGELOG.md`, no fuerces el hunk viejo: agrega la línea bajo
`## [Unreleased]` de la versión de upstream.

## 4. Gate local — obligatorio antes de mover ninguna rama

```sh
xcodegen generate
xcodebuild -project HerdrM.xcodeproj -scheme HerdrM -configuration Debug \
  -derivedDataPath build build -skipPackagePluginValidation \
  CODE_SIGN_IDENTITY="Apple Development" DEVELOPMENT_TEAM=JXGN27PTN9
```

```sh
test -S ~/.config/herdr/herdr.sock || echo "falta un herdr local corriendo — arráncalo en otro pane"
(cd Packages/HerdrKit && swift test)   # subshell: un test rojo no te deja dentro del subdirectorio
```

Rojo en cualquiera de los dos → no sigas. Corrige o pregunta.

## 5. Mover las ramas y publicar

```sh
git branch -f develop develop-sync
git branch -f main upstream/main
git checkout develop        # ANTES del -D: git se niega a borrar la rama en la que estas
git branch -D develop-sync

# `git branch -f main upstream/main` re-apunta el tracking de main a UPSTREAM: con eso, un
# `git push` desde main iria al repo de missuo. Devuelvelo a origin y comprueba.
git branch -u origin/main main
git for-each-ref --format='%(refname:short) → %(upstream:short)' refs/heads/main refs/heads/develop

# main tiene que quedar limpio de enjambre (es upstream/main puro)
git ls-tree -r main --name-only | grep -cE '^\.(claude|swarm|opencode)/'   # debe dar 0
# y la unica diferencia entre develop y upstream deben ser los tres directorios
git diff --name-only upstream/main develop | grep -vcE '^\.(claude|swarm|opencode)/'  # debe dar 0

git push --force-with-lease origin develop main
```

## 6. Retirar los backups viejos (deja solo el último)

Cada sync crea un `backup/develop-pre-sync-*`. Se acumulan y dejan de ser una red de
seguridad para volverse ruido. Conserva **solo el más reciente** y retira el resto:

```sh
# ordena por fecha de commit, no por nombre: los backups antiguos no llevan sufijo de hora
git for-each-ref --sort=-committerdate --format='%(refname:short)' 'refs/heads/backup/develop-pre-sync*' \
  | tail -n +2 | while read b; do
      echo "retirando $b ($(git log -1 --format='%h %ad' --date=short "$b"))"
      git branch -D "$b"
    done
git branch --list 'backup/*'   # debe quedar exactamente uno
```

Aquí `-D` es correcto y deliberado, no un descuido: un backup contiene por definición
historia que ya NO está en `develop` (es el `develop` de antes de reconstruirlo), así que
`-d` siempre se va a negar. La seguridad la da el patrón del `for-each-ref` —solo toca refs
bajo `backup/develop-pre-sync*`— y el `tail -n +2`, que preserva el más reciente. Nunca
amplíes ese glob.

## Reglas duras

- `--force-with-lease`, nunca `--force` a secas — si alguien más empujó a `origin/develop`
  o `origin/main` entre medio, que truene y se revise a mano.
- `pkill -x HerdrM` no mata nada: el proceso corre como `herdrm` en minúsculas. Para probar el
  binario recién compilado, primero `osascript -e 'quit app "herdrm"'` y contrasta
  `stat -f %Sm build/Build/Products/Debug/herdrm.app/Contents/MacOS/herdrm` contra
  `ps -o lstart= $(pgrep -x herdrm)` — `open` sobre una instancia ya viva solo la trae al
  frente y quedas probando el binario de ayer.
- No abras PR nuevo desde aquí. Este comando solo sincroniza; para proponer trabajo pendiente
  a missuo usa `/herdrm-flow` cortando la rama de `upstream/main` ya actualizado.
