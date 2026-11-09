# Producción y consumo de cerveza en Centroamérica y República Dominicana (2010–2023)

Proyecto de análisis de datos con **Python, SQL Server y Power BI** sobre datos públicos de la FAO (FAOSTAT). Incluye un modelo en estrella, un reporte de 4 páginas y proyecciones 2024–2028 validadas con backtesting.

**Autor:** Franklin Pastora · GitHub: [franklinPastora](https://github.com/franklinPastora)

> Datos: FAO (FAOSTAT), licencia CC BY 4.0. **La FAO no avala este trabajo.**

---

## Descripción

Los archivos masivos de FAOSTAT tienen más de 4 millones de filas cada uno. Este proyecto los filtra a lo necesario (cerveza y 7 países) y construye un flujo reproducible: Python descarga y limpia, SQL Server modela y Power BI muestra los resultados, con un enfoque especial en la posición de Nicaragua frente a la región.

## Objetivos

1. Comparar la producción de cerveza (hectolitros) de 7 países entre 2010 y 2023.
2. Comparar el consumo per cápita (kg/persona/año) y ubicar a Nicaragua frente al promedio regional.
3. Analizar importaciones y exportaciones de Nicaragua.
4. Proyectar producción y consumo 2024–2028 y **validar** el método con backtesting en lugar de elegirlo a ojo.

**Países:** Guatemala, El Salvador, Honduras, Nicaragua, Costa Rica, Panamá y República Dominicana.

## Dataset

| Fuente | Item FAOSTAT | Cobertura | Unidad | Filas usadas |
|---|---|---|---|---|
| Producción de cultivos y ganadería (QCL) | Beer of barley, malted | 2000–2023 | toneladas | 168 |
| Balances de alimentos (Food Balance Sheets) | Beer | 2010–2023 | miles de toneladas; kg/persona/año | 1,428 |

- QCL: `https://bulks-faostat.fao.org/production/Production_Crops_Livestock_E_All_Data_(Normalized).zip` (4,209,110 filas descargadas).
- FBS: `https://bulks-faostat.fao.org/production/FoodBalanceSheets_E_All_Data_(Normalized).zip` (4,820,497 filas descargadas).
- Consultado el 30 de octubre de 2026.
- Tabla de hechos final: 98 filas (7 países × 14 años).

## Stack

- **Python:** pandas, numpy, requests, matplotlib, seaborn, scikit-learn, openpyxl
- **SQL Server (T-SQL):** staging, dimensiones, tabla de hechos, vistas
- **Power BI Desktop:** modelo en estrella y medidas DAX

## Fases

| Fase | Qué se hace | Resultado |
|---|---|---|
| 1. Python | Descarga, filtra y exporta CSV; backtesting y proyecciones | `produccion_cerveza.csv`, `fbs_cerveza.csv`, `backtesting_cerveza.csv`, `proyecciones_cerveza.csv` |
| 2. SQL Server | Carga a staging, pivote de elementos, dimensiones, hechos y validaciones | `fact_cerveza`, `fact_proyeccion`, `dim_pais`, `dim_anio` |
| 3. Power BI | Modelo estrella, medidas DAX y 4 páginas | Reporte `CervezaFAO` |
| 4. Publicación | Documentación, capturas y cita de la FAO | Repositorio público |

Detalle de la carga en [`docs/carga_datos.md`](docs/carga_datos.md).

## Reporte en Power BI

1. **Resumen regional:** producción total, crecimiento anual y ranking de países.
2. **Consumo per cápita:** comparación por país, evolución y matriz con mapa de calor.
3. **Nicaragua vs región:** diferencia contra el promedio regional, balance comercial, importaciones y exportaciones.
4. **Proyecciones 2024–2028:** valores proyectados por país y método, con el resultado del backtesting.

Capturas en `docs/capturas/`.

## Proyecciones y backtesting

Para cada serie (7 países × producción y consumo) se probaron cuatro métodos: **naive** (repetir el último valor), **lineal**, **suavizado** (media móvil más tendencia) y **polinomio de grado 2**.

- **Prueba:** se entrena hasta un año de corte (2015 a 2020) y se proyectan los 3 años siguientes contra el dato real.
- **Métricas:** MAE y MAPE; se elige el método de menor MAE por serie y se compara entre países con el MAPE.
- **Proyección final:** el método ganador se reentrena con 2010–2023 y proyecta 2024–2028.

**Resultado:** ningún método gana siempre. De 14 series, gana naive en 6, lineal en 4, suavizado en 3 y polinomio de grado 2 en 1.

| País | Producción (método, MAPE) | Consumo per cápita (método, MAPE) |
|---|---|---|
| Costa Rica | naive, 22.5 % | naive, 27.8 % |
| República Dominicana | naive, 6.3 % | naive, 10.2 % |
| El Salvador | naive, 2.5 % | suavizado, 19.2 % |
| Guatemala | lineal, 13.2 % | lineal, 8.2 % |
| Honduras | lineal, 7.6 % | polinomio 2, 9.5 % |
| Nicaragua | lineal, 15.8 % | suavizado, 14.0 % |
| Panamá | suavizado, 17.3 % | naive, 11.3 % |

Las proyecciones son **orientativas**. En Nicaragua el error promedio a 3 años ronda entre 14 % y 16 %.

## Resultados clave

- **Producción 2023:** República Dominicana lidera con unos 5.38 millones de hl; Nicaragua es la menor de los 7 países, con 1.20 millones de hl.
- **Consumo per cápita 2023:** Panamá tiene el mayor (68.0 kg) y Nicaragua el menor (13.1 kg); el promedio simple de los 7 países es 33.4 kg.
- **Promedio 2010–2023:** Nicaragua 13.7 kg contra 30.5 kg del promedio simple (−16.8 kg).
- **Balance comercial de Nicaragua 2010–2023:** +62 mil toneladas acumuladas (exportaciones por encima de importaciones), con un aumento fuerte desde 2021 (de unas 3 mil t en 2018 a unas 36 mil t en 2022). No se interpreta sin revisar antes el `Flag` de la FAO.

## Limitaciones

- Muchos valores de la FAO son **estimados (E) o imputados (I)**; la columna `Flag` se conserva en SQL.
- Hay un **posible cambio de serie entre 2019 y 2020**: Costa Rica pasa de 11.4 a 37.0 kg per cápita y Nicaragua de 11.9 a 20.1 kg (y vuelve a 14.2 kg en 2021). Esto eleva los errores del backtesting.
- Hectolitros = toneladas × 10 (aproximación).
- El "promedio regional" es un promedio simple de los 7 países, no ponderado por población, e incluye a Nicaragua.
- Con 14 puntos por serie y ventanas de prueba que se solapan, el backtesting orienta, no demuestra.

## Cómo reproducirlo

1. `pip install -r requirements.txt`
2. Ejecuta `notebooks/cerveza_faostat.ipynb`: descarga los datos y genera los CSV en `data/`.
3. En SQL Server, ejecuta los scripts de `sql/` en orden (01 a 06). Ajusta las rutas de los CSV.
4. Abre `powerbi/CervezaFAO.pbix`, cambia el origen al nombre de tu servidor y actualiza.

## Estructura de carpetas

```
faostat-cerveza-centroamerica/
├── README.md
├── requirements.txt
├── data/
│   ├── produccion_cerveza.csv
│   ├── fbs_cerveza.csv
│   ├── backtesting_cerveza.csv
│   └── proyecciones_cerveza.csv
├── notebooks/
│   └── cerveza_faostat.ipynb
├── sql/
│   ├── 01_crear_bd_y_staging.sql
│   ├── 02_carga.sql
│   ├── 03_dimensiones.sql
│   ├── 04_fact_cerveza.sql
│   ├── 05_validaciones.sql
│   └── 06_vistas.sql
├── powerbi/
│   └── CervezaFAO.pbix
└── docs/
    ├── medidas_documentadas.md
    ├── carga_datos.md
    └── capturas/
```

No se sube el ZIP original de la FAO.

## Medidas DAX

Documentadas en [`docs/medidas_documentadas.md`](docs/medidas_documentadas.md): `Produccion hl`, `Produccion anio anterior`, `Crecimiento %`, `Consumo per capita kg`, `Ranking produccion`, `Promedio regional`, `Dif vs promedio regional`, `Dif Nicaragua vs promedio`, `Importaciones 1000t`, `Exportaciones 1000t`, `Balance comercial 1000t`, `Valor proyectado` y `Consumo proyectado kg`.

## Checklist pre-publicación

- [ ] Cifras del README iguales a las del dashboard final.
- [ ] Capturas de las 4 páginas en `docs/capturas/`.
- [ ] Sin el ZIP original ni rutas personales (C:/Users/...).
- [ ] Cita de la FAO y aviso de no aval en README, cuaderno y dashboard.
- [ ] `requirements.txt` con versiones fijadas (`pip freeze`).
- [ ] `backtesting_cerveza.csv` incluido en `data/`.
- [ ] Licencia del código elegida (por ejemplo MIT).

## Fuente y licencia

FAO. [año de última actualización según la ficha del dataset]. FAOSTAT: Crops and livestock products; Food Balance Sheets. Consultado el 30 de octubre de 2026. https://www.fao.org/faostat. Licence: CC-BY-4.0.

Código: [licencia elegida]. La FAO no avala este trabajo.
