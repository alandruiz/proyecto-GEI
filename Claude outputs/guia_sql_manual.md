# Guía: armar el modelo SQL a mano (notebook nuevo)

Objetivo: escribir vos mismo, en un notebook nuevo, el mismo modelo en estrella y las mismas 7 vistas que ya verificamos — para que quede como tu trabajo y lo puedas explicar en una entrevista. Yo ya dejé `sql/proyecto_gei.db` armado como referencia por si en algún paso el resultado no te cierra; podés compararlo, pero la idea es que este notebook nuevo sea el que realmente uses.

Al final vas a tener 10 CSV en `data/powerbi/` para importar a Power BI a mano, igual que en tu proyecto de Dengue.

---

## Paso 0 — Preparar el notebook

Creá `sql/02_modelado_sql.ipynb` (mismo estilo que tu notebook de EDA). Primera celda:

```python
import sqlite3
import pandas as pd
```

---

## Paso 1 — Cargar el CSV limpio

```python
df = pd.read_csv("../data/processed/emisiones_datos_totales_1990_2022_coma.csv")
df = df.rename(columns={
    "año": "anio",
    "valor_en_toneladas_de_co2e": "valor_tco2e",
})
df.shape
```

**Por qué renombro `año` → `anio` y el nombre de la columna de valor:** en SQL es más simple trabajar sin tildes en los nombres de columna (evita tener que escribir `"año"` entre comillas en cada consulta), y `valor_tco2e` es más corto para escribir en las vistas.

Verificá que te da `(7737, 9)` — si no, algo cambió en el CSV limpio y conviene revisar antes de seguir.

---

## Paso 2 — Crear la base y conectar

```python
conn = sqlite3.connect("../sql/proyecto_gei.db")
cur = conn.cursor()
```

---

## Paso 3 — Crear el esquema (tabla de hechos + 2 dimensiones)

Escribí y ejecutá esto (podés pegarlo en una celda con `cur.executescript(...)`, o correr cada `CREATE TABLE` por separado con `cur.execute(...)` para ir viendo que cada una se crea bien):

```python
cur.executescript("""
DROP TABLE IF EXISTS fact_emisiones;
DROP TABLE IF EXISTS dim_sector;
DROP TABLE IF EXISTS dim_gas;

CREATE TABLE dim_sector (
    sector_id   INTEGER PRIMARY KEY AUTOINCREMENT,
    sector      TEXT NOT NULL UNIQUE
);

CREATE TABLE dim_gas (
    gas_id      INTEGER PRIMARY KEY AUTOINCREMENT,
    tipo_de_gas TEXT NOT NULL UNIQUE,
    nombre_gas  TEXT NOT NULL
);

CREATE TABLE fact_emisiones (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    anio                INTEGER NOT NULL,
    sector_id           INTEGER NOT NULL REFERENCES dim_sector(sector_id),
    actividad           TEXT NOT NULL,
    subactividad        TEXT NOT NULL,
    categoria           TEXT NOT NULL,
    id_ipcc             TEXT NOT NULL,
    gas_id              INTEGER NOT NULL REFERENCES dim_gas(gas_id),
    valor_tco2e         REAL NOT NULL,
    tipo_de_registro    TEXT NOT NULL
);
""")
conn.commit()
```

**Por qué este diseño:** `fact_emisiones` es la tabla de hechos (un registro por observación, igual que tu CSV), y en vez de repetir el nombre del sector o del gas en cada fila, guardamos solo su `id` (clave foránea) apuntando a `dim_sector`/`dim_gas`. Esto es un esquema en estrella: una tabla de hechos grande rodeada de dimensiones chicas — el modelo que Power BI espera para armar relaciones y que las medidas DAX funcionen bien.

---

## Paso 4 — Poblar las dimensiones

```python
sectores = sorted(df["sector"].unique())
cur.executemany("INSERT INTO dim_sector (sector) VALUES (?)", [(s,) for s in sectores])

nombres_gas = {
    "CO2": "Dióxido de carbono",
    "CH4": "Metano",
    "N2O": "Óxido nitroso",
    "HFC": "Hidrofluorocarbonos",
    "PFC": "Perfluorocarbonos",
}
gases = sorted(df["tipo_de_gas"].unique())
cur.executemany(
    "INSERT INTO dim_gas (tipo_de_gas, nombre_gas) VALUES (?, ?)",
    [(g, nombres_gas.get(g, g)) for g in gases],
)
conn.commit()

pd.read_sql("SELECT * FROM dim_sector", conn)
```

Deberías ver 4 sectores. Corré también `pd.read_sql("SELECT * FROM dim_gas", conn)` y confirmá que son 5 gases.

---

## Paso 5 — Poblar la tabla de hechos

```python
sector_ids = pd.read_sql("SELECT sector_id, sector FROM dim_sector", conn)
gas_ids = pd.read_sql("SELECT gas_id, tipo_de_gas FROM dim_gas", conn)

df = df.merge(sector_ids, on="sector", how="left")
df = df.merge(gas_ids, on="tipo_de_gas", how="left")

assert df["sector_id"].isnull().sum() == 0
assert df["gas_id"].isnull().sum() == 0

columnas_fact = ["anio", "sector_id", "actividad", "subactividad",
                  "categoria", "id_ipcc", "gas_id", "valor_tco2e", "tipo_de_registro"]

df[columnas_fact].to_sql("fact_emisiones", conn, if_exists="append", index=False)
conn.commit()
```

**Verificación (hacela siempre después de cargar):**

```python
total_filas = cur.execute("SELECT COUNT(*) FROM fact_emisiones").fetchone()[0]
total_valor = cur.execute("SELECT SUM(valor_tco2e) FROM fact_emisiones").fetchone()[0]
print(total_filas, round(total_valor, 2))
```

Tiene que dar **7737** filas y **12398.33**. Si no coincide, no sigas — revisá el paso anterior.

---

## Paso 6 — Las 7 vistas (una por una, con qué está haciendo cada una)

Para cada vista: pegá el `CREATE VIEW` en una celda, ejecutalo con `cur.executescript(...)`, y en la celda siguiente consultala con `pd.read_sql("SELECT * FROM <nombre_vista>", conn)` para ver el resultado.

### 6.1 — Balance anual (emisiones, remociones, neto)

```sql
CREATE VIEW vw_balance_anual AS
SELECT
    anio,
    ROUND(SUM(CASE WHEN valor_tco2e > 0 THEN valor_tco2e ELSE 0 END), 2) AS emisiones,
    ROUND(SUM(CASE WHEN valor_tco2e < 0 THEN valor_tco2e ELSE 0 END), 2) AS remociones,
    ROUND(SUM(valor_tco2e), 2) AS balance_neto
FROM fact_emisiones
GROUP BY anio
ORDER BY anio;
```
`CASE WHEN` separa emisiones (positivo) de remociones (negativo) dentro del mismo `SUM`. Verificá: la suma de `balance_neto` de las 33 filas tiene que dar 12398.33 de nuevo.

### 6.2 — Totales y % de participación por sector

```sql
CREATE VIEW vw_totales_sector AS
SELECT
    s.sector,
    ROUND(SUM(f.valor_tco2e), 2) AS total_tco2e,
    ROUND(100.0 * SUM(f.valor_tco2e) / (SELECT SUM(valor_tco2e) FROM fact_emisiones), 1) AS participacion_pct
FROM fact_emisiones f
JOIN dim_sector s ON f.sector_id = s.sector_id
GROUP BY s.sector
ORDER BY total_tco2e DESC;
```
Acá aparece el primer `JOIN` con una dimensión, y una subconsulta escalar (el `SELECT` entre paréntesis) para calcular el % contra el total general. Debería darte Agricultura/Ganadería primero con 5899.92, Energía con 5422.41.

### 6.3 — Evolución por año y sector (formato largo, para el gráfico de líneas)

```sql
CREATE VIEW vw_evolucion_sector_anio AS
SELECT
    f.anio,
    s.sector,
    ROUND(SUM(f.valor_tco2e), 2) AS total_tco2e
FROM fact_emisiones f
JOIN dim_sector s ON f.sector_id = s.sector_id
GROUP BY f.anio, s.sector
ORDER BY f.anio, s.sector;
```
132 filas (33 años × 4 sectores).

### 6.4 — Variación interanual por sector (función de ventana: `LAG`)

```sql
CREATE VIEW vw_variacion_sector AS
WITH totales AS (
    SELECT f.anio, s.sector, SUM(f.valor_tco2e) AS total_tco2e
    FROM fact_emisiones f
    JOIN dim_sector s ON f.sector_id = s.sector_id
    GROUP BY f.anio, s.sector
)
SELECT
    anio,
    sector,
    ROUND(total_tco2e, 2) AS total_tco2e,
    ROUND(total_tco2e - LAG(total_tco2e) OVER (PARTITION BY sector ORDER BY anio), 2) AS variacion_absoluta,
    ROUND(
        100.0 * (total_tco2e - LAG(total_tco2e) OVER (PARTITION BY sector ORDER BY anio))
        / NULLIF(ABS(LAG(total_tco2e) OVER (PARTITION BY sector ORDER BY anio)), 0)
    , 1) AS variacion_pct
FROM totales
ORDER BY sector, anio;
```
`LAG(total_tco2e) OVER (PARTITION BY sector ORDER BY anio)` significa: "traeme el valor de la fila anterior, pero sin mezclar sectores distintos (`PARTITION BY sector`) y ordenando por año". Es la función de ventana que te permite comparar cada año contra el anterior sin un self-join. El primer año de cada sector va a dar `NULL` en variación (no hay año anterior) — es esperable. Para filtrar el último año: `pd.read_sql("SELECT * FROM vw_variacion_sector WHERE anio = 2022", conn)` — Energía debería dar +3.9%.

### 6.5 — Composición % por tipo de gas y año (función de ventana: `SUM() OVER`)

```sql
CREATE VIEW vw_composicion_gas_anio AS
WITH totales_gas AS (
    SELECT f.anio, g.tipo_de_gas, SUM(f.valor_tco2e) AS total_tco2e
    FROM fact_emisiones f
    JOIN dim_gas g ON f.gas_id = g.gas_id
    GROUP BY f.anio, g.tipo_de_gas
)
SELECT
    anio,
    tipo_de_gas,
    ROUND(total_tco2e, 2) AS total_tco2e,
    ROUND(100.0 * total_tco2e / SUM(total_tco2e) OVER (PARTITION BY anio), 1) AS participacion_pct
FROM totales_gas
ORDER BY anio, tipo_de_gas;
```
Acá `SUM(total_tco2e) OVER (PARTITION BY anio)` calcula el total del año SIN colapsar las filas (a diferencia de un `GROUP BY` normal) — por eso podés dividir cada gas por el total de su año en la misma fila. Para 2022, el CO2 debería dar 59.7%.

### 6.6 — Top 10 categorías 2018-2022 (función de ventana: `RANK`)

```sql
CREATE VIEW vw_top_categorias_recientes AS
WITH totales_categoria AS (
    SELECT categoria, SUM(valor_tco2e) AS total_tco2e
    FROM fact_emisiones
    WHERE anio >= 2018
    GROUP BY categoria
),
rankeadas AS (
    SELECT categoria, total_tco2e, RANK() OVER (ORDER BY total_tco2e DESC) AS ranking
    FROM totales_categoria
)
SELECT ranking, categoria, ROUND(total_tco2e, 2) AS total_tco2e
FROM rankeadas
WHERE ranking <= 10
ORDER BY ranking;
```
El primer puesto tiene que ser "Bovinos de Carne" con 361.49.

### 6.7 — Actividades dentro de Energía y Agricultura/Ganadería

```sql
CREATE VIEW vw_actividades_sector_principal AS
SELECT
    s.sector,
    f.actividad,
    ROUND(SUM(f.valor_tco2e), 2) AS total_tco2e
FROM fact_emisiones f
JOIN dim_sector s ON f.sector_id = s.sector_id
WHERE s.sector IN ('Energía', 'Agricultura, Ganadería, Silvicultura y Otros Usos de la Tierra')
GROUP BY s.sector, f.actividad
ORDER BY s.sector, total_tco2e DESC;
```
5 filas en total (3 actividades en Agricultura, 2 en Energía).

---

## Paso 7 — Exportar todo a CSV

```python
tablas_y_vistas = [
    "fact_emisiones", "dim_sector", "dim_gas",
    "vw_balance_anual", "vw_totales_sector", "vw_evolucion_sector_anio",
    "vw_variacion_sector", "vw_composicion_gas_anio",
    "vw_top_categorias_recientes", "vw_actividades_sector_principal",
]

for nombre in tablas_y_vistas:
    pd.read_sql(f"SELECT * FROM {nombre}", conn).to_csv(
        f"../data/powerbi/{nombre}.csv", index=False
    )
    print(nombre, "exportado")

conn.close()
```

Esto te va a pisar los CSV que ya están en `data/powerbi/` con los tuyos — está bien, es lo que queremos (que sea tu trabajo el que quede).

---

## Paso 8 — Importar a Power BI (a mano)

1. Abrí Power BI Desktop → **Obtener datos** → **Texto/CSV**.
2. Buscá la carpeta `data/powerbi/` y elegí `fact_emisiones.csv` → **Cargar**.
3. Repetí **Obtener datos → Texto/CSV** para cada uno de los otros 9 archivos, uno a la vez (`dim_sector`, `dim_gas`, y las 7 vistas `vw_...`). Van a aparecer como tablas separadas en el panel derecho de "Datos".
4. Por ahora no toques relaciones ni medidas — solo asegurate de que las 10 tablas quedaron importadas y que los tipos de dato se vean bien (los años como número entero, los `total_tco2e` como decimal).

Cuando termines de importar las 10, avisame y seguimos con el armado de relaciones en la vista de Modelo (fact ↔ dim_sector ↔ dim_gas) y recién después las medidas DAX — eso lo vemos en el próximo paso, con calma.
