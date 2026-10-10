# Bitácora — Moodle local: servicio, cuestionario y encuestas de criticidad (SBIRE)

- **Fecha:** 2026-10-10 (sede: América/Argentina, UTC-3)
- **Entorno:** Docker Desktop + WSL2 (Ubuntu-22.04), stack `local-moodle` (Moodle 4.4.2)
- **Repo:** `git@github.com:vtn-hz/local-moodle.git` (este repo solo versiona `config.php`, `README.md`, `.gitignore` y `local/`; el core está ignorado con `/*`)

> ⚠️ A diferencia de este documento, **todo el trabajo realizado vive en la base de datos de Moodle** (contenedores `moodle_app` + `moodle_db`), no en archivos. Por eso este `.md` documenta qué se hizo, dónde quedó y cuánto tomó.

---

## 1. Lo que se hizo

### 1.1 Levantar el servicio
- Se detectó que Docker Desktop corría desde Windows con **bind-mount de `moodle-src` vacío** (el contenedor no veía el código de Moodle → `/var/www/html` vacío, HTTP 403/404).
- **Fix:** se recrearon los contenedores usando el cliente `docker` nativo de WSL (`docker compose up -d --build`), con lo que el bind mount `./moodle-src:/var/www/html` quedó correcto.
- Resultado: `moodle_app` y `moodle_db` `Up (healthy)`, sitio operativo en <http://localhost:8080>.
- Acceso: usuario `admin` / `Admin1234!`.

### 1.2 Crear un curso + cuestionario (sin plugin)
- Script CLI que usa **solo API interna del core** (no es plugin): `scripts/seed_course_quiz.php`.
- Crea un curso y un **Cuestionario (mod_quiz)** calificable con `create_course()`, `add_moduleinfo()` (→ `quiz_add_instance`), `question_bank::get_qtype(...)->save_question()` y `quiz_add_quiz_question()`.
- Resultado:
  - Curso **Administración Empresarial** (id=2) → <http://localhost:8080/course/view.php?id=2>
  - Cuestionario **Cuestionario Diagnóstico** (id=1, cmid=2, sumgrades 5.0) → <http://localhost:8080/mod/quiz/view.php?id=2>
  - 5 preguntas (4 opción múltiple + 1 verdadero/falso) en el banco de preguntas del curso.

### 1.3 Crear las 2 encuestas de criticidad (sin plugin)
- Script CLI con la misma técnica: `scripts/seed_feedback_surveys.php` (módulo **Feedback/mod_feedback**, sin nota).
- Resultado en el curso id=2:

| Encuesta | Preguntas | URL |
|---|---|---|
| **Encuesta Inicial (Primer Año) - SBIRE** | 16 (n.º 1–16) | <http://localhost:8080/mod/feedback/view.php?id=3> |
| **Encuesta Cuatrimestral - SBIRE** | 12 (n.º 17–28) | <http://localhost:8080/mod/feedback/view.php?id=4> |

- Tipos usados: `multichoice` radio (`r>>>>>`); casillas múltiples (`c>>>>>`) para las preguntas 11, 23 y 24; `numeric` (2000–2026) para "Año de ingreso"; `textarea` para "Comentarios". Sin respuestas correctas ni calificación.

---

## 2. Dónde quedó guardada cada cosa

| Qué | Dónde |
|---|---|
| Servicio | Contenedores `moodle_app` / `moodle_db` (Docker) |
| Curso, encuestas y preguntas | **Base de datos** `moodle` → tablas `mdl_course`, `mdl_feedback`, `mdl_feedback_item`, `mdl_quiz`, `mdl_quiz_slots`, `mdl_question*`, `mdl_course_modules` |
| Scripts reutilizables | `scripts/seed_course_quiz.php` y `scripts/seed_feedback_surveys.php` (en la raíz del proyecto, **fuera** del repo) |
| Este documento | `local/md_SBIRE_bitacora_2026-10-10.md` (versionado en el repo) |

> Los scripts de `scripts/` se copian al contenedor para ejecutarse:
> ```
> docker cp scripts/seed_feedback_surveys.php moodle_app:/var/www/
> docker exec moodle_app php /var/www/seed_feedback_surveys.php
> ```
> Ambos son idempotentes (no duplican actividad/preguntas si ya existen).

---

## 3. Tiempo estimado

Estimado a partir de *timestamps* de la sesión (archivos + start de contenedores 14:30–14:55, UTC-3).

| Tarea | Tiempo estimado |
|---|---|
| Diagnóstico del entorno Docker/WSL y arranque del servicio (fix bind-mount) | ~20 min |
| Script curso + cuestionario (investigación de API interna + creación + verificación) | ~15 min |
| Script de las 2 encuestas Feedback (28 preguntas) + verificación | ~12 min |
| **Total** | **~50 min** |

*Tiempos aproximados; la mayor parte fue relevamiento de las APIs internas de Moodle 4.4.2.*