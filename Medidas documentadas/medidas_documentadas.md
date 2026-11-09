# Medidas DAX — Proyecto Cerveza

Modelo en estrella: `dim_pais`, `dim_anio`, `fact_cerveza` y `fact_proyeccion`. Relaciones 1 a muchos por `PaisID` y `Anio`.

Páginas del reporte: **Resumen Regional**, **Consumo per cápita**, **Nicaragua vs región** y **Proyecciones**.

Aviso general: cuando no hay año seleccionado, las sumas cubren 2010–2023 completo y los promedios se calculan sobre todos esos años. Indícalo en los títulos o usa un segmentador de año.

---

## Producción

### Produccion hl
```
Produccion hl = SUM(fact_cerveza[Produccion_hl])
```
Suma la producción en hectolitros (toneladas × 10). **Páginas:** Resumen Regional, Nicaragua vs región (base del ranking), Proyecciones (línea histórica).

### Produccion anio anterior
```
Produccion anio anterior =
VAR AnioActual = MAX(fact_cerveza[Anio])
RETURN
    CALCULATE([Produccion hl], dim_anio[Anio] = AnioActual - 1)
```
Producción del año anterior al último año en contexto. **Página:** Resumen Regional (apoyo de otras medidas).

### Crecimiento %
```
Crecimiento % =
VAR AnioActual = MAX(fact_cerveza[Anio])
VAR Actual = CALCULATE([Produccion hl], dim_anio[Anio] = AnioActual)
VAR Anterior = CALCULATE([Produccion hl], dim_anio[Anio] = AnioActual - 1)
RETURN
    DIVIDE(Actual - Anterior, Anterior)
```
Variación porcentual frente al año anterior. **Página:** Resumen Regional.

> Nota: una versión previa dividía la suma de todos los años entre un solo año y daba 1109.63 %. Esta versión compara el mismo año contra el anterior. Formato: porcentaje.

### Ranking produccion
```
Ranking produccion = RANKX(ALL(dim_pais[Pais]), [Produccion hl])
```
Posición del país según su producción (1 = mayor). Sin año seleccionado, usa el acumulado 2010–2023. **Página:** Nicaragua vs región.

---

## Consumo

### Consumo per capita kg
```
Consumo per capita kg = AVERAGE(fact_cerveza[Consumo_percapita_kg])
```
Consumo de cerveza en kg por persona al año. Con varios países, es un promedio simple (no ponderado por población). **Páginas:** Resumen Regional, Consumo per cápita, Nicaragua vs región.

### Promedio regional
```
Promedio regional =
CALCULATE([Consumo per capita kg], ALL(dim_pais))
```
Promedio simple de los 7 países, ignorando el filtro de país; incluye a Nicaragua. **Página:** Nicaragua vs región (línea de comparación).

### Dif vs promedio regional
```
Dif vs promedio regional =
VAR Promedio = CALCULATE(AVERAGE(fact_cerveza[Consumo_percapita_kg]), ALL(dim_pais))
RETURN
    IF(HASONEVALUE(dim_pais[Pais]), [Consumo per capita kg] - Promedio)
```
Diferencia en kg entre un país y el promedio regional; solo da resultado con **un único país** seleccionado, si no, aparece en blanco. **Página:** Consumo per cápita.

### Dif Nicaragua vs promedio
```
Dif Nicaragua vs promedio =
CALCULATE([Consumo per capita kg], dim_pais[Pais] = "Nicaragua") - [Promedio regional]
```
Diferencia en kg de Nicaragua frente al promedio regional, sin depender de un segmentador. **Página:** Nicaragua vs región.

> Formato: número decimal (kg), **no** porcentaje. Con todos los años da −16.8 kg (13.7 contra 30.5).

---

## Comercio exterior

### Importaciones 1000t
```
Importaciones 1000t = SUM(fact_cerveza[Importaciones_1000t])
```
Importaciones de cerveza en miles de toneladas. **Página:** Nicaragua vs región (gráfico de columnas).

### Exportaciones 1000t
```
Exportaciones 1000t = SUM(fact_cerveza[Exportaciones_1000t])
```
Exportaciones de cerveza en miles de toneladas. **Página:** Nicaragua vs región (gráfico de columnas).

### Balance comercial 1000t
```
Balance comercial 1000t =
SUM(fact_cerveza[Exportaciones_1000t]) - SUM(fact_cerveza[Importaciones_1000t])
```
Exportaciones menos importaciones (positivo = exporta más de lo que importa). Sin año seleccionado es el acumulado 2010–2023. **Página:** Nicaragua vs región.

---

## Proyecciones

### Valor proyectado
```
Valor proyectado =
CALCULATE(
    SUM(fact_proyeccion[Valor]),
    fact_proyeccion[Variable] = "Produccion_hl"
)
```
Producción proyectada en hl. El texto entre comillas debe coincidir **exactamente** con lo guardado en `fact_proyeccion[Variable]` (se comprueba con `SELECT DISTINCT Variable FROM fact_proyeccion;`). Es una **estimación**, no un dato de la FAO. **Página:** Proyecciones.

### Consumo proyectado kg
```
Consumo proyectado kg =
CALCULATE(
    AVERAGE(fact_proyeccion[Valor]),
    fact_proyeccion[Variable] = "Consumo_per_capita_kg"
)
```
Consumo per cápita proyectado (kg). Usa promedio y no suma, porque sumar valores per cápita no tiene sentido. **Página:** Proyecciones.

> Pendiente: filtrar por método (`Metodo`) para no mezclar naive y lineal en la misma cifra.
