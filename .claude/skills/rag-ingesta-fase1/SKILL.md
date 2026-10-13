---
name: rag-ingesta-fase1
description: "Construir la fase 1 del pipeline RAG de PDFs: API FastAPI con roles, validación de entrada, deduplicación y persistencia en MongoDB (GridFS + ficha con estado PDF OK)."
---

# Pipeline RAG de PDFs · Fase 1: entrada y persistencia

Guía para implementar la primera fase del gestor documental RAG: desde que el usuario sube un PDF hasta que queda guardado en MongoDB con su ficha en estado `PDF OK`.

## Alcance

### Dentro de esta fase

- API FastAPI con autenticación JWT y control por roles.
- Validación del fichero: firma PDF, tamaño, páginas, contraseña, corrupción.
- Deduplicación por SHA-256 del fichero.
- Persistencia del PDF original en GridFS y de su ficha en la colección `documentos`.
- Versionado (`sustituye_a`, `vigente`) y confidencialidad.
- Consulta, descarga y borrado de documentos; estadísticas por estado.
- Docker Compose con `mongo` + `api`.

### Fuera de esta fase (fase 2, no implementar)

- Celery, Redis, workers y supervisor.
- Extracción de texto, chunks, `hash_texto`, embeddings y Qdrant.
- Reintentos automáticos y reprocesado.

Deja preparado el punto de enganche (ver "Enganche con la fase 2") pero no implementes nada de la fase 2. Si una petición del usuario cae fuera del alcance, avisa antes de hacerla.

## Stack

- Python 3.11, FastAPI, Uvicorn.
- `pymongo` (síncrono) con `gridfs.GridFSBucket`. Los endpoints se declaran con `def` (no `async def`) para que FastAPI los ejecute en su threadpool.
- Pydantic v2 y `pydantic-settings` para esquemas y configuración.
- `pypdf` para abrir el PDF (páginas, cifrado, corrupción).
- `PyJWT` para tokens y `passlib[bcrypt]` para contraseñas.
- `pytest` + `httpx` (TestClient) para pruebas, contra un MongoDB real en Docker.

## Estructura del proyecto (MVC)

```text
proyecto-rag/
├── docker-compose.yml
└── api/
    ├── Dockerfile
    ├── requirements.txt
    ├── main.py                  # crea la app, registra routers, crea índices al arrancar
    ├── core/
    │   ├── config.py            # Settings desde variables de entorno
    │   ├── db.py                # cliente Mongo, base de datos y GridFSBucket
    │   ├── security.py          # JWT, hash de contraseñas, dependencias de rol
    │   └── errors.py            # excepción de dominio y manejador uniforme
    ├── models/                  # MODELO: entidades y enums
    │   ├── documento.py         # Estado, Rol, esquemas de la ficha
    │   └── usuario.py
    ├── repositories/            # MODELO: acceso a datos (único sitio que toca Mongo)
    │   ├── documentos_repo.py
    │   ├── gridfs_repo.py
    │   └── usuarios_repo.py
    ├── services/                # lógica de negocio
    │   ├── validacion.py        # comprobaciones del PDF
    │   ├── ingesta.py           # orquesta el alta de un documento
    │   └── encolado.py          # enganche con la fase 2 (de momento solo registra)
    ├── routers/                 # CONTROLADORES
    │   ├── auth.py
    │   ├── documentos.py
    │   └── estadisticas.py
    ├── scripts/
    │   └── crear_admin.py       # crea el primer usuario administrador
    └── tests/
```

### Reglas de capas (obligatorias)

- Los routers no importan `pymongo` ni tocan colecciones: llaman a servicios.
- Los servicios no conocen HTTP: lanzan `ErrorDominio`, nunca `HTTPException`.
- Solo los repositorios acceden a MongoDB y GridFS.

## Configuración (variables de entorno)

| Variable | Por defecto | Uso |
| --- | --- | --- |
| `MONGO_URL` | `mongodb://mongo:27017` | Conexión a MongoDB |
| `MONGO_DB` | `rag` | Base de datos |
| `MAX_MB` | `20` | Tamaño máximo del PDF |
| `MAX_PAGINAS` | `300` | Páginas máximas |
| `JWT_SECRET` | — (obligatoria) | Firma de tokens |
| `JWT_EXP_MIN` | `480` | Caducidad del token en minutos |

Nunca escribas secretos en el código ni en el `docker-compose.yml` versionado: usa un `.env` incluido en `.gitignore`.

## Modelo de datos

### Estados (definir todos ahora; en esta fase solo se asigna `PDF OK`)

```python
class Estado(str, Enum):
    PDF_OK = "PDF OK"
    PROCESANDO = "PROCESANDO VECTORES"
    EXITO = "SUBIDA CON ÉXITO"
    ERROR = "ERROR"
    DUPLICADO = "DUPLICADO"
    SIN_TEXTO = "SIN TEXTO"
```

### Roles

```python
class Rol(str, Enum):
    ADMIN = "administracion"
    EDITOR = "editor"
    LECTURA = "lectura"
```

### Colección `documentos` (una ficha por PDF)

| Campo | Tipo | Fase 1 |
| --- | --- | --- |
| `doc_id` | str (uuid4) | Se genera al subir |
| `gridfs_id` | ObjectId | Enlace al PDF en GridFS |
| `nombre_original` | str | Nombre del fichero subido |
| `hash_sha256` | str | Huella del fichero (único) |
| `tamano_bytes`, `paginas` | int | Del PDF validado |
| `tipo`, `departamento`, `curso`, `autor` | str | Metadatos del formulario |
| `fecha_documento` | date | Metadato del formulario |
| `confidencial` | bool | Por defecto `false` |
| `version` | int | 1, o versión anterior + 1 si sustituye |
| `vigente` | bool | `true` al subir |
| `sustituye_a` | str \| null | `doc_id` de la versión anterior |
| `estado` | Estado | `PDF OK` |
| `etapa_error`, `error_msg` | null | Reservados para fase 2 |
| `intentos` | int | `0` |
| `inicio_proceso`, `modelo_embed`, `hash_texto` | null | Reservados para fase 2 |
| `subido_por` | str | Usuario del token |
| `fecha_subida`, `fecha_actualizacion` | datetime UTC | Al crear |
| `historial` | list | `[{estado, fecha, detalle}]` |

Crea los campos reservados con `null` desde ahora para que la fase 2 no tenga que migrar fichas.

### Índices (crear al arrancar la app, de forma idempotente)

- `doc_id` único.
- `hash_sha256` único. Es lo que garantiza la deduplicación incluso con dos subidas simultáneas.
- `estado`.
- `(tipo, departamento, vigente)`.
- `fecha_subida` descendente.

### Colección `usuarios`

`usuario` (único), `nombre`, `departamento`, `rol`, `password_hash`, `activo`.

## Permisos por rol

| Acción | Administración | Editor | Lectura |
| --- | --- | --- | --- |
| Consultar y descargar documentos | Sí | Sí | Sí, sin confidenciales |
| Subir PDF | Sí | Sí | No |
| Eliminar documento | Sí | No | No |
| Ver estadísticas | Sí | Sí | Sí |

Los permisos se comprueban **en la API** con una dependencia (`requiere_rol(Rol.ADMIN, Rol.EDITOR)`), nunca solo en la interfaz. Sin token o con token inválido: 401. Rol insuficiente: 403.

## Endpoints

| Método y ruta | Rol | Respuesta correcta |
| --- | --- | --- |
| `POST /auth/login` | público | 200 `{access_token, rol}` |
| `POST /documentos` | admin, editor | 202 `{doc_id, estado: "PDF OK"}` |
| `GET /documentos` | todos | 200, lista paginada con filtros `estado`, `tipo`, `departamento`, `vigente` |
| `GET /documentos/{doc_id}` | todos | 200, ficha completa |
| `GET /documentos/{doc_id}/pdf` | todos | 200, `application/pdf` en streaming |
| `DELETE /documentos/{doc_id}` | admin | 204 |
| `GET /estadisticas` | todos | 200, total por estado |

Para el rol `lectura`, los listados y consultas excluyen `confidencial: true`, y pedir uno confidencial por id devuelve 404 (no 403, para no revelar que existe).

`POST /documentos` recibe `multipart/form-data`: el fichero `file` y los campos `tipo`, `departamento`, `curso`, `fecha_documento`, `autor`, `confidencial` y `sustituye_a` (opcional).

## Flujo de `POST /documentos` (orden obligatorio)

El orden importa porque no hay transacciones: se valida todo antes de escribir nada, y si una escritura falla se deshace la anterior.

1. **Rol**: admin o editor; si no, 403.
2. **Metadatos**: validados por Pydantic; si faltan o son inválidos, 422.
3. **Tamaño**: leer el fichero por bloques y cortar al superar `MAX_MB`, sin cargarlo entero antes de comprobarlo; si lo supera, 413.
4. **Firma**: los primeros bytes deben ser `%PDF-`; si no, 400 `NO_ES_PDF`. Nunca te fíes de la extensión ni del `content_type`.
5. **Apertura con pypdf**: si está cifrado, 400 `PDF_PROTEGIDO`; si falla al abrir, 400 `PDF_CORRUPTO`; si supera `MAX_PAGINAS`, 400 `DEMASIADAS_PAGINAS`.
6. **SHA-256** de los bytes.
7. **Duplicado**: si ya existe una ficha con ese hash, 409 `DUPLICADO` con el `doc_id` y el nombre del documento existente.
8. **`sustituye_a`**, si viene: debe existir y estar vigente; si no, 422.
9. **Guardar en GridFS** con metadatos mínimos (`doc_id`, `hash_sha256`). Si falla, 500 `ERROR_ALMACENAMIENTO`: no se ha escrito nada.
10. **Insertar la ficha** con `estado: "PDF OK"` y la primera entrada del historial.
    - Si salta `DuplicateKeyError` en `hash_sha256` (dos subidas simultáneas del mismo fichero): borrar el fichero de GridFS y devolver 409.
    - Cualquier otro fallo: borrar el fichero de GridFS y devolver 500.
11. **Versionado**: si hay `sustituye_a`, marcar la versión anterior con `vigente: false` y fijar `version = anterior + 1`.
12. **Enganche fase 2**: llamar a `encolar_procesado(doc_id)`.
13. Responder **202** con `{doc_id, estado}`.

### Esqueleto del servicio

```python
def dar_de_alta(fichero: bytes, nombre: str, meta: MetadatosIn, usuario: Usuario) -> Ficha:
    info = validacion.validar_pdf(fichero)            # pasos 3-5, lanza ErrorDominio
    sha = hashlib.sha256(fichero).hexdigest()
    if (existente := documentos_repo.por_hash(sha)):
        raise ErrorDominio(409, "DUPLICADO", "El documento ya existe",
                           {"doc_id": existente.doc_id, "nombre": existente.nombre_original})
    anterior = _comprobar_sustitucion(meta.sustituye_a)

    doc_id = str(uuid4())
    gridfs_id = gridfs_repo.guardar(fichero, nombre, {"doc_id": doc_id, "hash_sha256": sha})
    try:
        ficha = documentos_repo.insertar(_nueva_ficha(doc_id, gridfs_id, sha, info, meta, usuario, anterior))
    except DuplicateKeyError:
        gridfs_repo.borrar(gridfs_id)
        raise ErrorDominio(409, "DUPLICADO", "El documento ya existe")
    except Exception:
        gridfs_repo.borrar(gridfs_id)
        raise

    if anterior:
        documentos_repo.marcar_no_vigente(anterior.doc_id)
    encolado.encolar_procesado(doc_id)
    return ficha
```

## Borrado en cascada (`DELETE /documentos/{doc_id}`)

Orden: primero el fichero de GridFS, después la ficha. Así, si algo falla a mitad, queda una ficha apuntando a un fichero inexistente (detectable y reintentable) en lugar de un fichero huérfano invisible. Dejar un comentario `# FASE 2: borrar también los vectores de Qdrant por doc_id` en el servicio de borrado.

## Errores

Todas las respuestas de error tienen el mismo formato:

```json
{ "codigo": "DUPLICADO", "mensaje": "El documento ya existe", "detalle": { "doc_id": "…", "nombre": "…" } }
```

| HTTP | Código | Cuándo |
| --- | --- | --- |
| 400 | `NO_ES_PDF`, `PDF_PROTEGIDO`, `PDF_CORRUPTO`, `DEMASIADAS_PAGINAS` | Validación del fichero |
| 401 | `NO_AUTENTICADO` | Sin token o token inválido o caducado |
| 403 | `SIN_PERMISO` | Rol insuficiente |
| 404 | `NO_ENCONTRADO` | doc_id inexistente o confidencial para lectura |
| 409 | `DUPLICADO` | Hash ya existente |
| 413 | `DEMASIADO_GRANDE` | Supera `MAX_MB` |
| 422 | `METADATOS_INVALIDOS` | Formulario incorrecto o `sustituye_a` no válido |
| 500 | `ERROR_ALMACENAMIENTO` | Fallo al escribir en GridFS o en la colección |

Los mensajes van en español, dicen qué ha pasado y cómo solucionarlo, y nunca exponen trazas internas.

## Enganche con la fase 2

`services/encolado.py` expone una única función:

```python
def encolar_procesado(doc_id: str) -> None:
    """Fase 1: solo registra. Fase 2: enviará la tarea a Celery (vectorizar.delay)."""
    logger.info("Documento %s listo para procesar (pendiente fase 2)", doc_id)
```

No importes Celery ni Redis en esta fase.

## Docker Compose (fase 1)

Dos servicios: `mongo` (imagen oficial `mongo:7`, volumen `mongo_data`, puerto 27017) y `api` (build `./api`, puerto 8000, `depends_on: mongo`, variables desde `.env`). La API se conecta a `mongodb://mongo:27017`, no a `localhost`. Arranque con `docker compose up -d --build`.

## Pruebas mínimas (pytest)

Cada caso debe tener su test, contra el MongoDB del compose y una base de datos de pruebas que se vacía entre tests:

1. PDF válido como editor: 202, ficha con `estado: "PDF OK"`, `gridfs_id` que existe en GridFS, `intentos: 0`.
2. Imagen renombrada a `.pdf`: 400 `NO_ES_PDF` y nada escrito.
3. Fichero mayor que `MAX_MB`: 413.
4. PDF con contraseña: 400 `PDF_PROTEGIDO`.
5. PDF truncado: 400 `PDF_CORRUPTO`.
6. El mismo PDF con otro nombre: 409 con el `doc_id` original, y solo un fichero en GridFS.
7. Rol lectura intentando subir: 403. Sin token: 401.
8. Subida con `sustituye_a`: la nueva queda `vigente` con `version` 2 y la anterior `vigente: false`.
9. Rol lectura listando: no ve documentos confidenciales.
10. Borrado como admin: desaparecen la ficha y el fichero de GridFS. Como editor: 403.
11. Si falla la inserción de la ficha (simulado con mock), no queda el fichero en GridFS.

Genera los PDFs de prueba en la propia suite (por ejemplo, con `pypdf` o `reportlab`), no los subas al repositorio.

## Definición de terminado

- [ ] `docker compose up` levanta `mongo` y `api` sin pasos manuales, salvo crear el `.env`.
- [ ] `scripts/crear_admin.py` crea el primer administrador.
- [ ] Todos los tests pasan.
- [ ] La documentación automática en `/docs` muestra todos los endpoints con ejemplos.
- [ ] Ningún router accede directamente a MongoDB.
- [ ] Los índices se crean solos al arrancar.
- [ ] El README explica cómo arrancar, crear el admin y ejecutar los tests.
