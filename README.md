# Athenea

> Asistente educativo basado en RAG (*Retrieval-Augmented Generation*): responde a las dudas del alumnado y las familias apoyándose en los documentos oficiales del centro y citando siempre sus fuentes.

[![CI](https://github.com/IvanGonzalezHernandez/athenea/actions/workflows/ci.yml/badge.svg)](https://github.com/IvanGonzalezHernandez/athenea/actions/workflows/ci.yml)
[![Licencia: MIT](https://img.shields.io/badge/licencia-MIT-blue.svg)](LICENSE)

## Índice

- [Visión](#visión)
- [Funcionalidades previstas](#funcionalidades-previstas)
- [Arquitectura](#arquitectura)
- [Pipeline de ingesta](#pipeline-de-ingesta)
- [Gestor documental](#gestor-documental)
- [Puesta en marcha](#puesta-en-marcha)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Documentación](#documentación)
- [Cómo contribuir](#cómo-contribuir)
- [Equipo](#equipo)
- [Licencia](#licencia)

## Visión

El personal del centro (secretaría, jefatura de estudios, orientación, departamentos…) sube los PDFs oficiales: calendarios, normativa, guías, tasas, horarios. Athenea los procesa en segundo plano y los pone a disposición de un chatbot. Cada respuesta se genera a partir de fragmentos de esos documentos y cita de dónde sale la información, para que se pueda comprobar.

## Funcionalidades previstas

- [ ] Subida de PDFs con metadatos (tipo, fecha, curso, departamento, autor) y validación de entrada
- [ ] Detección de duplicados por fichero (SHA-256) y por contenido
- [ ] Procesado asíncrono: extracción de texto, troceado en *chunks* y generación de *embeddings*
- [ ] Indexación en una base vectorial con filtros de vigencia y confidencialidad
- [ ] Versionado de documentos ("sustituye a") para que el chatbot solo use la versión vigente
- [ ] Reintentos automáticos, reprocesado manual y supervisor de atascos
- [ ] Gestor documental con panel de monitorización y roles
- [ ] Chatbot con cita de las fuentes
- [ ] Evaluación de la calidad de las respuestas

## Arquitectura

```text
Documentos ──► Ingesta y troceado ──► Embeddings ──► Base vectorial
                                                           │
Pregunta ──► Embedding de la pregunta ──► Recuperación ◄───┘
                                               │
                                               ▼
                                    LLM + contexto ──► Respuesta con fuentes
```

Stack propuesto, pendiente de aceptar en el [ADR-0001](docs/adr/0001-elegir-stack-tecnologico.md):

| Pieza | Tecnología |
| ----- | ---------- |
| API | FastAPI |
| Cola de tareas | Celery + Redis (worker y `beat` para el supervisor) |
| Metadatos y ficheros | MongoDB (colección `documentos`) + GridFS (PDF original) |
| Base vectorial | Qdrant |
| Embeddings | `multilingual-e5-base` |
| Front | Streamlit |

Todo se levanta con Docker Compose en siete servicios: `mongo`, `qdrant`, `redis`, `api` (puerto 8000), `worker`, `beat` y `front` (puerto 8501). La propuesta está en [`ide de compose.txt`](docs/adr/Flujo%20y%20monitor/monitor/ide%20de%20compose.txt).

## Pipeline de ingesta

![Pipeline de ingesta documental para RAG](docs/adr/Flujo%20y%20monitor/pipeline_rag.svg)

En resumen:

1. La API recibe el PDF en `POST /documentos` y lo valida: firma `%PDF`, como máximo 20 MB y 300 páginas, sin contraseña ni corrupción y sin duplicar un SHA-256 existente.
2. Guarda el original en GridFS, crea la ficha con estado `PDF OK` y responde `202` con el `doc_id` sin esperar al procesado.
3. Un worker de Celery extrae el texto, lo trocea (800 tokens con 100 de solapamiento), genera los *embeddings* y hace un *upsert* idempotente en Qdrant con `id = uuid5(doc_id, nº de chunk)`.
4. Los fallos temporales se reintentan 3 veces (1, 5 y 15 min). Si se agotan, el documento pasa a `ERROR` con la etapa y el mensaje, y se puede reprocesar a mano después de arreglar la causa.
5. Un supervisor se ejecuta cada 5 minutos. Pasa a `ERROR` lo que lleve más de 15 minutos procesando y elimina los vectores huérfanos.

Estados de un documento: `PDF OK` → `PROCESANDO VECTORES` → `SUBIDA CON ÉXITO`, o bien `DUPLICADO`, `SIN TEXTO` (necesita OCR) o `ERROR`. Los documentos confidenciales o no vigentes nunca se usan en las respuestas.

## Gestor documental

Las maquetas del front están en [`docs/adr/Flujo y monitor/monitor`](docs/adr/Flujo%20y%20monitor/monitor):

| Pantalla | Para qué sirve |
| -------- | -------------- |
| [Panel](docs/adr/Flujo%20y%20monitor/monitor/1-dashboard.png) | Documentos por estado, errores por etapa, actividad diaria y documentos que requieren revisión |
| [Documentos](docs/adr/Flujo%20y%20monitor/monitor/2-documentos.png) | Listado con filtros, ficha de cada documento e historial de estados |
| [Subir PDF](docs/adr/Flujo%20y%20monitor/monitor/3-subir%20PDF.png) | Formulario de subida con metadatos, confidencialidad y "sustituye a" |
| [Errores](docs/adr/Flujo%20y%20monitor/monitor/4-control%20errores.png) | Errores por etapa, registro de intentos y reprocesado individual o en lote |
| [Configuración](docs/adr/Flujo%20y%20monitor/monitor/5-configuracion.png) | Límites, indexación, reintentos, supervisor, usuarios y permisos |

![Panel de ingesta](docs/adr/Flujo%20y%20monitor/monitor/1-dashboard.png)

Roles: **Administración** (subir, eliminar, reprocesar y configurar), **Editor** (subir y reprocesar) y **Lectura** (solo chatbot, sin confidenciales).

## Puesta en marcha

> Todavía no hay código. Esta sección se completará cuando se acepte el ADR-0001 y exista el `docker-compose.yml`.

```bash
git clone https://github.com/IvanGonzalezHernandez/athenea.git
cd athenea
cp .env.example .env   # y rellenar las variables
```

## Estructura del repositorio

```text
.
├── .claude/
│   └── skills/athenea/   # Skill de Claude Code con el contexto del proyecto
├── .github/              # Plantillas de issues y PR, CI, CODEOWNERS
├── docs/
│   └── adr/              # Registro de decisiones de arquitectura
│       └── Flujo y monitor/  # Diagrama del pipeline, maquetas y compose propuestos
├── .env.example          # Variables de entorno necesarias (sin secretos)
├── CONTRIBUTING.md       # Flujo de trabajo del equipo
└── README.md
```

Estructura prevista del código:

```text
docker-compose.yml
api/      # FastAPI (main.py) y Celery (tareas.py)
front/    # Streamlit (app.py)
```

## Documentación

| Documento | Contenido |
| --------- | --------- |
| [ADR-0000](docs/adr/0000-plantilla.md) | Plantilla para nuevas decisiones |
| [ADR-0001](docs/adr/0001-elegir-stack-tecnologico.md) | Elección del stack (propuesto) |
| [Pipeline de ingesta](docs/adr/Flujo%20y%20monitor/pipeline_rag.svg) | Flujo, ciclo de vida, control de errores y garantías del diseño |
| [Maquetas del monitor](docs/adr/Flujo%20y%20monitor/monitor) | Las cinco pantallas del gestor documental y la estructura de contenedores |
| [Lenguajes](docs/adr/Flujo%20y%20monitor/monitor/lenguaje.txt) | FastAPI para el back y Streamlit para el front |

Si usas [Claude Code](https://claude.com/claude-code), la skill de [`.claude/skills/athenea`](.claude/skills/athenea/SKILL.md) le da todo este contexto automáticamente al trabajar en el repo.

## Cómo contribuir

Lee [CONTRIBUTING.md](CONTRIBUTING.md) antes de abrir tu primera rama. En resumen: cada miembro trabaja en su rama `features/rama_<nombre>` (o en una `feature/*` para tareas concretas) que sale de `develop`, usa commits convencionales y abre una PR hacia `develop`. `main` solo recibe versiones desde `develop`.

## Equipo

| Nombre | GitHub | Rol |
| ------ | ------ | --- |
| Iván González | [@IvanGonzalezHernandez](https://github.com/IvanGonzalezHernandez) | |
| Asier | [@Asiernc12](https://github.com/Asiernc12) | |
| Miguel | [@Migas67](https://github.com/Migas67) | |
| Guillermo | [@gcruzg2601edu](https://github.com/gcruzg2601edu) | |

## Licencia

Distribuido bajo licencia MIT. Consulta [LICENSE](LICENSE).
