# CONCERNS — herdrm

Escritor unico: el orquestador. Append-only.

- La receta de build depende de un cert Apple Development local (`DEVELOPMENT_TEAM=JXGN27PTN9`). Si expira, `make build` y el gate se caen juntos.
- `project.yml` fija el Developer ID del maintainer. Cualquier PR que lo toque se rechaza; el override vive fuera del repo.
