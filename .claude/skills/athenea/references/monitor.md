# Monitor: gestor documental RAG

Maquetas en `docs/adr/Flujo y monitor/monitor/`. El monitor es el front de Streamlit, se titula "Gestor documental RAG" y tiene una barra superior con cinco secciones. A la derecha de la barra se muestran el usuario, el departamento y el rol (por ejemplo, "Secretaría · Administración").

## Pantallas

| Pantalla | Captura | Qué muestra y qué permite |
| -------- | ------- | ------------------------- |
| Panel | `1-dashboard.png` | Tarjetas de documentos totales, disponibles en el chatbot, en proceso, en error y tiempo medio de procesado. Gráficas de documentos por estado, errores por etapa y actividad diaria. Tabla de documentos con error con botón Reprocesar y Reprocesar todos. Bloque "Requieren revisión" con `SIN TEXTO` y `DUPLICADO`. Filtros por periodo y departamento, y Exportar informe |
| Documentos | `2-documentos.png` | Tabla paginada con estado, vigente, acceso (confidencial), versión, chunks y fecha de subida, con acciones Ver, Descargar y Eliminar. Búsqueda y filtros por estado, tipo, departamento y vigencia. Panel lateral con la ficha del documento: metadatos, tamaño y páginas, chunks, modelo, SHA-256 e historial de estados |
| Subir PDF | `3-subir PDF.png` | Zona de arrastre (un PDF por subida, hasta 20 MB) y formulario con tipo, departamento, curso académico, fecha, autor, "Sustituye a" (opcional) y el interruptor de confidencial. Lista de las comprobaciones previas y "Tus subidas recientes" con progreso de chunks y motivos de rechazo |
| Errores | `4-control errores.png` | Recuento por etapa (Extracción, Embeddings, Qdrant). Tabla con selección múltiple y Reprocesar seleccionados. Detalle del error con el registro de intentos y sus esperas. Lista de reprocesados recientes |
| Configuración | `5-configuracion.png` | Solo para Administración. Parámetros de validación, indexación, reintentos y supervisión (ver `pipeline.md`), y gestión de usuarios y permisos |

Las etapas de error que muestra la interfaz son Extracción, Embeddings y Qdrant. Al guardar `etapa_error`, usa valores que se correspondan con ellas.

## Roles

| Rol | Permisos |
| --- | -------- |
| Administración | Subir, eliminar, reprocesar y configurar |
| Editor | Subir y reprocesar |
| Lectura | Solo el chatbot, sin documentos confidenciales (alumnado y familias) |

## Etiquetas de estado en la interfaz

`SUBIDA CON ÉXITO` (verde), `PROCESANDO` (ámbar), `ERROR` (rojo), y `SIN TEXTO` y `DUPLICADO` (gris). Al subir, los ficheros que no pasan la validación aparecen como `RECHAZADO`, con el motivo. Ese estado no se guarda en la colección, porque el fichero nunca llega a guardarse.
