# Proyecto Netflix: limpieza, normalización y consultas SQL

Preparación del dataset `netflix_titles.csv` (8.807 títulos) para cargarlo en una base de datos relacional (PostgreSQL, gestionada con DBeaver) y consultarlo con SQL.

## Contenido

```
.
├── README.md
├── netflix_titles_organizado.csv   # CSV único, limpio y ordenado (13 columnas)
└── csv/                            # Versión normalizada, lista para la base de datos
    ├── 00_schema.sql               # CREATE TABLE de las 10 tablas
    ├── 01_ratings.csv
    ├── 02_titles.csv
    ├── 03_directors.csv
    ├── 04_title_directors.csv
    ├── 05_actors.csv
    ├── 06_title_cast.csv
    ├── 07_countries.csv
    ├── 08_title_countries.csv
    ├── 09_genres.csv
    ├── 10_title_genres.csv
    └── 11_consultas.sql            # Consultas A y B
```

| Archivo | Cuándo usarlo |
|---|---|
| `netflix_titles_organizado.csv` | Revisar o analizar los datos, o importarlos a una sola tabla. |
| `csv/` (10 CSV + esquema) | Cargar la base de datos relacional y ejecutar las consultas. |

## Problemas del archivo original y cómo se resolvieron

| Problema | Solución |
|---|---|
| 3 filas con las columnas desplazadas (`s5542`, `s5795`, `s5814`): la duración estaba en `rating` y `duration` vacía. | Se movió el valor a `duration` y `rating` quedó vacío. |
| Fechas en texto inglés ("September 25, 2021"). | Convertidas a ISO `YYYY-MM-DD`. |
| `duration` mezclaba minutos y temporadas ("90 min", "2 Seasons"). | Se separó en `duration_value` (entero) y `duration_unit` (`min` / `seasons`). |
| Columnas con varios valores separados por coma (`director`, `cast`, `country`, `listed_in`). | Se separaron en tablas de catálogo y tablas de relación. |
| Espacios sobrantes y valores repetidos dentro de las listas. | Se recortaron y se eliminaron duplicados. |
| `cast` es palabra reservada en SQL. | Se renombró a `cast_members` en el CSV plano; en el modelo normalizado es la tabla `title_cast`. |

Los valores que faltan en el original se dejaron **vacíos (NULL)**, sin inventar datos: `director` (2.634), `cast` (825), `country` (831), `date_added` (10) y `rating` (7).

## Modelo relacional

```
ratings ──< titles >── title_directors ──> directors
              │
              ├── title_cast ──────────> actors
              ├── title_countries ─────> countries
              └── title_genres ────────> genres
```

| Tabla | Filas | Llave | Descripción |
|---|---:|---|---|
| `ratings` | 14 | `rating_id` | Clasificaciones por edad (TV-MA, PG-13, …). |
| `titles` | 8.807 | `show_id` | Tabla principal: tipo, título, fecha, año, rating, duración, descripción. |
| `directors` | 4.993 | `director_id` | Catálogo de directores. |
| `title_directors` | 6.977 | `(show_id, director_id)` | Relación título ↔ director. |
| `actors` | 36.439 | `actor_id` | Catálogo de actores. |
| `title_cast` | 64.124 | `(show_id, actor_id)` | Relación título ↔ actor. |
| `countries` | 122 | `country_id` | Catálogo de países. |
| `title_countries` | 10.012 | `(show_id, country_id)` | Relación título ↔ país. |
| `genres` | 42 | `genre_id` | Catálogo de géneros (campo `listed_in` original). |
| `title_genres` | 19.323 | `(show_id, genre_id)` | Relación título ↔ género. |

## Carga en la base de datos

1. Ejecuta `csv/00_schema.sql` para crear las tablas.
2. Importa los CSV **en este orden** (por las llaves foráneas):
   1. Catálogos: `01_ratings`, `03_directors`, `05_actors`, `07_countries`, `09_genres`
   2. Tabla principal: `02_titles`
   3. Relaciones: `04_title_directors`, `06_title_cast`, `08_title_countries`, `10_title_genres`
3. En DBeaver: clic derecho sobre la tabla → *Importar datos* → CSV. Verifica que el separador sea `,`, la codificación UTF-8 y que la primera fila se use como encabezado.

Comprobación rápida tras cargar:

```sql
SELECT 'titles' AS tabla, COUNT(*) FROM titles          -- 8807
UNION ALL SELECT 'actors', COUNT(*) FROM actors         -- 36439
UNION ALL SELECT 'title_cast', COUNT(*) FROM title_cast; -- 64124
```

## Consultas (`csv/11_consultas.sql`)

### Consulta A: catálogo cinematográfico por género
Muestra título, año de lanzamiento, clasificación por edad, duración y género de las **películas** (`type = 'Movie'`), filtradas por género con el parámetro `:genero`. Une `titles`, `title_genres`, `genres` y `ratings`. Orden: año descendente y, en empate, título alfabético.

### Consulta B: trazabilidad geográfica y de participación
Muestra título, tipo de producción, país de origen, participante, rol y año de estreno, filtrado por país con el parámetro `:pais`. El rol se deduce de la tabla de origen: `title_cast` → `Actor` y `title_directors` → `Director` (`UNION ALL`). Orden: título y rol, ambos alfabéticos.

### Parámetros según el motor

| Motor | Sintaxis |
|---|---|
| PostgreSQL (SQL nativo) | `$1` |
| DBeaver | `:nombre` |
| SQL Server | `@nombre` |
| MySQL / SQLite (conectores) | `?` |

## Errores frecuentes en DBeaver

- **`operator does not exist: character varying = integer`**: el valor del parámetro llegó como número. Escríbelo entre comillas simples.
- **`column "colombia" does not exist`**: falta escribir el valor entre comillas simples en la ventana de parámetros. Debe ser `'Colombia'`, no `Colombia`.
- **La consulta corre pero no devuelve filas**: revisa cómo está guardado el nombre con `SELECT * FROM countries WHERE name ILIKE '%colom%';`. Los países y géneros están en inglés, tal como vienen del dataset (`'Spain'`, `'Dramas'`).

## Limitaciones conocidas

- Los nombres separados por coma que en realidad son uno solo (por ejemplo, "Jr.") pueden haberse partido en dos registros en `actors` o `directors`.
- El dataset no incluye el rol de cada participante, por eso se deduce (`Actor` / `Director`).
- Un título con varios países aparece una vez por cada país en la Consulta B.
- Los géneros vienen en inglés y con el nombre original del dataset (por ejemplo, `International TV Shows`).
