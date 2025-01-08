---
description: Comment utiliser DuckDB pour interroger OpenStreetMap ?
tags: ['article','osm','duckdb']
---

# DuckDB et OSM

J'ai découvert DuckDB en même temps que l'outil QuackOSM grâce à [cet article](https://m-weigand.de/posts/07_duckdb-a_game_changer_for_analyzing_openstreetmap) de Matthias Weigand. Cet article/prise de note reprend donc énormément du sien.

## C'est quoi DuckDB ?

DuckDB est un SGBDR qui à l'instar de SQLite/Spatialite ne repose pas (nécessairement) sur un serveur et peut être embarqué dans un script ou une application. Une base de données est ainsi un simple fichier portable d'extension `.duckdb`. Au contraire de SQLite, il est optimisé pour l'analyse et on peut donc charger dans cet objectif un gros volume de données dans une base. 

Sous macOS, on peut l'installer avec Homebrew:

```sh
brew install duckdb
```

## Analyser OpenStreetMap

DuckDB, à l'instar de PostgreSQL, dispose d'une extension spatiale et permet donc de traiter facilement des données géographiques. Pour l'installer et l'activer, on peut lancer DuckDB (`duckdb`) puis faire:

```sql
D INSTALL spatial;
D LOAD spatial;
```

L'extension spatiale dispose d'une fonction permettant d'ouvrir et d'exécuter des requêtes directement sur un fichier `.osm.pbf` ! C'est une possibilité intéressante, mais un peu trop "brute": on ne dispose que de la géométrie des nodes, et il faut alors recomposer soi-même celle des ways et des relations.

Pour répondre à cette problématique, Matthias Weigand présente l'outil Python QuackOSM. C'est un peu un équivalent à osm2pgsql côté PostgreSQL, mais pour DuckDB: il permet la transformation d'un fichier `.osm.pbf` en Geoparquet, chargeable facilement dans DuckDB. QuackOSM crée une unique table qui contient pour chaque entité son id OSM, ses tags sous la forme d'une liste de listes et sa géométrie recomposée (simple ou multiple).

Pour installer QuackOSM:

```sh
pip install quackosm[cli]
```

### Charger des données OSM dans DuckDB

On trouve le monde entier au format `.osm.pbf` découpé par région sur [Geofabrik](https://download.geofabrik.de/). On pourrait prendre une région plus petite, automatiquement moins gourmande, mais ici on va travailler sur la France entière:

```sh
curl https://download.geofabrik.de/europe/france-latest.osm.pbf -O
```

On transforme le fichier OSM en fichier Geoparquet avec QuackOSM:

```sh
quackosm france-latest.osm.pbf
```

La transformation est _longue_. De mon côté, il a fallu patienter environ 22 minutes. À la fin, QuackOSM crée le fichier `files/france-latest_nofilter_noclip_compact.geoparquet`.

On crée une base de données dans le fichier `osm.duckdb` (si on ne crée pas de fichier, la base sera temporaire):

```sh
duckdb osm.duckdb
```

Enfin, on charge le Geoparquet dans une nouvelle table `osm`:

```sql
D CREATE TABLE osm AS
SELECT * 
FROM read_parquet('files/france-latest_nofilter_noclip_compact.geoparquet');
```

### Interroger les données

La table `osm` créée précédemment embarque 3 champs:

```sql
D DESCRIBE osm;
┌─────────────┬───────────────────────┬─────────┬─────────┬─────────┬─────────┐
│ column_name │      column_type      │  null   │   key   │ default │  extra  │
│   varchar   │        varchar        │ varchar │ varchar │ varchar │ varchar │
├─────────────┼───────────────────────┼─────────┼─────────┼─────────┼─────────┤
│ feature_id  │ VARCHAR               │ YES     │         │         │         │
│ tags        │ MAP(VARCHAR, VARCHAR) │ YES     │         │         │         │
│ geometry    │ GEOMETRY              │ YES     │         │         │         │
└─────────────┴───────────────────────┴─────────┴─────────┴─────────┴─────────┘
Run Time (s): real 0.005 user 0.001717 sys 0.003183
```

Le champ tags est en fait un dictionnaire de listes. Pour appeler la valeur d'un tag, on peut faire `tags['key'][1]`. On peut aussi, pour vérifier sa valeur, utiliser la fonction `list_contains(tags['key'], 'value')`.

Ici, un exemple de requête pour obtenir le nombre de boulangeries par région:

```sql
D WITH regions as (
	SELECT tags['name'] as name, geometry 
	FROM osm 
	WHERE list_contains(tags['admin_level'], '4') AND list_contains(tags['boundary'], 'administrative') AND ST_GeometryType(geometry) IN ['POLYGON','MULTIPOLYGON']
)
SELECT name[1] as region_name, COUNT(*) as nb_boulangeries, regions.geometry
FROM osm
INNER JOIN regions ON ST_CONTAINS(regions.geometry, osm.geometry)
WHERE list_contains(osm.tags['shop'], 'bakery')
GROUP BY region_name, regions.geometry;
```

Résultat:

```sql
┌──────────────────────┬─────────────────┬─────────────────────────────────────┐
│     region_name      │ nb_boulangeries │              geometry               │
│       varchar        │      int64      │              geometry               │
├──────────────────────┼─────────────────┼─────────────────────────────────────┤
│ Grand Est            │            2254 │ POLYGON ((3.3843811 48.4780187, 3…  │
│ Centre-Val de Loire  │            1072 │ POLYGON ((0.053532 47.1993113, 0.…  │
│ Normandie            │            1543 │ MULTIPOLYGON (((0.1341337 49.4285…  │
│ Corse                │             171 │ MULTIPOLYGON (((9.4716379 42.9833…  │
│ Pays de la Loire     │            1753 │ MULTIPOLYGON (((-2.3071691 47.025…  │
│ Nouvelle-Aquitaine   │            2975 │ MULTIPOLYGON (((-1.7827806 43.359…  │
│ Bretagne             │            1615 │ MULTIPOLYGON (((-1.8268825 48.681…  │
│ Île-de-France        │            4388 │ POLYGON ((1.447346 49.0535756, 1.…  │
│ Hauts-de-France      │            1882 │ MULTIPOLYGON (((1.6610204 50.1794…  │
│ Auvergne-Rhône-Alpes │            3925 │ POLYGON ((2.063551 44.9766576, 2.…  │
│ Occitanie            │            2885 │ MULTIPOLYGON (((-0.3252919 42.916…  │
│ Bourgogne-Franche-…  │            1481 │ POLYGON ((2.8444817 47.5448761, 2…  │
│ Provence-Alpes-Côt…  │            2386 │ MULTIPOLYGON (((7.3923513 43.7200…  │
├──────────────────────┴─────────────────┴─────────────────────────────────────┤
│ 13 rows                                                            3 columns │
└──────────────────────────────────────────────────────────────────────────────┘
Run Time (s): real 17.081 user 48.228187 sys 14.968477
```

Il est possible d'exporter le résultat de la requête en GeoJSON:

```sql
COPY (
	WITH regions as (
	SELECT tags['name'] as name, geometry 
	FROM osm 
	WHERE list_contains(tags['admin_level'], '4') AND list_contains(tags['boundary'], 'administrative') AND ST_GeometryType(geometry) IN ['POLYGON','MULTIPOLYGON']
	)
	SELECT name[1] as region_name, COUNT(*) as nb_boulangeries, regions.geometry
	FROM osm
	INNER JOIN regions ON ST_CONTAINS(regions.geometry, osm.geometry)
	WHERE list_contains(osm.tags['shop'], 'bakery')
	GROUP BY region_name, regions.geometry
) 
TO 'output.geojson'
WITH (FORMAT GDAL, DRIVER 'GeoJSON', LAYER_CREATION_OPTIONS 'WRITE_BBOX=YES');
```

En suivant le même schéma, on peut exporter dans d'autres formats supportés par le driver GDAL, comme le Geopackage (Driver `'GPKG'`).

## ⚠️ À noter

### Empreinte mémoire

En chargeant une grosse donnée (par exemple, la France entière), DuckDB va prendre ses aises en RAM. On peut limiter la quantité utilisée par DuckDB sur chaque session:

```sql
SET memory_limit='2GB';
```
Avec 2 GB, les performances restent similaires à une ou deux secondes près sur ma machine et je n'ai pas rencontré de problème d'allocation.

### Temps d'exécution

On peut activer l'affichage du temps d'exécution de chaque requête de la session avec la commande:

```sql
.timer on
```
