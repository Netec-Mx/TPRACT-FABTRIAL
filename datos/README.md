# Datos del taller

Esta carpeta contiene el dataset reproducible del taller **Arquitectura de Datos Moderna con Microsoft Fabric**.

Los datos están diseñados para cargarse manualmente en un Lakehouse de Microsoft Fabric y almacenarse en OneLake. A partir de estos archivos se construirán las capas Bronze, Silver y Gold durante los laboratorios.

---

## Archivos disponibles

| Archivo | Filas | Uso principal |
|---|---:|---|
| `ventas_2024.csv` | 67.987 | Transacciones de ventas del año 2024. |
| `ventas_2025.csv` | 67.401 | Transacciones de ventas del año 2025. |
| `ventas_2026.csv` | 31.354 | Transacciones de ventas del año 2026. |
| `productos.csv` | 500 | Catálogo de productos. |
| `clientes.csv` | 5.000 | Maestro de clientes. |
| `tiendas.csv` | 60 | Maestro de tiendas. |
| `fechas.csv` | 904 | Dimensión de fechas. |
| `presupuesto.csv` | 14.400 | Presupuesto mensual por tienda y categoría. |

El archivo `manifest_datos.csv` contiene conteos y hash SHA-256 para control de integridad.

---

## Cómo se usan en el taller

1. En el Capítulo 1, los archivos se cargan manualmente en el Lakehouse `lh_ventas_fuente`, ruta `Files/origen/ventas/raw`.
2. En el mismo capítulo, se crea un shortcut interno de OneLake desde `lh_ventas` hacia esa carpeta de origen.
3. En el Capítulo 2, un pipeline de Data Factory copia o consume esos archivos y registra tablas Delta Bronze.
4. En el Capítulo 3, un notebook transforma Bronze hacia Silver y Gold.
5. En el Capítulo 4, las tablas Gold y dimensiones se consumen desde un modelo semántico Direct Lake.
6. En el Capítulo 5, la tabla `gold_alertas_operativas` y el reporte permiten configurar alertas con Fabric Activator.

---

## Contrato de datos resumido

### ventas_*.csv

| Columna | Tipo esperado | Descripción |
|---|---|---|
| `id_transaccion` | string | Identificador único de la venta. |
| `fecha_venta` | date | Fecha de la transacción. |
| `id_cliente` | string | Llave hacia clientes. |
| `id_producto` | string | Llave hacia productos. |
| `id_tienda` | string | Llave hacia tiendas. |
| `cantidad` | integer | Unidades vendidas. |
| `precio_unitario` | decimal | Precio unitario de venta. |
| `descuento_pct` | decimal | Porcentaje de descuento aplicado. |
| `metodo_pago` | string | Medio de pago. |
| `canal_venta` | string | Canal comercial. |
| `estado_transaccion` | string | Estado de la transacción. |
| `moneda` | string | Moneda de la transacción. |
| `costo_unitario` | decimal | Costo unitario estimado. |
| `vendedor_id` | string | Identificador del vendedor. |
| `origen_dato` | string | Sistema de origen simulado. |

### productos.csv

| Columna | Tipo esperado |
|---|---|
| `id_producto` | string |
| `sku` | string |
| `nombre_producto` | string |
| `categoria` | string |
| `subcategoria` | string |
| `marca` | string |
| `precio_lista` | decimal |
| `costo_unitario` | decimal |
| `estado_producto` | string |
| `fecha_actualizacion` | date |

### clientes.csv

| Columna | Tipo esperado |
|---|---|
| `id_cliente` | string |
| `nombre_cliente` | string |
| `segmento` | string |
| `ciudad` | string |
| `region` | string |
| `departamento` | string |
| `genero` | string |
| `rango_edad` | string |
| `fecha_alta` | date |
| `estado_cliente` | string |

### tiendas.csv

| Columna | Tipo esperado |
|---|---|
| `id_tienda` | string |
| `nombre_tienda` | string |
| `ciudad` | string |
| `region` | string |
| `departamento` | string |
| `formato_tienda` | string |
| `fecha_apertura` | date |
| `estado_tienda` | string |

### fechas.csv

| Columna | Tipo esperado |
|---|---|
| `fecha` | date |
| `anio` | integer |
| `mes` | integer |
| `nombre_mes` | string |
| `trimestre` | integer |
| `semana_iso` | integer |
| `dia_mes` | integer |
| `dia_semana` | integer |
| `es_fin_semana` | string |
| `anio_mes` | string |

### presupuesto.csv

| Columna | Tipo esperado |
|---|---|
| `anio_mes` | string |
| `id_tienda` | string |
| `categoria` | string |
| `presupuesto_venta` | decimal |
| `presupuesto_unidades` | integer |
| `escenario` | string |

---

## Validaciones de carga esperadas

Después de cargar los archivos en OneLake y leerlos desde Spark, los conteos esperados son:

```text
ventas_*.csv       166742 filas
productos.csv         500 filas
clientes.csv         5000 filas
tiendas.csv            60 filas
fechas.csv            904 filas
presupuesto.csv     14400 filas
```
