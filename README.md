# Athenea

> Asistente educativo basado en RAG (*Retrieval-Augmented Generation*): responde a las dudas del alumnado y las familias apoyándose en los documentos oficiales del centro y citando siempre sus fuentes.

[![CI](https://github.com/IvanGonzalezHernandez/athenea/actions/workflows/ci.yml/badge.svg)](https://github.com/IvanGonzalezHernandez/athenea/actions/workflows/ci.yml)
[![Licencia: MIT](https://img.shields.io/badge/licencia-MIT-blue.svg)](LICENSE)

## Índice

- [Visión](#visión)
- [Fases del proyecto](#fases-del-proyecto)
- [Arquitectura](#arquitectura)
- [Pipeline de ingesta](#pipeline-de-ingesta)
- [API](#api)
- [Modelo de datos](#modelo-de-datos)
- [Gestor documental](#gestor-documental)
- [Configuración](#configuración)
- [Puesta en marcha](#puesta-en-marcha)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Documentación](#documentación)
- [Convenciones](#convenciones)
- [Cómo contribuir](#cómo-contribuir)
- [Equipo](#equipo)
- [Licencia](#licencia)

## Visión

El personal del centro (secretaría, jefatura de estudios, orientación, departamentos…) sube los PDFs oficiales: calendarios, normativa, guías, tasas, horarios. Athenea los procesa en segundo plano y los pone a disposición de un chatbot. Cada respuesta se genera a partir de fragmentos de esos documentos y cita de dónde sale la información, para que se pueda comprobar.

## Fases del proyecto

| Fase | Alcance | Estado |
| ---- | ------- | ------ |
| **1. Entrada y persistencia** | API FastAPI con JWT y roles, validación del PDF, deduplicación por SHA-256, GridFS y ficha en `PDF OK`, versionado, confidencialidad, consulta, descarga, borrado y estadísticas. Compose con `mongo` y `api` | Diseñada ([skill](.claude/skills/rag-ingesta-fase1/SKILL.md)) |
| **2. Procesado** | Celery + Redis, worker de extracción y troceado, `hash_texto`, embeddings, Qdrant, reintentos, reprocesado y supervisor | Diseñada ([pipeline](docs/adr/Flujo%20y%20monitor/pipeline_rag.svg)) |
| **Gestor documental** | Front Streamlit con panel, documentos, subida, errores y configuración | Maquetado ([monitor](docs/adr/Flujo%20y%20monitor/monitor)) |
| **Chatbot** | Recuperación con filtros, LLM, respuesta con citas y evaluación de calidad | Pendiente de diseñar |

Funcionalidades:

- [ ] Subida de PDFs con metadatos (tipo, fecha, curso, departamento, autor) y validación de entrada
- [ ] Autenticación con JWT y permisos por rol comprobados en la API
- [ ] Detección de duplicados por fichero (SHA-256) y por contenido
- [ ] Versionado de documentos ("sustituye a") para que el chatbot solo use la versión vigente
- [ ] Procesado asíncrono: extracción de texto, troceado en *chunks* y generación de *embeddings*
- [ ] Indexación en una base vectorial con filtros de vigencia y confidencialidad
- [ ] Reintentos automáticos, reprocesado manual y supervisor de atascos
- [ ] Gestor documental con panel de monitorización
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
| Lenguaje | Python 3.11 |
| API | FastAPI + Uvicorn, Pydantic v2 y `pydantic-settings` |
| Autenticación | JWT (`PyJWT`) y contraseñas con `passlib[bcrypt]` |
| Metadatos y ficheros | MongoDB (`pymongo`) + GridFS para el PDF original |
| Lectura de PDF | `pypdf` (páginas, cifrado, corrupción) |
| Cola de tareas | Celery + Redis (worker y `beat` para el supervisor) |
| Base vectorial | Qdrant |
| Embeddings | `multilingual-e5-base` |
| Front | Streamlit |
| Pruebas | `pytest` + `httpx` contra un MongoDB real en Docker |
| LLM del chatbot | Por decidir |

La API sigue una arquitectura MVC por capas: los **routers** (controladores) llaman a los **servicios** (lógica de negocio), y solo los **repositorios** tocan MongoDB y GridFS. Los servicios no conocen HTTP: lanzan errores de dominio y un manejador común los convierte en respuestas.

Todo se levanta con Docker Compose. La fase 1 usa `mongo` y `api`; el sistema completo tiene siete servicios: `mongo` (27017), `qdrant` (6333), `redis` (6379), `api` (8000), `worker`, `beat` y `front` (8501). La propuesta está en [`ide de compose.txt`](docs/adr/Flujo%20y%20monitor/monitor/ide%20de%20compose.txt).

## Pipeline de ingesta

![Pipeline de ingesta documental para RAG](docs/adr/Flujo%20y%20monitor/pipeline_rag.svg)

1. **Subida** (`POST /documentos`). La API valida todo antes de escribir nada: el rol, los metadatos, el tamaño (leído por bloques), la firma `%PDF-`, que se abra sin contraseña ni errores, el número de páginas y que el SHA-256 no exista ya.
2. **Persistencia.** Guarda el original en GridFS y crea la ficha en estado `PDF OK`. No hay transacciones: si falla la ficha, se borra el fichero de GridFS para no dejar huérfanos. Responde `202` con el `doc_id` sin esperar al procesado.
3. **Procesado** (fase 2). Un worker de Celery extrae el texto, lo trocea (800 tokens con 100 de solapamiento), genera los *embeddings* y hace un *upsert* idempotente en Qdrant con `id = uuid5(doc_id, nº de chunk)`. Si no hay texto, el documento queda en `SIN TEXTO`; si el texto ya existe en otro documento, en `DUPLICADO`.
4. **Supervisor** (fase 2). Se ejecuta cada 5 minutos. Pasa a `ERROR` lo que lleve más de 15 minutos procesando y elimina los vectores cuyo documento ya no existe.

Estados de un documento:

```text
PDF OK ──► PROCESANDO VECTORES ──► SUBIDA CON ÉXITO
                │
                ├──► DUPLICADO   (mismo texto que otro documento)
                ├──► SIN TEXTO   (escaneado, requiere OCR)
                └──► ERROR       (reintentos agotados o timeout del supervisor)

ERROR ──► Reprocesar (manual) ──► se borran sus vectores ──► vuelve a la cola
```

Control de errores en el procesado:

| Aspecto | Reintento automático | Reprocesado manual |
| ------- | -------------------- | ------------------ |
| Para qué fallos | Temporales: servicio caído, corte de red, límite de peticiones | Persistentes: alguien tiene que arreglar algo antes |
| Ejemplos | Qdrant reiniciándose, error 429 de la API de embeddings | Bug ya corregido, cambio de modelo |
| Quién actúa | Nadie, se resuelve solo | Una persona, con el botón Reprocesar |
| Límite | 3 reintentos (1, 5 y 15 min); después, `ERROR` con la etapa y el mensaje | Sin límite; `intentos` queda como información |

El borrado (`DELETE /documentos/{doc_id}`) elimina primero el fichero de GridFS, después la ficha y, en la fase 2, sus vectores de Qdrant. Si algo falla a medias, el supervisor limpia los restos.

Garantías que cualquier cambio debe mantener: el PDF original nunca se pierde, no hay duplicados ni en MongoDB ni en Qdrant, ningún documento queda bloqueado, los errores se pueden rastrear por etapa, los confidenciales quedan fuera del RAG y la API no se bloquea.

## API

| Método y ruta | Rol | Respuesta correcta |
| ------------- | --- | ------------------ |
| `POST /auth/login` | Público | `200` con `{access_token, rol}` |
| `POST /documentos` | Administración, Editor | `202` con `{doc_id, estado: "PDF OK"}` |
| `GET /documentos` | Todos | `200`, lista paginada con filtros `estado`, `tipo`, `departamento` y `vigente` |
| `GET /documentos/{doc_id}` | Todos | `200`, ficha completa |
| `GET /documentos/{doc_id}/pdf` | Todos | `200`, PDF en *streaming* |
| `DELETE /documentos/{doc_id}` | Administración | `204` |
| `GET /estadisticas` | Todos | `200`, total por estado |

`POST /documentos` recibe `multipart/form-data` con el fichero `file` y los campos `tipo`, `departamento`, `curso`, `fecha_documento`, `autor`, `confidencial` y `sustituye_a` (opcional). FastAPI publica la documentación interactiva en `/docs`.

Permisos por rol, comprobados en la API y no solo en la interfaz:

| Acción | Administración | Editor | Lectura |
| ------ | -------------- | ------ | ------- |
| Consultar y descargar documentos | Sí | Sí | Sí, sin confidenciales |
| Subir PDF | Sí | Sí | No |
| Reprocesar errores | Sí | Sí | No |
| Eliminar documento | Sí | No | No |
| Configurar | Sí | No | No |
| Ver estadísticas | Sí | Sí | Sí |

Para el rol Lectura, pedir un documento confidencial por su id devuelve `404` y no `403`, para no revelar que existe.

Todos los errores tienen el mismo formato, con mensajes en español que explican qué ha pasado y cómo solucionarlo, sin trazas internas:

```json
{ "codigo": "DUPLICADO", "mensaje": "El documento ya existe", "detalle": { "doc_id": "…", "nombre": "…" } }
```

| HTTP | Código | Cuándo |
| ---- | ------ | ------ |
| 400 | `NO_ES_PDF`, `PDF_PROTEGIDO`, `PDF_CORRUPTO`, `DEMASIADAS_PAGINAS` | Validación del fichero |
| 401 | `NO_AUTENTICADO` | Sin token, o token inválido o caducado |
| 403 | `SIN_PERMISO` | Rol insuficiente |
| 404 | `NO_ENCONTRADO` | `doc_id` inexistente, o confidencial para Lectura |
| 409 | `DUPLICADO` | El SHA-256 ya existe; devuelve el documento original |
| 413 | `DEMASIADO_GRANDE` | Supera el tamaño máximo |
| 422 | `METADATOS_INVALIDOS` | Formulario incorrecto o `sustituye_a` no válido |
| 500 | `ERROR_ALMACENAMIENTO` | Fallo al escribir en GridFS o en la colección |

## Modelo de datos

**Colección `documentos`**, con una ficha por PDF. Los campos de la fase 2 se crean desde el principio con `null` para no tener que migrar fichas.

| Campo | Contenido |
| ----- | --------- |
| `doc_id` | uuid4 generado al subir |
| `gridfs_id` | Enlace al PDF en GridFS |
| `nombre_original`, `tamano_bytes`, `paginas` | Datos del fichero validado |
| `hash_sha256` | Huella del fichero (único) |
| `tipo`, `departamento`, `curso`, `autor`, `fecha_documento` | Metadatos del formulario |
| `confidencial` | Si es verdadero, queda fuera del RAG |
| `version`, `vigente`, `sustituye_a` | Versionado: al sustituir, la versión anterior pasa a no vigente |
| `estado`, `historial` | Estado actual y lista de cambios `{estado, fecha, detalle}` |
| `intentos`, `etapa_error`, `error_msg` | Control de errores |
| `inicio_proceso`, `modelo_embed`, `hash_texto` | Procesado (fase 2) |
| `subido_por`, `fecha_subida`, `fecha_actualizacion` | Auditoría |

Índices, creados al arrancar la API: `doc_id` único, `hash_sha256` único (garantiza la deduplicación incluso con dos subidas simultáneas), `estado`, `(tipo, departamento, vigente)` y `fecha_subida` descendente.

**Colección `usuarios`**: `usuario` (único), `nombre`, `departamento`, `rol` (`administracion`, `editor` o `lectura`), `password_hash` y `activo`.

## Gestor documental

Las maquetas del front están en [`docs/adr/Flujo y monitor/monitor`](docs/adr/Flujo%20y%20monitor/monitor):

| Pantalla | Para qué sirve |
| -------- | -------------- |
| [Panel](docs/adr/Flujo%20y%20monitor/monitor/1-dashboard.png) | Documentos por estado, errores por etapa, actividad diaria y documentos que requieren revisión |
| [Documentos](docs/adr/Flujo%20y%20monitor/monitor/2-documentos.png) | Listado con filtros, ficha de cada documento e historial de estados |
| [Subir PDF](docs/adr/Flujo%20y%20monitor/monitor/3-subir%20PDF.png) | Formulario de subida con metadatos, confidencialidad y "sustituye a" |
| [Errores](docs/adr/Flujo%20y%20monitor/monitor/4-control%20errores.png) | Errores por etapa (Extracción, Embeddings, Qdrant), registro de intentos y reprocesado individual o en lote |
| [Configuración](docs/adr/Flujo%20y%20monitor/monitor/5-configuracion.png) | Límites, indexación, reintentos, supervisor, usuarios y permisos. Solo Administración |

![Panel de ingesta](docs/adr/Flujo%20y%20monitor/monitor/1-dashboard.png)

## Configuración

Variables de entorno de la API (fase 1):

| Variable | Por defecto | Uso |
| -------- | ----------- | --- |
| `MONGO_URL` | `mongodb://mongo:27017` | Conexión a MongoDB (dentro de Docker, nunca `localhost`) |
| `MONGO_DB` | `rag` | Base de datos |
| `MAX_MB` | `20` | Tamaño máximo del PDF |
| `MAX_PAGINAS` | `300` | Páginas máximas |
| `JWT_SECRET` | Obligatoria | Firma de los tokens |
| `JWT_EXP_MIN` | `480` | Caducidad del token, en minutos |

En la fase 2 se añaden `QDRANT_URL` y `REDIS_URL`, y el front recibe `API_URL`.

Parámetros del procesado, editables desde la pantalla de Configuración:

| Parámetro | Valor por defecto |
| --------- | ----------------- |
| Detección de duplicados por SHA-256 y por contenido | Activas |
| Modelo de embeddings | `multilingual-e5-base` (cambiarlo obliga a reindexar todo) |
| Tamaño de chunk y solapamiento | 800 y 100 tokens |
| OCR para escaneados | Desactivado (se marcan como `SIN TEXTO`) |
| Reintentos automáticos | 3, con esperas de 1, 5 y 15 min |
| Tiempo máximo en PROCESANDO | 15 min |
| Frecuencia del supervisor | 5 min |

## Puesta en marcha

> Todavía no hay código. Estos son los pasos previstos para la fase 1.

```bash
git clone https://github.com/IvanGonzalezHernandez/athenea.git
cd athenea
cp .env.example .env                                       # rellenar JWT_SECRET y el resto
docker compose up -d --build                               # levanta mongo y api
docker compose exec api python scripts/crear_admin.py      # crea el primer administrador
docker compose exec api pytest                             # ejecuta las pruebas
```

La API quedará en <http://localhost:8000> y su documentación en <http://localhost:8000/docs>.

## Estructura del repositorio

```text
.
├── .claude/skills/       # Skills de Claude Code con el contexto del proyecto
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
api/
├── main.py               # Crea la app, registra los routers y crea los índices
├── core/                 # Configuración, conexión a Mongo, seguridad JWT y errores
├── models/               # Entidades y enums (Estado, Rol)
├── repositories/         # Único acceso a MongoDB y GridFS
├── services/             # Validación, ingesta y enganche con la fase 2
├── routers/              # Controladores: auth, documentos, estadísticas
├── scripts/              # crear_admin.py
├── tareas.py             # Celery: worker y beat (fase 2)
└── tests/
front/
└── app.py                # Streamlit
```

## Documentación

| Documento | Contenido |
| --------- | --------- |
| [ADR-0000](docs/adr/0000-plantilla.md) | Plantilla para nuevas decisiones |
| [ADR-0001](docs/adr/0001-elegir-stack-tecnologico.md) | Elección del stack (propuesto) |
| [Pipeline de ingesta](docs/adr/Flujo%20y%20monitor/pipeline_rag.svg) | Flujo, ciclo de vida, control de errores y garantías del diseño |
| [Maquetas del monitor](docs/adr/Flujo%20y%20monitor/monitor) | Las cinco pantallas del gestor documental y la estructura de contenedores |
| [Lenguajes](docs/adr/Flujo%20y%20monitor/monitor/lenguaje.txt) | FastAPI para el back y Streamlit para el front |

Si usas [Claude Code](https://claude.com/claude-code), las skills del repo le dan este contexto automáticamente:

| Skill | Para qué |
| ----- | -------- |
| [`athenea`](.claude/skills/athenea/SKILL.md) | Visión general: diseño, pipeline, monitor, roles y forma de trabajar |
| [`rag-ingesta-fase1`](.claude/skills/rag-ingesta-fase1/SKILL.md) | Guía detallada para implementar la fase 1: capas, endpoints, orden de validación, errores, pruebas y definición de terminado |

## Convenciones

- **Idioma:** todo en español, desde la documentación y los mensajes hasta los estados, las colecciones, los campos y los commits.
- **Capas:** los routers no importan `pymongo`, los servicios no lanzan `HTTPException` y solo los repositorios acceden a MongoDB.
- **Validar antes de escribir:** no hay transacciones, así que se valida todo antes de guardar y, si una escritura falla, se deshace la anterior.
- **Pruebas:** cada caso de validación, permiso y versionado tiene su test. Los PDFs de prueba se generan en la propia suite y no se suben al repositorio.
- **Secretos:** nunca en el código ni en el `docker-compose.yml`. Se guardan en `.env`, que está ignorado, y se documentan en `.env.example`.

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
