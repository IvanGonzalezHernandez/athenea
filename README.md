# Athenea

> Asistente educativo basado en RAG (*Retrieval-Augmented Generation*): responde a las dudas del alumnado apoyándose en el material del curso y citando siempre sus fuentes.

[![CI](https://github.com/IvanGonzalezHernandez/athenea/actions/workflows/ci.yml/badge.svg)](https://github.com/IvanGonzalezHernandez/athenea/actions/workflows/ci.yml)
[![Licencia: MIT](https://img.shields.io/badge/licencia-MIT-blue.svg)](LICENSE)

## Índice

- [Visión](#visión)
- [Funcionalidades previstas](#funcionalidades-previstas)
- [Arquitectura](#arquitectura)
- [Puesta en marcha](#puesta-en-marcha)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Cómo contribuir](#cómo-contribuir)
- [Equipo](#equipo)
- [Licencia](#licencia)

## Visión

Athenea permite cargar apuntes, libros y documentos de una asignatura y conversar con ellos. Cada respuesta se genera a partir de los fragmentos recuperados del propio material, de forma que el alumno pueda comprobar de dónde sale la información.

## Funcionalidades previstas

- [ ] Ingesta de documentos (PDF, Markdown, DOCX) y troceado en fragmentos
- [ ] Generación de *embeddings* y almacenamiento en una base de datos vectorial
- [ ] Recuperación de los fragmentos más relevantes para cada pregunta
- [ ] Generación de respuestas con cita de las fuentes
- [ ] Interfaz de chat para el alumnado
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

Las decisiones técnicas (stack, modelos, base vectorial…) se documentan como ADR en [`docs/adr`](docs/adr).

## Puesta en marcha

> Pendiente de definir el stack. Esta sección se completará en cuanto se acepte el ADR correspondiente.

```bash
git clone https://github.com/IvanGonzalezHernandez/athenea.git
cd athenea
cp .env.example .env   # y rellenar las variables
```

## Estructura del repositorio

```text
.
├── .github/          # Plantillas de issues y PR, CI, CODEOWNERS
├── docs/
│   └── adr/          # Registro de decisiones de arquitectura
├── .env.example      # Variables de entorno necesarias (sin secretos)
├── CONTRIBUTING.md   # Flujo de trabajo del equipo
└── README.md
```

## Cómo contribuir

Lee [CONTRIBUTING.md](CONTRIBUTING.md) antes de abrir tu primera rama. En resumen: rama `feature/*` desde `develop`, commits convencionales y PR con al menos una revisión.

## Equipo

| Nombre | GitHub | Rol |
| ------ | ------ | --- |
| Iván González | [@IvanGonzalezHernandez](https://github.com/IvanGonzalezHernandez) | |
| | | |
| | | |
| | | |

## Licencia

Distribuido bajo licencia MIT. Consulta [LICENSE](LICENSE).
