# Pipeline de ingesta documental

Resumen de `docs/adr/Flujo y monitor/pipeline_rag.svg`. Es una arquitectura asíncrona y no transaccional, con validación, deduplicación y reprocesado controlado.

## Índice

- [Flujo paso a paso](#flujo-paso-a-paso)
- [Colección documentos](#colección-documentos)
- [Ciclo de vida: campo estado](#ciclo-de-vida-campo-estado)
- [Reintento automático y reprocesado manual](#reintento-automático-y-reprocesado-manual)
- [Supervisor programado](#supervisor-programado)
- [Borrado](#borrado)
- [Garantías del diseño](#garantías-del-diseño)
- [Parámetros por defecto](#parámetros-por-defecto)

## Flujo paso a paso

1. **Cliente o interfaz web.** Un usuario autenticado y con rol sube el PDF con sus metadatos: tipo, fecha, curso, departamento y autor. Puede marcarlo como confidencial e indicar a qué documento sustituye.
2. **API FastAPI, `POST /documentos`.** No procesa el PDF. Valida, guarda y responde enseguida con `202` y el `doc_id`.
3. **Validación de entrada**, antes de guardar nada:
   - Firma `%PDF` real, no solo la extensión.
   - Máximo 20 MB y 300 páginas.
   - Sin contraseña y sin corrupción.
   - Huella SHA-256 única, con índice único en MongoDB.
   - Respuestas: `400` si el fichero no es válido, `409` si es un duplicado (devuelve el `doc_id` del original) y `500` si falla el guardado en GridFS (el usuario repite la subida).
4. **GridFS** guarda el PDF original sin modificar. Es la fuente de verdad, y cualquier reprocesado parte de ahí sin pedir nada al usuario.
5. **Colección `documentos`.** Se crea la ficha, enlazada al PDF mediante `gridfs_id`, con `estado = PDF OK`.
6. **Cola de tareas (Celery + Redis).** Recibe el `doc_id`. Ante fallos temporales reintenta hasta 3 veces con espera creciente de 1, 5 y 15 minutos, sin bloquear al resto.
7. **Worker:**
   - Pone `estado = PROCESANDO VECTORES`.
   - Lee el PDF de GridFS, extrae el texto y lo divide en chunks (800 tokens con 100 de solapamiento).
   - Si no hay texto, marca `SIN TEXTO` (requiere OCR). Si `hash_texto` ya existe en otro documento, marca `DUPLICADO`. En ninguno de los dos casos se vectoriza.
8. **Modelo de embeddings.** Convierte cada chunk en un vector y guarda la versión en `modelo_embed`, para saber qué hay que reindexar si cambia el modelo.
9. **Qdrant.** Upsert idempotente con `id = uuid5(doc_id, nº de chunk)`, así que repetir el proceso sobrescribe en vez de duplicar. El payload incluye `vigente` y `confidencial` para filtrar. Al terminar, `estado = SUBIDA CON ÉXITO`.

## Colección documentos

| Campo | Contenido |
| ----- | --------- |
| `doc_id` | Identificador del documento |
| `gridfs_id` | Enlace al PDF en GridFS |
| `hash_sha256` | Huella del fichero, con índice único (duplicado exacto) |
| `hash_texto` | Huella del texto extraído (duplicado por contenido) |
| `tipo`, `fecha`, `autor`, `curso`, `departamento` | Metadatos de la subida |
| `estado` | Ver el ciclo de vida |
| `etapa_error`, `error_msg` | Dónde y por qué falló |
| `intentos` | Número de intentos (informativo) |
| `inicio_proceso` | Cuándo empezó el procesado; lo usa el supervisor |
| `modelo_embed` | Modelo de embeddings con el que se indexó |
| `version`, `vigente` | Versionado; al sustituir un documento, el anterior pasa a no vigente |
| `confidencial` | Si es verdadero, queda fuera del RAG |

## Ciclo de vida: campo estado

```text
PDF OK ──► PROCESANDO VECTORES ──► SUBIDA CON ÉXITO
                │
                ├──► DUPLICADO   (mismo texto que otro documento)
                ├──► SIN TEXTO   (escaneado, requiere OCR)
                └──► ERROR       (reintentos agotados o timeout del supervisor)

ERROR ──► Reprocesar (manual) ──► se borran sus vectores ──► vuelve a la cola (paso 6)
```

## Reintento automático y reprocesado manual

| Aspecto | Automático (cola de tareas) | Manual (botón Reprocesar) |
| ------- | --------------------------- | ------------------------- |
| Para qué fallos | Temporales: servicio caído, corte de red, límite de peticiones | Persistentes: alguien tiene que arreglar algo antes |
| Ejemplos | Qdrant reiniciándose, error 429 de la API de embeddings | Bug ya corregido, servidor de embeddings caído toda la tarde, cambio de modelo |
| Quién actúa | Nadie, se resuelve solo | Una persona, después de arreglar la causa |
| Límite | 3 reintentos con espera creciente; si se agotan, pasa a `ERROR` | Sin límite automático; `intentos` queda como información |

## Supervisor programado

Es una tarea de Celery `beat` que se ejecuta cada 5 minutos y hace dos cosas:

1. Pasa a `ERROR` los documentos que llevan más de 15 minutos en `PROCESANDO VECTORES`, por ejemplo porque el worker se cayó a mitad.
2. Reconcilia MongoDB y Qdrant: elimina los vectores cuyo `doc_id` ya no existe en la colección, que son restos de borrados a medias.

## Borrado

`DELETE /documentos/{doc_id}` borra en cascada la ficha de la colección, el PDF de GridFS y todos sus vectores de Qdrant, filtrando por `doc_id`. Solo afecta a ese documento. Si se queda a medias, por ejemplo porque Qdrant no responde, el supervisor limpia después los restos.

## Garantías del diseño

Cualquier cambio tiene que mantener estas garantías:

- El PDF original nunca se pierde.
- No hay duplicados ni en MongoDB ni en Qdrant.
- Ningún documento queda bloqueado.
- Los errores se pueden rastrear por etapa.
- Los documentos confidenciales quedan fuera del RAG.
- La API no se bloquea.

## Parámetros por defecto

Son los de la pantalla de Configuración y se pueden cambiar desde ella:

| Parámetro | Valor |
| --------- | ----- |
| Tamaño máximo | 20 MB |
| Páginas máximas | 300 |
| Detección de duplicados (SHA-256) | Activa |
| Duplicados por contenido | Activa |
| Modelo de embeddings | `multilingual-e5-base` (cambiarlo obliga a reindexar todo) |
| Tamaño de chunk / solapamiento | 800 / 100 tokens |
| OCR para escaneados | Desactivado, por eso se marcan como `SIN TEXTO` |
| Reintentos automáticos | 3, con esperas de 1, 5 y 15 min |
| Tiempo máximo en PROCESANDO | 15 min |
| Frecuencia del supervisor | 5 min |
