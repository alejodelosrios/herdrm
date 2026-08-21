---
description: Crea o etiqueta issues de herdrm con la plantilla del enjambre (única fuente de verdad de labels y plantillas)
argument-hint: <título o número de issue>
---

# /herdrm-issue

Único lugar donde viven **plantillas y labels**. `/herdrm-research` los consume; no los duplica.

Repo del tracker: **`alejodelosrios/herdrm`** (tu fork; Issues habilitado). Jamás abras issues en
`missuo/herdrm` — ahí no tienes triage; para hablar con upstream se usa el PR o un comentario.

## Labels

| Label | Significado |
|---|---|
| `ready-for-agent` | enriquecido al contrato; un builder lo ejecuta sin reinvestigar. **Precondición del fleet.** |
| `needs-research` | idea cruda; pasa por `/herdrm-research` antes de tocar código |
| `blocked` | tiene un edge abierto; no entra a ninguna ola |
| `upstream-candidate` | el fix va en PR a missuo/herdrm |
| `fork-only` | NO va al PR (firma, `project.yml`, el enjambre, `.claude/`) |
| `manual-gate` | un criterio exige abrir la app y mirar → lo cierra el humano |
| `area:ui` `area:kit` `area:ssh` `area:build` | superficie tocada; alimenta el reparto en olas |

Crear los que falten: `gh label create <n> --repo alejodelosrios/herdrm`.

## Plantilla de cuerpo

```markdown
## Qué se construye
El comportamiento end-to-end desde la perspectiva del usuario. NO una lista capa por capa.

## Verificación (<YYYY-MM-DD>)
Lo confirmado contra HEAD, con `file:line` frescos y los grep que cerraron el alcance.
Cada afirmación externa atada a su fuente primaria.

## Fix propuesto
Ejecutable tal cual: archivo:línea y snippet cuando ahorre una decisión.
Los valores que se ven raros pero son deliberados van marcados como tales.

## Impacto en el PR upstream
¿missuo lo aceptaría? ¿toca firma/`project.yml` (→ `fork-only`)? ¿entrada en CHANGELOG [Unreleased]?

## Ejecutabilidad (fleet/flow)
Auto-ejecutable | Gate de verificación manual | Requiere decisión
Grafo: `#A ∥ #B → #C`

## Criterios de aceptación
- [ ] Falsable: un grep con resultado esperado, `swift test` en verde, el build en verde,
      o una observación concreta CON el gesto exacto.

## Blocked by
`#N`, o "Ninguno — puede arrancar ya".

## Excluido
| Qué | Por qué no |
```

## Reglas duras

- **Un criterio que no se puede fallar no es criterio.** "Se ve bien" se rechaza.
- **Un criterio que no se midió se queda SIN marcar.** Marcarlo "porque el código lo hace" es el fallo
  que el gate existe para evitar.
- Los criterios `manual-gate` **no los marca ningún agente jamás**.
- Cubierto parcialmente → márcalo y di qué falta.
- **El censo del cierre recorre body Y comentarios** (`grep -cE '^- \[ \]'`): si el plan vigente se
  publicó en un comentario, las casillas viven ahí y la API de edición del body no las toca.
- Un `gh issue edit` puede salir con código 0 sin haber mutado nada → **relee y confirma** el cambio.
- Transiciones de estado atadas a un **evento observable y a un actor por función**, nunca a "cuando
  termines": *el que hace el trabajo lo marca en curso; el que abre el PR lo marca en revisión*. Quien
  asuma el rol asume **todos** sus deberes de trazabilidad.
