# Desarrollo de un Dashboard de Ventas para Supervisión de Locales Retail

## Índice

1. [Introducción](#introducción)
2. [Fuente de Datos](#fuente-de-datos)
3. [Plan de Métricas](#plan-de-métricas)
4. [Hipótesis](#hipotesis)
5. [Modelo de Datos](#modelo-de-datos)
6. [ETL](#ETL)
7. [Dashboard](#dashboard)
8. [Conclusión](#conclusión)

## Introducción

En el desarrollo de este proyecto, la elección de la fuente de datos fue fundamental. Opté por utilizar información interna del negocio real, ya que contenía datos altamente valiosos que, hasta el momento, no se estaban aprovechando. Esta información sin uso es crucial porque está directamente relacionada con las ventas, un aspecto central para la operación de cualquier comercio. El análisis va desde las ventas totales hasta la evaluación del desempeño individual de cada vendedora en el local.

La fuente de datos utilizada en este análisis proviene de los puntos de venta de tres locales. Estos puntos de venta recopilan información detallada de todas las transacciones realizadas en cada local. Las transacciones contienen datos clave como los montos de las ventas, los medios de pago utilizados, las vendedoras responsables de las ventas, los clientes involucrados, las fechas y horarios en que ocurrieron.

## Fuente de Datos

En un inicio dos tablas como fuentes de datos, basadas en los puntos de venta de cada local.

**Análisis de Venta**

![](https://github.com/RodriBustamante/DataAnalysis_Proyecto/blob/main/imagenes/ETL/Bronze_Layer2.png)

Esta fuente de datos, tiene todas las transacciones de los puntos de venta de cada local, con datos asociados fundamentales para el análisis de ventas y financiero.

## Plan de Métricas

El plan de métricas fue enfocado en obtener la mejor información para que la supervisión pueda corroborar la productividad en ventas de cada equipo.

![](https://github.com/RodriBustamante/DataAnalysis_Proyecto/blob/main/imagenes/Plan_Métricas.png)

[Link al Plan](https://docs.google.com/spreadsheets/d/1gcPnxIB98Yn8IsfQwROQ-1OX1a7pq0O_wbsURNq8oJc/edit?usp=sharing)

## Hipótesis

1. Más del 60% de las ventas son realizadas a clientes fidelizados
2. Los equipos muestran una tendencia positiva en los KPI fundamentales (u/tkt y tkt promedio)
3. Los horarios en los que más se vende es entre las 20:00hs y las 22:00hs
4. El medio de pago más utilizado es tarjeta

## Modelo de Datos

En un inicio el modelo de datos era el siguiente:

![](https://github.com/RodriBustamante/DataAnalysis_Proyecto/blob/main/imagenes/Datamart.png)

A medida que fui progresando en el desarrollo y probando como daban los datos, concluí que los datos relevados en las tablas por DIM_ARTICULOS, generaban ciertos errores, por lo que decídi crear una tabla de hechos relacionada directamente con los artículos. Quedando el modelo de datos de la sigueinte forma:

![](https://github.com/RodriBustamante/DataAnalysis_Proyecto/blob/main/imagenes/Modelo_Datos2.png)

## ETL

**Data Flow**

![](https://github.com/RodriBustamante/DataAnalysis_Proyecto/blob/main/imagenes/DataFlow2.png)

### Bronze Layer

Las tablas iniciales son tomadas del sistema de Punto de Venta de los locales del Norte, Centro y Sur. Las mismas contienen datos en una venta de tiempo de entre un año para el caso de AVM y de más de 10 años para el caso de Stock. Las columnas que pertenecen a estas tablas son:

![](https://github.com/RodriBustamante/DataAnalysis_Proyecto/blob/main/imagenes/ETL/Bronze_Layer.png)

En total tengo dos informes AVM separados, por la limitación de descarga en el software de la empresa. El primer informe contiene las transacciones del Sur y Centro, mientras que el último solo las del Norte. Además, de otra tabla, voy a sacar los datos de Stock, no porque necesita algo de stock particulamente sino porque de allí puedo sacar la linea de los productos, dato interesante para hacer análisis.

Entonces debo unir tanto AVM_Norte con AVM_CentroSur cómo Stock_Norte con Stock_CentroSur, el objetivo es lograr una tabla base que sirva como capa de bronce. A partir de la misma, se realizará la capa de plata con menor granularidad y más enfocado al plan de métricas.

**Stock: me permite tener la lista de artículos separados por línea**

```sql
CREATE OR REPLACE TABLE `trabajofinaledvai.Capa_Bronce.Stock` AS


WITH TablaAux AS (
  SELECT `Código Artículo`,`Descripción`,`Linea`
  FROM `trabajofinaledvai.Capa_Bronce.Stock_Norte` AS StockNorte
  UNION ALL (SELECT `Código Artículo`,`Descripción`,`Linea`  FROM `trabajofinaledvai.Capa_Bronce.Stock_CentroSur`)
)
SELECT DISTINCT * from TablaAux
```

**Análisis de Venta Minorista: me permite tener todas las transacciones del punto de venta del local**

```sql
CREATE OR REPLACE TABLE `trabajofinaledvai.Capa_Bronce.AVM` AS


WITH TablaAux AS (
  SELECT Local, Fecha, Hora, Abrev, Numero AS Factura, `Cod Cliente`, Cliente,`Codigo Articulo`, Descripcion, Cantidad, Vendedor, `Precio neto`, `Precio neto final`, Importe, `Medio de Pago`
  FROM `trabajofinaledvai.Capa_Bronce.AVM_Norte` AS StockNorte
  UNION ALL (SELECT Local, Fecha, Hora, Abrev, Numero AS Factura, `Cod Cliente`, Cliente,`Codigo Articulo`, Descripcion, Cantidad, Vendedor, `Precio neto`, `Precio neto final`, Importe, `Medio de Pago`  FROM `trabajofinaledvai.Capa_Bronce.AVM_CentroSur`)
)


SELECT *  FROM TablaAux ORDER BY Fecha DESC
```

De esta forma concluiría el desarrollo de la capa bronce, ya tengo la base para armar lo que sigue.

### Silver Layer

A partir de ahora haré un JOIN que me permite unir tanto la tabla AVM con la de Stock. Esto me permitirá interactuar con una sola tabla con todos los datos crudos y ya filtrados previamente según las necesidades del plan de métricas.

```sql
SELECT *
FROM `trabajofinaledvai.Capa_Bronce.AVM` AS AVM
LEFT JOIN `trabajofinaledvai.Capa_Bronce.Stock` AS Stock
ON `Codigo Articulo` = `Código Artículo`
```

Al hacer la consulta, utilicé un LEFT JOIN, para verificar si había artículos que no estén cargados en el sistema, de manera que los encontraría como valores nulos. Lo que me llevo a encontrar que habían ciertos valores que aparecían como nulos. Pude determinar que los valores nulos no correspondían a un artículo sino que estaban relacionados con la operatoria del sistema transaccional, el cual para colocar descuentos carga en el sistema un artículo relacionado al descuento. 

**QUERY para verificar:**

```sql
WITH Tabla_Aux AS (
  SELECT *
  FROM `trabajofinaledvai.Capa_Bronce.AVM` AS AVM
  LEFT JOIN `trabajofinaledvai.Capa_Bronce.Stock` AS Stock
  ON `Codigo Articulo` = `Código Artículo`
)
SELECT DISTINCT `Codigo Articulo`, `Código Artículo`, Descripcion, `Precio Neto`
FROM Tabla_Aux
WHERE `Código Artículo` IS NULL
```

Este dato adicionado altera completamente las métricas, por lo que se decide, hacer un JOIN, para obviar los descuentos y tomar los artículos realmente comprados.

**QUERY base para comenzar con la capa silver:**

```sql
CREATE OR REPLACE TABLE `trabajofinaledvai.Capa_Plata.AVM_Base` AS
SELECT *
FROM `trabajofinaledvai.Capa_Bronce.AVM` AS AVM
JOIN `trabajofinaledvai.Capa_Bronce.Stock` AS Stock
ON `Codigo Articulo` = `Código Artículo`
```

**QUERY para estandarizar los datos y descartar columnas innecesarias:**

```sql
CREATE OR REPLACE TABLE `trabajofinaledvai.Capa_Plata.AVM_SilverLayer` AS
SELECT
FACTURA,
FECHA,
HORA,
LOCAL AS NOMBRE_LOCAL,
`Cod Cliente` AS CLIENTE_ID,
CLIENTE AS NOMBRE_CLIENTE,
Vendedor AS NOMBRE_VENDEDOR,
`Código Artículo` AS ARTICULO_ID,
`Descripción` AS NOMBRE_ARTICULO,
Linea AS LINEA_ARTICULO,
`Medio de Pago` AS METODO_PAGO,
Cantidad AS CANTIDAD_FACTURA,
Importe AS IMPORTE_FACTURA
FROM `trabajofinaledvai.Capa_Plata.AVM_Base`
ORDER BY FECHA DESC, HORA DESC
```

Concluye el desarrollo de las tablas en la capa de plata.

### Gold Layer

**Big Query**

![](https://github.com/RodriBustamante/DataAnalysis_Proyecto/blob/main/imagenes/BigQuery.png)

La última etapa se realizó casi en su totalidad den bigquery, salvo por necesidades se modificaron las tablas para aprovechar más los datos. Para armar la capa de oro, voy a seguir la estructura del modelo de datos, con las tablas de hechos y las tablas de dimensiones correspondientes.

**QUERY para agregar lo ID para crear las tablas de dimensiones:**

```sql
CREATE OR REPLACE TABLE `trabajofinaledvai.Capa_Oro.AVM_GoldLayer` AS
SELECT
FACTURA,
DENSE_RANK() OVER (ORDER BY FECHA, HORA) AS FECHA_ID,
FECHA,
HORA,
NOMBRE_LOCAL,
DENSE_RANK() OVER (ORDER BY NOMBRE_LOCAL) AS LOCAL_ID,
DENSE_RANK() OVER (ORDER BY NOMBRE_CLIENTE) AS CLIENTE_ID,
NOMBRE_CLIENTE,
DENSE_RANK() OVER (ORDER BY NOMBRE_VENDEDOR) AS VENDEDOR_ID,
NOMBRE_VENDEDOR,
DENSE_RANK() OVER (ORDER BY ARTICULO_ID, LINEA_ARTICULO) AS ARTICULO_ID,
NOMBRE_ARTICULO,
LINEA_ARTICULO,
DENSE_RANK() OVER (ORDER BY METODO_PAGO) AS MEDIO_ID,
METODO_PAGO,
CANTIDAD_FACTURA,
IMPORTE_FACTURA
FROM `trabajofinaledvai.Capa_Plata.AVM_SilverLayer`
ORDER BY FECHA DESC, HORA DESC
```

**QUERY para crear la tabla de dimensiones de locales:**

```sql
CREATE OR REPLACE TABLE `trabajofinaledvai.Capa_Oro.DIM_LOCALES` AS
SELECT DISTINCT
LOCAL_ID,
NOMBRE_LOCAL,
FROM `trabajofinaledvai.Capa_Oro.AVM_GoldLayer`
ORDER BY LOCAL_ID ASC
```

**QUERY para crear la tabla de dimensiones de clientes:**

```sql
CREATE OR REPLACE TABLE `trabajofinaledvai.Capa_Oro.DIM_CLIENTES` AS
SELECT DISTINCT
  CLIENTE_ID,
  NOMBRE_CLIENTE,
FROM `trabajofinaledvai.Capa_Oro.AVM_GoldLayer`
ORDER BY CLIENTE_ID
```

**QUERY para crear la tabla de dimensiones de artículos:**

```sql
CREATE OR REPLACE TABLE `trabajofinaledvai.Capa_Oro.DIM_ARTICULOS` AS
SELECT DISTINCT
  ARTICULO_ID,
  NOMBRE_ARTICULO,
  LINEA_ARTICULO,
FROM `trabajofinaledvai.Capa_Oro.AVM_GoldLayer`
ORDER BY ARTICULO_ID ASC
```
**QUERY para crear la tabla de dimensiones de vendedor:**

```sql
CREATE OR REPLACE TABLE `trabajofinaledvai.Capa_Oro.DIM_VENDEDOR` AS
SELECT DISTINCT
  VENDEDOR_ID,
  NOMBRE_VENDEDOR,
FROM `trabajofinaledvai.Capa_Oro.AVM_GoldLayer`
ORDER BY VENDEDOR_ID ASC
```

**QUERY para crear la tabla de dimensiones de tiempo:**

```sql
CREATE OR REPLACE TABLE `trabajofinaledvai.Capa_Oro.DIM_TIEMPO` AS
SELECT DISTINCT
    FECHA_ID,
    FECHA,
    HORA,
    FORMAT_DATE('%B', FECHA) AS MES,
    CASE
    WHEN EXTRACT(QUARTER FROM FECHA) = 1 THEN '1er trimestre'
    WHEN EXTRACT(QUARTER FROM FECHA) = 2 THEN '2do trimestre'
    WHEN EXTRACT(QUARTER FROM FECHA) = 3 THEN '3er trimestre'
    WHEN EXTRACT(QUARTER FROM FECHA) = 4 THEN '4to trimestre'
  END AS QUARTER
FROM `trabajofinaledvai.Capa_Oro.AVM_GoldLayer`
ORDER BY FECHA DESC, HORA DESC
```

**QUERY para crear la tabla de hechos de las transacciones de facturación:**

 ```sql
CREATE OR REPLACE TABLE `trabajofinaledvai.Capa_Oro.HECHOS_FACTURACION` AS
SELECT
FACTURA,
FECHA_ID,
LOCAL_ID,
CLIENTE_ID,
VENDEDOR_ID,
ARRAY_AGG(STRUCT(ARTICULO_ID)) AS ARTICULO_ID,
MEDIO_ID,
SUM(CANTIDAD_FACTURA) AS CANTIDAD,
ANY_VALUE(IMPORTE_FACTURA) AS IMPORTE_FACTURA
FROM `trabajofinaledvai.Capa_Oro.AVM_GoldLayer`
GROUP BY FACTURA, FECHA_ID, LOCAL_ID,CLIENTE_ID,VENDEDOR_ID,MEDIO_ID
```

**MODIFICACIÓN DE LA QUERY ANTERIOR, para crear la tabla de hechos de las transacciones de facturación:**

```sql
CREATE OR REPLACE TABLE `trabajofinaledvai.Capa_Oro.HECHOS_FACTURACION` AS
SELECT
FACTURA,
FECHA_ID,
LOCAL_ID,
CLIENTE_ID,
VENDEDOR_ID,
ANY_VALUE(ARTICULO_ID) AS ARTICULO_ID,
ANY_VALUE(MEDIO_ID) AS MEDIO_ID,
SUM(CANTIDAD_FACTURA) AS CANTIDAD_FACTURA,
MAX(IMPORTE_FACTURA) AS IMPORTE_FACTURA
FROM `trabajofinaledvai.Capa_Oro.AVM_GoldLayer`
GROUP BY FACTURA, FECHA, FECHA_ID, LOCAL_ID, VENDEDOR_ID, CLIENTE_ID
ORDER BY FECHA DESC
```

El cambio se debe a que la tabla original tiene un error, en el que tiene varios medios para una misma factura, lo que no corresponde, ya que en situaciones así debería aparecer “varios”. El cambio favorece ya que podemos discriminar correctamente cada factura según su medio de pago y evitar tener montos duplicados.

**QUERY para crear la tabla de hechos de las transacciones de artículos:**

```sql
CREATE OR REPLACE TABLE `trabajofinaledvai.Capa_Oro.HECHOS_ARTICULOS` AS
SELECT
FACTURA,
ARTICULO_ID,
CANTIDAD_FACTURA,
FROM `trabajofinaledvai.Capa_Oro.AVM_GoldLayer`
ORDER BY FECHA DESC
```

Surge de la necesidad de separa las facturas de cada artículo que se vendio en cada una. Distorsionaba los datos y no permitía calcular correctamente los montos por local, ni las unidades de cada artículo que se vendían.

**PowerQuery**

![](https://github.com/RodriBustamante/DataAnalysis_Proyecto/blob/main/imagenes/PowerQuery.png)

Al terminar la etapa en BigQuery, necesite agregar ciertas columnas a algunas dimensiones para poder segmentar mejor los datos y darles una mejor visualización.

**Agregar una segmentación de medios de pago para poder entender más a nivel macro**

Lo hice de manera manual, ya que era muy difícil distinguir cada uno

![](https://github.com/RodriBustamante/DataAnalysis_Proyecto/blob/main/imagenes/ETL/Gold_LayerPQ4.png)

**Agregar datos a la tabla calendario, como la segmentación de horas, para obtener datos más fáciles de leer de la actividad**

![](https://github.com/RodriBustamante/DataAnalysis_Proyecto/blob/main/imagenes/ETL/Gold_LayerPQ5.png)

La query que utilice fue la siguiente:

![](https://github.com/RodriBustamante/DataAnalysis_Proyecto/blob/main/imagenes/ETL/Gold_LayerPQ6.png)

## Desarrollo del Dashboard de Ventas para la Supervisión

A partir de aquí comence con las medidas DAX, para poder verificar las hipotesís y siguiendo el plan de métricas.

*Suma de los importes de cada transacción realizada en los puntos de venta*

![](https://github.com/RodriBustamante/DataAnalysis_Proyecto/blob/main/imagenes/DAX/SumImporte.png)

*Cálculo de las unidades vendidas por cada factura*

![](https://github.com/RodriBustamante/DataAnalysis_Proyecto/blob/main/imagenes/DAX/Productividad.png)

*Cálculo del ticket promedio, que es el importe total sobre las facturas emitidas*

![](https://github.com/RodriBustamante/DataAnalysis_Proyecto/blob/main/imagenes/DAX/TKTpromedio.png)

*Suma de la cantidad de articulos vendidos por factura*

![](https://github.com/RodriBustamante/DataAnalysis_Proyecto/blob/main/imagenes/DAX/ContarArtículos.png)


### Dashboard

**INICIO**

Esta página está diseñada como la primera interacción del usuario con el dashboard. Muestra de manera clara las ventas distribuidas por mes, permitiendo a los supervisores obtener una visión rápida y precisa de cómo han fluctuado las ventas en un periodo de tiempo específico. Esto facilita la identificación de patrones o tendencias estacionales y brinda una perspectiva inicial del desempeño del negocio.

**GENERAL**
En esta hoja, se presentan los indicadores clave de rendimiento (KPIs) más importantes del negocio. Estos KPIs están diseñados para ofrecer un resumen ejecutivo que permite a la supervisión evaluar de forma rápida las áreas que necesitan atención.

**DETALLE**
Esta sección profundiza en las ventas a nivel individual, permitiendo un análisis detallado y personalizado del desempeño de cada vendedor o equipo. Se puede visualizar cómo ha evolucionado el rendimiento de cada vendedor a lo largo del tiempo, lo que facilita la identificación de patrones de éxito, áreas de mejora, y diferencias en las estrategias de venta entre ellos. Esta página es fundamental para la gestión del equipo de ventas, ya que ofrece una vista más granular que ayuda a tomar decisiones informadas en términos de reconocimiento o capacitación adicional.

**MEDIOS DE PAGO**
Esta página está diseñada para mostrar los montos detallados según los diferentes medios de pago utilizados a lo largo del año. Los supervisores pueden analizar la distribución de los pagos por tarjeta, efectivo, transferencias, entre otros. La página también permite aplicar filtros dinámicos para ver los datos en un día específico, un mes determinado, o incluso desglosados por local. 

**CLIENTES**
En esta hoja se presenta un análisis enfocado en los clientes, mostrando los montos de compras por cliente con mayores montos. Esto facilita el seguimiento de los clientes frecuentes y permite identificar patrones de fidelización. Además, ofrece insights sobre aquellos clientes que podrían necesitar un impulso adicional para volver a comprar.

[Link al Dashboard](https://rodribustamante.github.io/DataAnalysis_Proyecto/Reporte.html)

## Conclusión

1. Más del 20% de las ventas son realizadas a clientes fidelizados
- Se pudo verificar que ningúno de los locales cumplia con este KPI fundamental, las estrategias para fidelizar se deben poner en marcha, ya que de no ser así pueden perderse clientes potenciales.
  
2. Los equipos muestran una tendencia positiva en los KPI fundamentales (u/tkt y tkt promedio)
- Norte y Sur, no muestran ninguna tendencia en particular, más bien tienen un ida y vuelta sobre los mismos valores. Es fundamental revisar semana a semana, para poder apuntar al crecimeinto
- Centro, en cambio, muestra un gran crecimiento en la productividad.
  
3. Los horarios en los que más se vende es entre las 20:00hs y las 22:00hs
- Se pudo verificar que el rango horario con más venta es de 18:00 a 20:00 y en segundo lugar el horario planteado en la hipotesis. Esto quiere decir que, a la hora de reforzar el equipo de ventas debemos apuntar en esos horarios.
  
4. El medio de pago más utilizado es tarjeta
- Pudo verificarse dicha afirmación, por lo tanto, es importante tener un seguimiento de control de las tarjetas y del costo financiero que implican. Al ser el medio de pago más usado, tener claro los pros y cons es fundamental.

## Links

[Link al Plan](https://docs.google.com/spreadsheets/d/1gcPnxIB98Yn8IsfQwROQ-1OX1a7pq0O_wbsURNq8oJc/edit?usp=sharing)

[Link al Dashboard](https://rodribustamante.github.io/DataAnalysis_Proyecto/Reporte.html)

