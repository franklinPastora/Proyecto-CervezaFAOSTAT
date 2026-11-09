# Carga de datos — Proyecto Cerveza

Requisitos: Python 3 con las librerías de `requirements.txt`, SQL Server 2017 o superior y SSMS. Las rutas (`C:/Datos/...`) son un ejemplo: cámbialas por las tuyas.

## Orden de ejecución

1. **Python:** descargar, filtrar y exportar los 3 CSV (`produccion_cerveza.csv`, `fbs_cerveza.csv`, `proyecciones_cerveza.csv`).
2. **SQL 2.1:** crear la base y las tablas de staging.
3. **SQL 2.2:** cargar los CSV a staging.
4. **SQL 2.3:** crear las dimensiones.
5. **SQL 2.4:** crear las tablas de hechos.
6. **SQL 2.5:** ejecutar las validaciones.
7. **SQL 2.6:** crear la vista de crecimiento.
8. **Power BI:** conectar a SQL Server y actualizar.

## Paso 0 — Python

- Filtrar solo `Beer of barley, malted` (QCL) y `Beer` (FBS), para los 7 países.
- Los nombres de elemento deben respetar las minúsculas: `Import quantity` y `Export quantity`.
- Exportar sin índice y conservar la columna `Flag`.
- El CSV de proyecciones debe tener las columnas `Area, Anio, Variable, Valor, Metodo, Es_estimacion` y los nombres de país **en inglés como los usa la FAO** (por ejemplo `Panama`, `Dominican Republic`).
- No subir a GitHub el ZIP original (55 MB).

## 2.1 Base y staging

```sql
CREATE DATABASE CervezaFAO;
GO
USE CervezaFAO;

CREATE TABLE stg_produccion (Area NVARCHAR(60), Item NVARCHAR(60), Element NVARCHAR(60), Year INT, Unit NVARCHAR(20), Value FLOAT, Flag NVARCHAR(5));
CREATE TABLE stg_fbs (Area NVARCHAR(60), Item NVARCHAR(60), Element NVARCHAR(80), Year INT, Unit NVARCHAR(20), Value FLOAT, Flag NVARCHAR(5));
CREATE TABLE stg_proyeccion (Area NVARCHAR(60), Anio INT, Variable NVARCHAR(40), Valor FLOAT, Metodo NVARCHAR(40), Es_estimacion BIT);
```

## 2.2 Carga

```sql
BULK INSERT stg_produccion FROM 'C:/Datos/produccion_cerveza.csv' WITH (FORMAT = 'CSV', FIRSTROW = 2, CODEPAGE = '65001');
BULK INSERT stg_fbs FROM 'C:/Datos/fbs_cerveza.csv' WITH (FORMAT = 'CSV', FIRSTROW = 2, CODEPAGE = '65001');
BULK INSERT stg_proyeccion FROM 'C:/Datos/proyecciones_cerveza.csv' WITH (FORMAT = 'CSV', FIRSTROW = 2, CODEPAGE = '65001');
```

Si `BULK INSERT` falla por permisos de la ruta, usa el asistente **Importar archivo plano** de SSMS. El orden de columnas del CSV debe coincidir con el de la tabla.

## 2.3 Dimensiones

```sql
CREATE TABLE dim_pais (PaisID INT IDENTITY PRIMARY KEY, Area_FAO NVARCHAR(60), Pais NVARCHAR(60));
INSERT INTO dim_pais (Area_FAO, Pais) VALUES
('Guatemala','Guatemala'),('El Salvador','El Salvador'),('Honduras','Honduras'),
('Nicaragua','Nicaragua'),('Costa Rica','Costa Rica'),('Panama','Panamá'),
('Dominican Republic','República Dominicana');

CREATE TABLE dim_anio (Anio INT PRIMARY KEY);
WITH n AS (SELECT 2010 AS Anio UNION ALL SELECT Anio + 1 FROM n WHERE Anio < 2028)
INSERT INTO dim_anio SELECT Anio FROM n;
```

## 2.4 Tablas de hechos

```sql
WITH fbs AS (
  SELECT Area, Year,
    SUM(CASE WHEN Element = 'Production' THEN Value END) AS Produccion_fbs_1000t,
    SUM(CASE WHEN Element = 'Import quantity' THEN Value END) AS Importaciones_1000t,
    SUM(CASE WHEN Element = 'Export quantity' THEN Value END) AS Exportaciones_1000t,
    SUM(CASE WHEN Element = 'Domestic supply quantity' THEN Value END) AS Oferta_interna_1000t,
    SUM(CASE WHEN Element = 'Food' THEN Value END) AS Consumo_1000t,
    SUM(CASE WHEN Element = 'Food supply quantity (kg/capita/yr)' THEN Value END) AS Consumo_percapita_kg
  FROM stg_fbs GROUP BY Area, Year)
SELECT p.PaisID, f.Year AS Anio,
  pr.Value AS Produccion_ton,
  pr.Value * 10 AS Produccion_hl,  -- aproximación: 1 t ≈ 10 hl
  pr.Flag AS Flag_produccion,
  f.Produccion_fbs_1000t, f.Importaciones_1000t, f.Exportaciones_1000t,
  f.Oferta_interna_1000t, f.Consumo_1000t, f.Consumo_percapita_kg
INTO fact_cerveza
FROM fbs f
JOIN dim_pais p ON p.Area_FAO = f.Area
LEFT JOIN stg_produccion pr ON pr.Area = f.Area AND pr.Year = f.Year AND pr.Element = 'Production';

SELECT p.PaisID, s.Anio, s.Variable, s.Valor, s.Metodo, s.Es_estimacion
INTO fact_proyeccion
FROM stg_proyeccion s JOIN dim_pais p ON p.Area_FAO = s.Area;
```

## 2.5 Validaciones

```sql
SELECT COUNT(*) FROM fact_cerveza;                          -- esperado: 98 (7 países x 14 años)
SELECT PaisID, COUNT(*) FROM fact_cerveza GROUP BY PaisID;   -- 14 por país
SELECT * FROM fact_cerveza
WHERE Importaciones_1000t IS NULL OR Consumo_percapita_kg IS NULL;   -- revisar huecos
SELECT DISTINCT Element FROM stg_fbs WHERE Element LIKE '%capita%';  -- confirmar nombres de elemento
SELECT DISTINCT Variable FROM fact_proyeccion;               -- textos que usan las medidas DAX
```

## 2.6 Vista de crecimiento anual

```sql
CREATE VIEW vw_crecimiento_produccion AS
SELECT PaisID, Anio, Produccion_hl,
  (Produccion_hl - LAG(Produccion_hl) OVER (PARTITION BY PaisID ORDER BY Anio)) * 100.0
  / NULLIF(LAG(Produccion_hl) OVER (PARTITION BY PaisID ORDER BY Anio), 0) AS Crecimiento_pct
FROM fact_cerveza;
```

## Power BI

Obtener datos → SQL Server → modo Importar: `dim_pais`, `dim_anio`, `fact_cerveza`, `fact_proyeccion` y, opcionalmente, `vw_crecimiento_produccion`. Relaciones 1 a muchos: `dim_pais[PaisID]` y `dim_anio[Anio]` hacia ambas tablas de hechos.
