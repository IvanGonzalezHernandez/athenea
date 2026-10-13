# Guía de contribución

Gracias por contribuir a Athenea. Esta guía recoge cómo trabajamos los cuatro miembros del equipo para que el repositorio se mantenga ordenado.

## Ramas

| Rama | Propósito | Protegida |
| ---- | --------- | --------- |
| `main` | Versión estable. Solo recibe merges desde `develop` o `hotfix/*`. | Sí |
| `develop` | Integración del trabajo en curso. Rama por defecto. | Sí |
| `feature/<issue>-<descripcion>` | Nueva funcionalidad. Sale de `develop`. | No |
| `fix/<issue>-<descripcion>` | Corrección de un error. Sale de `develop`. | No |
| `docs/<descripcion>` | Solo documentación. | No |
| `hotfix/<descripcion>` | Corrección urgente sobre `main`. | No |

Ejemplo: `feature/12-ingesta-pdf`.

## Flujo de trabajo

1. Asígnate un issue (o créalo) y muévelo a **En curso** en el tablero.
2. Actualiza `develop` y crea tu rama:

   ```bash
   git switch develop && git pull
   git switch -c feature/12-ingesta-pdf
   ```

3. Haz commits pequeños siguiendo [Conventional Commits](#commits).
4. Antes de abrir la PR, trae los últimos cambios de `develop`:

   ```bash
   git fetch origin && git rebase origin/develop
   ```

5. Abre una PR hacia `develop`, rellena la plantilla y enlaza el issue (`Closes #12`).
6. Necesita **1 aprobación** y la CI en verde. Se integra con **squash merge**.
7. Borra la rama tras el merge.

Las versiones se publican abriendo una PR de `develop` a `main` y creando una etiqueta `vX.Y.Z` ([SemVer](https://semver.org/lang/es/)).

## Commits

Usamos [Conventional Commits](https://www.conventionalcommits.org/es/v1.0.0/). El título de la PR debe seguir también este formato, porque será el mensaje del commit tras el squash.

```text
<tipo>(<ámbito opcional>): <descripción en imperativo>
```

| Tipo | Uso |
| ---- | --- |
| `feat` | Nueva funcionalidad |
| `fix` | Corrección de un error |
| `docs` | Documentación |
| `refactor` | Cambio de código sin cambiar comportamiento |
| `test` | Añadir o corregir pruebas |
| `chore` | Mantenimiento, dependencias, configuración |
| `ci` | Integración continua |

Ejemplos: `feat(ingesta): añade lector de PDF`, `fix(chat): evita respuestas sin fuentes`.

## Revisión de código

- Revisa el comportamiento, no solo el estilo: ¿hace lo que dice el issue?
- Comentarios concretos y amables. Usa *Suggestion* de GitHub cuando proceda.
- Quien abre la PR resuelve las conversaciones; quien revisa aprueba.

## Decisiones de arquitectura

Cualquier decisión técnica relevante (lenguaje, framework, modelo, base vectorial…) se documenta como ADR en [`docs/adr`](docs/adr) usando la [plantilla](docs/adr/0000-plantilla.md) y se aprueba mediante PR.

## Secretos

Nunca subas claves de API ni credenciales. Usa `.env` (ignorado por git) y documenta las variables en `.env.example`.
