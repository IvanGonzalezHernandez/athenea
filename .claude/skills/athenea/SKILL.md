---
name: athenea
description: Contexto y reglas del proyecto Athenea, un asistente educativo RAG para un centro educativo (gestor documental de PDFs + chatbot que cita fuentes). Úsala siempre que trabajes en el repo de Athenea o se hable de él, aunque no se nombre la skill, en particular al programar la API FastAPI, los workers Celery, el front Streamlit, el docker-compose, la ingesta de PDFs, los estados de los documentos, MongoDB/GridFS, Qdrant, los embeddings, el panel de monitorización, o al redactar ADR, issues, ramas, commits y PR del equipo.
---

# Athenea

Athenea es un RAG educativo que hacen cuatro personas. El personal del centro (secretaría, jefatura, orientación…) sube los PDFs oficiales con sus metadatos, el sistema los indexa en segundo plano, y el alumnado y las familias preguntan a un chatbot que responde con ese material y cita la fuente.

Esta skill resume lo que dice `docs/`. Si algo de aquí contradice lo que hay en el repo, manda el repo: vuelve a leer el documento afectado y avisa de la diferencia.

## Estado de las decisiones

- `docs/adr/0001-elegir-stack-tecnologico.md` sigue como **Propuesto**. Lo que hay en `docs/adr/Flujo y monitor/` es la propuesta de diseño del equipo, todavía sin aceptar formalmente.
- Antes de dar algo por decidido (stack, modelo, límites), compruébalo en el ADR. Si propones una decisión nueva, redáctala como ADR copiando `docs/adr/0000-plantilla.md`, con el siguiente número libre.

## Stack propuesto

| Pieza | Tecnología | Papel |
| ----- | ---------- | ----- |
| API | FastAPI (`api/main.py`) | Recibe subidas y borrados, valida y encola |
| Tareas | Celery + Redis (`api/tareas.py`) | Worker de procesado y `beat` para el supervisor |
| Metadatos | MongoDB, colección `documentos` | Ficha y estado de cada documento |
| Ficheros | GridFS | PDF original, inmutable |
| Vectores | Qdrant | Chunks vectorizados con filtros `vigente` y `confidencial` |
| Embeddings | `multilingual-e5-base` | Versionado en el campo `modelo_embed` |
| Front | Streamlit (`front/app.py`) | Gestor documental y monitor |

Estructura prevista del código, según `contendores estructura.png`:

```text
docker-compose.yml
api/
  Dockerfile
  requirements.txt
  main.py      # FastAPI
  tareas.py    # Celery (worker y beat)
front/
  Dockerfile
  requirements.txt
  app.py       # Streamlit
```

Servicios de `ide de compose.txt`: `mongo` (27017), `qdrant` (6333), `redis` (6379), `api` (8000), `worker`, `beat` y `front` (8501). `api` y `worker` reciben `MONGO_URL`, `QDRANT_URL` y `REDIS_URL`, y el front recibe `API_URL`. El `.env.example` actual todavía usa nombres genéricos (`LLM_API_KEY`, `VECTOR_DB_URL`…): cuando añadas variables, alinéalo con las del compose.

## El pipeline de ingesta en una frase

La API valida y guarda el PDF, crea la ficha en estado `PDF OK` y responde `202` con el `doc_id`. Un worker lo trocea, vectoriza y lo sube a Qdrant. Un supervisor desbloquea los atascos y limpia los restos.

Lee `references/pipeline.md` antes de tocar cualquier parte de la ingesta, los estados, los reintentos, el borrado o el supervisor. Ahí están los códigos HTTP, los campos de la colección, el ciclo de vida completo y las garantías que el diseño debe mantener.

Reglas que se rompen con facilidad, y por qué importan:

- **La API no procesa el PDF.** Si extrae texto o genera embeddings dentro de la petición, el usuario se queda esperando y un PDF grande tumba la API.
- **El id de cada vector es determinista:** `uuid5(doc_id, nº de chunk)`. Gracias a eso, reprocesar sobrescribe en vez de duplicar. Un id aleatorio rompe esa garantía.
- **Los confidenciales nunca llegan al chatbot.** Toda búsqueda en Qdrant para responder debe filtrar `confidencial = false` y `vigente = true`.
- **El PDF de GridFS nunca se modifica.** Es la fuente de verdad desde la que se reprocesa.
- **Cada error guarda `etapa_error` y `error_msg`.** El monitor agrupa los errores por etapa (Extracción, Embeddings, Qdrant). Sin esos campos, el error no se puede diagnosticar.

## Monitor (front Streamlit)

Hay maquetas de cinco pantallas: Panel, Documentos, Subir PDF, Errores y Configuración. También hay tres roles: Administración, Editor y Lectura. Para implementar o modificar el front, lee `references/monitor.md`, que describe qué muestra cada pantalla y qué puede hacer cada rol. Abre las capturas de `docs/adr/Flujo y monitor/monitor/` cuando necesites el detalle visual.

## Forma de trabajar del equipo

La guía completa está en `CONTRIBUTING.md`. Lo esencial:

- La rama por defecto es `develop`. `main` solo recibe merges desde `develop` (merge commit + etiqueta `vX.Y.Z`) y está protegida: PR, 1 aprobación y CI en verde.
- `develop` admite push directo, pero lo normal es trabajar en una rama y abrir PR. Cada miembro tiene la suya (`features/rama_asier`, `features/rama_miguel`, `features/rama_ivan`, `features/rama_guillermo`). Para tareas concretas, `CONTRIBUTING.md` usa `feature/<issue>-<descripcion>`.
- Los commits y los títulos de PR siguen Conventional Commits en español e imperativo, por ejemplo `feat(ingesta): añade validación de firma PDF`. La CI comprueba el título de la PR.
- La CI pasa `markdownlint` sobre todos los `.md`. Deja una línea en blanco alrededor de títulos, listas, tablas y bloques de código, e indica siempre el lenguaje del bloque.
- Nunca subas `.env` ni claves. Documenta las variables nuevas en `.env.example`.

## Idioma

Todo el proyecto está en español: documentación, mensajes de la interfaz, estados (`SUBIDA CON ÉXITO`, `SIN TEXTO`…), nombres de colecciones y campos (`documentos`, `vigente`, `curso`…) y commits. Mantén ese criterio en el código nuevo para que el modelo de datos coincida con el diseño.
