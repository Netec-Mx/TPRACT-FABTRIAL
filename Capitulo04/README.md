# Alta Velocidad: Publicación de modelo Direct Lake y conexión a Power BI

## Metadatos

| Atributo | Valor |
|---|---|
| **Duración estimada** | 120 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear (Create) |
| **Laboratorio previo requerido** | Lab 03-00-01 |
| **Tecnologías principales** | Microsoft Fabric Semantic Model, Direct Lake, DAX, Power BI Desktop, Delta Lake |

---

## Descripción General

En este laboratorio construirás la capa de consumo analítico de tu arquitectura Medallion. Partirás de las tablas Delta limpias y enriquecidas que residen en la capa Silver del Lakehouse (creadas en el Lab 03) y crearás un modelo semántico en modo **Direct Lake** dentro de Microsoft Fabric, que leerá los archivos Parquet directamente desde OneLake sin necesidad de copiar ni importar datos. A continuación, definirás el esquema estrella con relaciones, medidas DAX de negocio y jerarquías; conectarás el modelo a Power BI Desktop mediante Live Connection; y construirás un informe interactivo con múltiples visualizaciones. El laboratorio concluye con la revisión del rendimiento usando el Performance Analyzer de Power BI y la configuración de opciones de optimización del modelo.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio serás capaz de:

- [ ] Crear un modelo semántico Direct Lake en Microsoft Fabric apuntando a las tablas Delta de la capa Silver del Lakehouse
- [ ] Definir relaciones de esquema estrella, medidas DAX, jerarquías y perspectivas dentro del editor de modelo semántico de Fabric
- [ ] Conectar Power BI Desktop al modelo semántico publicado en Fabric mediante Live Connection y construir un informe interactivo con mínimo 5 visualizaciones
- [ ] Comparar el comportamiento de Direct Lake vs Import Mode usando el Performance Analyzer de Power BI y explicar las diferencias arquitectónicas
- [ ] Configurar opciones de optimización del modelo semántico para consultas de alta velocidad y publicar el informe en el Workspace de Fabric

---

## Prerrequisitos

### Conocimiento Previo

| Área | Nivel Requerido |
|---|---|
| Microsoft Fabric (Lakehouse, Workspace, Notebooks) | Intermedio — completar Labs 01, 02 y 03 |
| Power BI Desktop y Power BI Service | Intermedio — creación de informes, publicación |
| Modelado dimensional (esquema estrella, hechos y dimensiones) | Básico — comprensión conceptual |
| DAX básico (SUM, CALCULATE, DIVIDE, funciones de tiempo) | Básico — haber escrito al menos medidas simples |
| Delta Lake (tablas registradas en metastore) | Básico — comprendido en Lab 03 |

### Acceso y Recursos

| Recurso | Estado Requerido |
|---|---|
| Workspace de Microsoft Fabric con capacidad Trial activa | ✅ Activo (creado en Lab 01) |
| Lakehouse `LH_Ventas` con tablas Silver registradas | ✅ Disponible (creado en Lab 03) |
| Power BI Desktop versión mayo 2024 o superior | ✅ Instalado en el equipo local |
| Navegador Microsoft Edge o Chrome v110+ | ✅ Disponible |
| Acceso al portal de Fabric: `https://app.fabric.microsoft.com` | ✅ Sesión iniciada con cuenta del curso |

> ⚠️ **DEPENDENCIA CRÍTICA**: Este laboratorio requiere que las siguientes tablas Delta estén registradas en la sección **Tables** del Lakehouse `LH_Ventas`: `silver_ventas`, `silver_clientes`, `silver_productos`, `silver_tiendas` y `dim_fecha`. Si alguna de estas tablas no aparece en la sección Tables (solo aparece en Files), ejecuta el paso de registro al inicio de este lab antes de continuar.

---

## Entorno de Laboratorio

### Hardware Recomendado

| Componente | Mínimo | Recomendado |
|---|---|---|
| RAM | 8 GB | 16 GB (para Power BI Desktop + navegador simultáneamente) |
| Procesador | Intel Core i5 8ª gen / AMD Ryzen 5 | Intel Core i7 / AMD Ryzen 7 |
| Almacenamiento libre | 500 MB | 1 GB |
| Resolución de pantalla | 1366×768 | 1920×1080 |
| Conexión a Internet | 10 Mbps | 25 Mbps |

### Software Requerido

| Software | Versión | Uso en este Lab |
|---|---|---|
| Microsoft Edge / Chrome | v110+ | Portal de Fabric, editor de modelo semántico |
| Power BI Desktop | Mayo 2024+ | Conexión Live, construcción de informe |
| Microsoft Fabric (browser) | Trial activo | Modelo semántico, publicación |

### Verificación del Entorno Antes de Comenzar

Antes de iniciar el laboratorio, verifica que el entorno esté listo ejecutando las siguientes comprobaciones:

**Comprobación 1 — Tablas Silver disponibles en el Lakehouse:**

Abre el portal de Fabric (`https://app.fabric.microsoft.com`), navega a tu Workspace y abre el Lakehouse `LH_Ventas`. En el panel izquierdo, expande la sección **Tables** y confirma que aparecen las siguientes tablas:

```
Tables/
├── silver_ventas
├── silver_clientes
├── silver_productos
├── silver_tiendas
└── dim_fecha
```

**Comprobación 2 — Registrar tablas faltantes (si aplica):**

Si alguna tabla no aparece en la sección Tables, abre un Notebook en el mismo Workspace y ejecuta el siguiente código para registrarla:

```python
# Ejecutar solo si la tabla NO aparece en la sección Tables del Lakehouse
# Reemplaza 'nombre_tabla' con el nombre real de la tabla faltante

tablas_requeridas = [
    "silver_ventas",
    "silver_clientes", 
    "silver_productos",
    "silver_tiendas",
    "dim_fecha"
]

for tabla in tablas_requeridas:
    spark.sql(f"""
        CREATE TABLE IF NOT EXISTS {tabla}
        USING DELTA
        LOCATION 'Tables/{tabla}'
    """)
    print(f"✅ Tabla registrada: {tabla}")

# Verificar que todas las tablas están disponibles
tablas_disponibles = [t.name for t in spark.catalog.listTables()]
print("\n📋 Tablas disponibles en el metastore:")
for t in tablas_disponibles:
    print(f"  - {t}")
```

**Comprobación 3 — Power BI Desktop instalado:**

Abre Power BI Desktop en tu equipo. En el menú **Ayuda → Acerca de**, confirma que la versión es mayo 2024 (2.129.x) o superior.

---

## Instrucciones Paso a Paso

---

### Paso 1: Explorar el SQL Endpoint del Lakehouse y Verificar las Tablas Silver

**Objetivo:** Confirmar que todas las tablas Delta de la capa Silver están accesibles a través del SQL Endpoint del Lakehouse, que es la interfaz que utilizará el modelo semántico Direct Lake para resolver consultas en modo fallback.

**Instrucciones:**

1. En el portal de Fabric (`https://app.fabric.microsoft.com`), navega a tu **Workspace** del curso.

2. En la lista de ítems del Workspace, localiza el ítem **`LH_Ventas`** de tipo **Lakehouse** y haz clic sobre él para abrirlo.

3. En la esquina superior derecha del Lakehouse, localiza el selector de modo. Haz clic en el desplegable que dice **Lakehouse** y selecciona **SQL analytics endpoint**.

   > 📌 El SQL Endpoint proporciona una interfaz SQL sobre las tablas Delta del Lakehouse. Es el mecanismo de fallback que usa Direct Lake cuando no puede resolver una consulta desde el caché en memoria.

4. En el editor SQL del endpoint, ejecuta la siguiente consulta para verificar el contenido de las tablas:

   ```sql
   -- Verificar estructura y conteo de tablas Silver
   SELECT 
       'silver_ventas' AS tabla,
       COUNT(*) AS total_registros,
       MIN(fecha_venta) AS fecha_minima,
       MAX(fecha_venta) AS fecha_maxima
   FROM silver_ventas
   
   UNION ALL
   
   SELECT 
       'silver_clientes',
       COUNT(*),
       NULL,
       NULL
   FROM silver_clientes
   
   UNION ALL
   
   SELECT 
       'silver_productos',
       COUNT(*),
       NULL,
       NULL
   FROM silver_productos
   
   UNION ALL
   
   SELECT 
       'silver_tiendas',
       COUNT(*),
       NULL,
       NULL
   FROM silver_tiendas
   
   UNION ALL
   
   SELECT 
       'dim_fecha',
       COUNT(*),
       NULL,
       NULL
   FROM dim_fecha;
   ```

5. Anota los conteos de registros de cada tabla. Los usarás como referencia de validación más adelante.

6. Ejecuta también esta consulta para revisar las columnas clave que usarás en el modelo:

   ```sql
   -- Revisar columnas de la tabla de hechos
   SELECT TOP 5
       id_venta,
       id_cliente,
       id_producto,
       id_tienda,
       fecha_venta,
       cantidad,
       precio_unitario,
       descuento,
       monto_total,
       costo_total,
       margen
   FROM silver_ventas
   ORDER BY fecha_venta DESC;
   ```

**Salida Esperada:**

La primera consulta debe retornar 5 filas, una por cada tabla, con conteos de registros mayores a cero. La segunda consulta debe mostrar 5 filas de la tabla `silver_ventas` con todas las columnas indicadas sin valores nulos en los campos clave.

**Verificación:**

✅ Todas las tablas retornan registros (COUNT > 0).  
✅ La tabla `silver_ventas` contiene las columnas `id_cliente`, `id_producto`, `id_tienda`, `fecha_venta` y `monto_total`.  
✅ No hay errores de tipo "table not found" o "column not found".

---

### Paso 2: Crear un Modelo Semántico Personalizado en Fabric

**Objetivo:** Crear un nuevo modelo semántico personalizado (no el predeterminado) en el Workspace de Fabric, seleccionando únicamente las tablas Silver necesarias para el análisis de ventas.

**Instrucciones:**

1. Regresa a la vista principal del Lakehouse `LH_Ventas` (cambia el selector de vuelta a **Lakehouse**).

2. En la barra de herramientas superior del Lakehouse, haz clic en el botón **New semantic model** (o busca la opción en el menú **...** → **New semantic model**).

   > 📌 Esta opción crea un modelo semántico personalizado que tú controlas, a diferencia del modelo predeterminado que Fabric genera automáticamente. El modelo personalizado te permite seleccionar tablas específicas, renombrarlas y configurar propiedades avanzadas.

3. En el panel que aparece, asigna el nombre: **`SM_Ventas_DirectLake`**

4. En la lista de tablas disponibles, selecciona las siguientes tablas marcando sus casillas:
   - ✅ `silver_ventas`
   - ✅ `silver_clientes`
   - ✅ `silver_productos`
   - ✅ `silver_tiendas`
   - ✅ `dim_fecha`

5. Haz clic en **Confirm** (o **Crear**) para crear el modelo semántico.

6. Fabric creará el modelo y abrirá automáticamente el **editor de modelo semántico**. Confirma que en la barra de estado o en las propiedades del modelo aparece el indicador **Direct Lake** como modo de almacenamiento.

7. Verifica que en el panel de **Data** (o **Datos**) del editor aparecen las 5 tablas seleccionadas.

**Salida Esperada:**

El editor de modelo semántico se abre con las 5 tablas visibles en el panel de datos. El modo del modelo es **Direct Lake**. En el Workspace, aparece un nuevo ítem de tipo **Semantic model** con el nombre `SM_Ventas_DirectLake`.

**Verificación:**

✅ El ítem `SM_Ventas_DirectLake` aparece en el Workspace con el ícono de modelo semántico.  
✅ Las 5 tablas están visibles en el editor del modelo.  
✅ El modo de almacenamiento es Direct Lake (verificable en las propiedades del modelo).

---

### Paso 3: Definir el Esquema Estrella — Relaciones entre Tablas

**Objetivo:** Configurar las relaciones entre la tabla de hechos `silver_ventas` y las tablas de dimensiones, estableciendo el esquema estrella que permitirá análisis multidimensional en Power BI.

**Instrucciones:**

1. Dentro del editor del modelo semántico `SM_Ventas_DirectLake`, navega a la vista de **Modelo** (ícono de diagrama en el panel izquierdo, similar a la vista de relaciones de Power BI Desktop).

2. Arrastra las tablas para organizar el diagrama en forma de estrella:
   - Coloca `silver_ventas` en el **centro** (tabla de hechos).
   - Coloca `silver_clientes`, `silver_productos`, `silver_tiendas` y `dim_fecha` alrededor (tablas de dimensiones).

3. Crea la relación entre `silver_ventas` y `silver_clientes`:
   - Haz clic en el campo **`id_cliente`** de la tabla `silver_ventas` y arrástralo hasta el campo **`id_cliente`** de la tabla `silver_clientes`.
   - En el panel de propiedades de la relación, configura:
     - **Cardinalidad**: Muchos a uno (*:1)
     - **Dirección del filtro cruzado**: Único (Single) — desde dimensión hacia hechos
     - **Activa**: ✅ Sí

4. Crea la relación entre `silver_ventas` y `silver_productos`:
   - Arrastra **`id_producto`** de `silver_ventas` hacia **`id_producto`** de `silver_productos`.
   - Cardinalidad: Muchos a uno (*:1), filtro único, activa.

5. Crea la relación entre `silver_ventas` y `silver_tiendas`:
   - Arrastra **`id_tienda`** de `silver_ventas` hacia **`id_tienda`** de `silver_tiendas`.
   - Cardinalidad: Muchos a uno (*:1), filtro único, activa.

6. Crea la relación entre `silver_ventas` y `dim_fecha`:
   - Arrastra **`fecha_venta`** de `silver_ventas` hacia **`fecha`** de `dim_fecha`.
   - Cardinalidad: Muchos a uno (*:1), filtro único, activa.

   > ⚠️ **Nota importante**: La columna de fecha en `silver_ventas` debe ser de tipo `date` o `datetime`. Si la relación no se puede crear por incompatibilidad de tipos, ejecuta el siguiente código en un Notebook para corregir el tipo de dato y luego actualiza el modelo:

   ```python
   from pyspark.sql.functions import col, to_date
   from delta.tables import DeltaTable
   
   # Corregir tipo de dato de fecha_venta si es necesario
   df_ventas = spark.read.format("delta").load("Tables/silver_ventas")
   
   # Verificar tipo actual
   print("Tipo actual de fecha_venta:", df_ventas.schema["fecha_venta"].dataType)
   
   # Si es StringType, convertir a DateType
   df_ventas_corregido = df_ventas.withColumn(
       "fecha_venta", 
       to_date(col("fecha_venta"), "yyyy-MM-dd")
   )
   
   # Sobreescribir la tabla Delta
   df_ventas_corregido.write \
       .format("delta") \
       .mode("overwrite") \
       .option("overwriteSchema", "true") \
       .saveAsTable("silver_ventas")
   
   print("✅ Tipo de dato corregido. Regresa al editor del modelo y actualiza las tablas.")
   ```

7. Verifica que el diagrama muestra las 4 líneas de relación conectando `silver_ventas` con cada dimensión. Las líneas deben mostrar el símbolo `*` en el lado de hechos y `1` en el lado de dimensiones.

**Salida Esperada:**

El diagrama del modelo muestra un esquema estrella claro: `silver_ventas` en el centro con 4 líneas de relación hacia `silver_clientes`, `silver_productos`, `silver_tiendas` y `dim_fecha`. No hay advertencias de relaciones ambiguas o inactivas.

**Verificación:**

✅ Existen exactamente 4 relaciones activas en el modelo.  
✅ Todas las relaciones son de cardinalidad Muchos a uno (*:1).  
✅ No aparecen advertencias de "relación ambigua" o "relación inactiva no deseada".  
✅ El diagrama tiene forma de estrella con `silver_ventas` como tabla central.

---

### Paso 4: Crear Medidas DAX de Negocio

**Objetivo:** Definir las medidas DAX fundamentales para el análisis de ventas, incluyendo totales, variaciones temporales, márgenes y KPIs de negocio, que serán reutilizables en todos los informes conectados al modelo.

**Instrucciones:**

1. En el editor del modelo semántico, selecciona la tabla `silver_ventas` en el panel de datos.

2. Haz clic en **Nueva medida** (o el botón `+` junto a la tabla) para abrir el editor DAX.

3. Crea las siguientes medidas una por una. Para cada medida, escribe la expresión DAX, verifica que no hay errores de sintaxis y guarda:

   **Medida 1 — Total Ventas:**
   ```dax
   Total Ventas = 
   SUM(silver_ventas[monto_total])
   ```

   **Medida 2 — Total Unidades Vendidas:**
   ```dax
   Total Unidades = 
   SUM(silver_ventas[cantidad])
   ```

   **Medida 3 — Costo Total:**
   ```dax
   Costo Total = 
   SUM(silver_ventas[costo_total])
   ```

   **Medida 4 — Margen Bruto:**
   ```dax
   Margen Bruto = 
   [Total Ventas] - [Costo Total]
   ```

   **Medida 5 — Margen % :**
   ```dax
   Margen % = 
   DIVIDE(
       [Margen Bruto],
       [Total Ventas],
       0
   )
   ```

   **Medida 6 — Ventas YTD (Año hasta la fecha):**
   ```dax
   Ventas YTD = 
   TOTALYTD(
       [Total Ventas],
       dim_fecha[fecha]
   )
   ```

   **Medida 7 — Ventas Período Anterior (mismo período del año anterior):**
   ```dax
   Ventas Año Anterior = 
   CALCULATE(
       [Total Ventas],
       SAMEPERIODLASTYEAR(dim_fecha[fecha])
   )
   ```

   **Medida 8 — Variación vs Año Anterior (absoluta):**
   ```dax
   Variación vs Año Anterior = 
   [Total Ventas] - [Ventas Año Anterior]
   ```

   **Medida 9 — Variación vs Año Anterior (%):**
   ```dax
   Variación % vs Año Anterior = 
   DIVIDE(
       [Variación vs Año Anterior],
       [Ventas Año Anterior],
       BLANK()
   )
   ```

   **Medida 10 — Ticket Promedio:**
   ```dax
   Ticket Promedio = 
   DIVIDE(
       [Total Ventas],
       DISTINCTCOUNT(silver_ventas[id_venta]),
       0
   )
   ```

4. Para las medidas de formato monetario (`Total Ventas`, `Costo Total`, `Margen Bruto`, `Ventas YTD`, `Ventas Año Anterior`, `Variación vs Año Anterior`, `Ticket Promedio`), selecciona cada medida y en el panel de propiedades configura:
   - **Formato**: Moneda
   - **Símbolo de moneda**: $ (o el símbolo correspondiente al dataset del curso)
   - **Decimales**: 2

5. Para `Margen %` y `Variación % vs Año Anterior`:
   - **Formato**: Porcentaje
   - **Decimales**: 1

6. Crea una **tabla de medidas** dedicada para organizar las medidas. Para esto:
   - En el editor, busca la opción **Ingresar datos** o **Nueva tabla calculada**.
   - Crea una tabla vacía llamada `_Medidas` con la expresión:
   
   ```dax
   _Medidas = ROW("placeholder", BLANK())
   ```
   
   - Mueve todas las medidas creadas a esta tabla usando la opción **Mover a tabla** en las propiedades de cada medida.
   - Oculta la columna `placeholder` de la tabla `_Medidas`.

   > 📌 Organizar las medidas en una tabla dedicada es una buena práctica de modelado: facilita la navegación en Power BI Desktop y evita que las medidas aparezcan dispersas entre las columnas de las tablas de hechos.

**Salida Esperada:**

La tabla `_Medidas` aparece en el panel de datos con las 10 medidas creadas. Cada medida muestra el ícono de calculadora (∑). No hay errores de sintaxis DAX en ninguna medida.

**Verificación:**

✅ Las 10 medidas están creadas y agrupadas en la tabla `_Medidas`.  
✅ Las medidas monetarias tienen formato de moneda con 2 decimales.  
✅ `Margen %` y `Variación % vs Año Anterior` tienen formato de porcentaje.  
✅ Ninguna medida muestra el ícono de error (triángulo amarillo).

---

### Paso 5: Definir Jerarquías y Configurar la Tabla de Fechas

**Objetivo:** Crear jerarquías de tiempo y geografía que permitan a los usuarios hacer drill-down en los informes, y marcar la tabla `dim_fecha` como tabla de fechas para habilitar las funciones de inteligencia de tiempo en DAX.

**Instrucciones:**

1. **Marcar `dim_fecha` como tabla de fechas:**
   - Selecciona la tabla `dim_fecha` en el panel de datos.
   - Busca la opción **Marcar como tabla de fechas** (puede estar en el menú contextual o en la barra de herramientas del editor).
   - En el diálogo, selecciona la columna **`fecha`** como columna de fecha.
   - Confirma. Fabric validará que la columna contiene fechas únicas y continuas.

   > ⚠️ Si `dim_fecha` no tiene una columna `fecha` con fechas únicas y sin huecos, la validación fallará. En ese caso, verifica en el Lab 03 que la tabla de fechas fue generada correctamente.

2. **Crear la Jerarquía de Tiempo en `dim_fecha`:**
   - En la tabla `dim_fecha`, localiza las columnas: `anio`, `trimestre`, `mes`, `semana` y `fecha`.
   - Haz clic derecho sobre la columna `anio` y selecciona **Crear jerarquía**.
   - Nombra la jerarquía: **`Jerarquía Temporal`**
   - Agrega los siguientes niveles en orden arrastrando las columnas a la jerarquía:
     1. `anio` → renombrar nivel como **Año**
     2. `trimestre` → renombrar nivel como **Trimestre**
     3. `mes` → renombrar nivel como **Mes**
     4. `fecha` → renombrar nivel como **Día**

3. **Crear la Jerarquía Geográfica en `silver_tiendas`:**
   - En la tabla `silver_tiendas`, localiza las columnas: `pais`, `region`, `ciudad` y `nombre_tienda`.
   - Crea una nueva jerarquía llamada **`Jerarquía Geográfica`** con los niveles:
     1. `pais` → **País**
     2. `region` → **Región**
     3. `ciudad` → **Ciudad**
     4. `nombre_tienda` → **Tienda**

4. **Crear la Jerarquía de Producto en `silver_productos`:**
   - En la tabla `silver_productos`, localiza las columnas: `categoria`, `subcategoria` y `nombre_producto`.
   - Crea una jerarquía llamada **`Jerarquía Producto`** con los niveles:
     1. `categoria` → **Categoría**
     2. `subcategoria` → **Subcategoría**
     3. `nombre_producto` → **Producto**

5. **Configurar columnas de ordenamiento:**
   - En `dim_fecha`, selecciona la columna `nombre_mes` (nombre del mes en texto).
   - En las propiedades, configura **Ordenar por columna** → `numero_mes` (número del mes 1-12).
   - Repite para `nombre_trimestre` → ordenar por `numero_trimestre`.

6. **Ocultar columnas de clave foránea en la tabla de hechos:**
   - En `silver_ventas`, oculta las columnas que son claves foráneas y no deben aparecer directamente en los informes:
     - Haz clic derecho sobre `id_cliente` → **Ocultar en vista de informe**
     - Repite para: `id_producto`, `id_tienda`, `fecha_venta` (la fecha se accede a través de `dim_fecha`)
   
   > 📌 Ocultar las claves foráneas evita que los usuarios arrastren columnas de ID al informe en lugar de usar las dimensiones descriptivas. Esto es una práctica estándar de modelado dimensional.

**Salida Esperada:**

Las tres jerarquías aparecen en sus respectivas tablas con el ícono de jerarquía. La tabla `dim_fecha` muestra el ícono especial de tabla de fechas. Las columnas de clave foránea en `silver_ventas` aparecen con el ícono de "oculto" (ojo tachado).

**Verificación:**

✅ `dim_fecha` está marcada como tabla de fechas sin errores de validación.  
✅ Las jerarquías `Jerarquía Temporal`, `Jerarquía Geográfica` y `Jerarquía Producto` existen con sus niveles correctos.  
✅ Las columnas `id_cliente`, `id_producto`, `id_tienda` en `silver_ventas` están ocultas en la vista de informe.  
✅ El mes se ordena correctamente por número (Enero=1, Febrero=2, ...).

---

### Paso 6: Guardar y Publicar el Modelo Semántico

**Objetivo:** Guardar todos los cambios del modelo semántico en el Workspace de Fabric y verificar que el modelo está publicado y accesible para conexiones externas desde Power BI Desktop.

**Instrucciones:**

1. En el editor del modelo semántico, haz clic en el botón **Guardar** (ícono de disquete o `Ctrl+S`).

2. Espera a que Fabric confirme que los cambios se guardaron. Deberías ver un mensaje de confirmación en la barra de notificaciones.

3. Navega al **Workspace** del curso. Confirma que el ítem `SM_Ventas_DirectLake` aparece en la lista con el tipo **Semantic model**.

4. Haz clic en los tres puntos (`...`) junto al modelo `SM_Ventas_DirectLake` y selecciona **Settings** (Configuración).

5. En la sección **Gateway and cloud connections**, verifica que no se requiere una gateway on-premises (Direct Lake sobre OneLake no la necesita).

6. En la sección **Server settings** o **Advanced**, localiza la opción **Direct Lake behavior** y verifica que está configurada en **Automatic** (comportamiento predeterminado que permite fallback a DirectQuery si es necesario).

7. Copia la **URL del modelo semántico** desde la barra de direcciones del navegador cuando estés en la página del modelo. Tendrá un formato similar a:
   ```
   https://app.fabric.microsoft.com/groups/{workspace-id}/datasets/{dataset-id}
   ```
   Guarda esta URL; la usarás para conectarte desde Power BI Desktop.

8. Alternativamente, anota el **nombre del Workspace** y el **nombre del modelo semántico** (`SM_Ventas_DirectLake`), que son suficientes para conectarse desde Power BI Desktop.

**Salida Esperada:**

El modelo `SM_Ventas_DirectLake` aparece publicado en el Workspace con el indicador de modo Direct Lake. La configuración no muestra requerimientos de gateway. El modelo está listo para recibir conexiones.

**Verificación:**

✅ El ítem `SM_Ventas_DirectLake` aparece en el Workspace.  
✅ No se requiere gateway on-premises.  
✅ El comportamiento Direct Lake está configurado como Automatic.  
✅ Tienes anotado el nombre del Workspace y del modelo semántico.

---

### Paso 7: Conectar Power BI Desktop al Modelo Semántico (Live Connection)

**Objetivo:** Establecer una conexión Live desde Power BI Desktop al modelo semántico `SM_Ventas_DirectLake` publicado en Fabric, sin crear una copia local de los datos.

**Instrucciones:**

1. Abre **Power BI Desktop** en tu equipo local.

2. En la pantalla de inicio, haz clic en **Obtener datos** (o `Ctrl+Alt+F5`).

3. En el buscador del diálogo "Obtener datos", escribe **Power BI** y selecciona **Conjuntos de datos de Power BI** (Power BI datasets / Semantic Models).

4. Haz clic en **Conectar**.

5. En el panel de OneLake Data Hub que aparece, busca el modelo `SM_Ventas_DirectLake`. Puedes:
   - Usar el buscador del panel y escribir `SM_Ventas`
   - Navegar por el Workspace del curso en el panel izquierdo

6. Selecciona **`SM_Ventas_DirectLake`** y haz clic en **Conectar** (no en "Importar").

   > ⚠️ **Importante**: Debes hacer clic en **Conectar** (que establece una Live Connection) y NO en "Importar". Si importas, crearás una copia local de los datos y perderás el modo Direct Lake. La opción correcta establece una conexión en vivo que delega todas las consultas al modelo semántico en Fabric.

7. Power BI Desktop establecerá la conexión y mostrará en la barra de estado inferior el texto **"Conectado en directo a: SM_Ventas_DirectLake"** (o "Live connection to: SM_Ventas_DirectLake").

8. En el panel **Datos** (lado derecho de Power BI Desktop), verifica que aparecen:
   - La tabla `_Medidas` con las 10 medidas creadas
   - Las tablas `silver_ventas`, `silver_clientes`, `silver_productos`, `silver_tiendas` y `dim_fecha`
   - Las jerarquías dentro de cada tabla

9. Verifica que **NO** puedes crear nuevas tablas ni columnas calculadas en Power BI Desktop (estas opciones deben estar deshabilitadas en modo Live Connection). Esto es el comportamiento esperado; el modelo se edita en Fabric, no en el cliente.

**Salida Esperada:**

Power BI Desktop muestra la barra de estado con "Conectado en directo a: SM_Ventas_DirectLake". El panel de datos muestra todas las tablas y medidas del modelo. Las opciones de "Nueva tabla" y "Nueva columna" están deshabilitadas en la cinta de opciones.

**Verificación:**

✅ La barra de estado de Power BI Desktop indica "Live connection" al modelo `SM_Ventas_DirectLake`.  
✅ Las 10 medidas de la tabla `_Medidas` son visibles en el panel de datos.  
✅ Las 3 jerarquías están disponibles en sus respectivas tablas.  
✅ Las opciones de modelado local (nueva tabla, nueva columna calculada) están deshabilitadas.

---

### Paso 8: Construir el Informe Interactivo en Power BI Desktop

**Objetivo:** Crear un informe con mínimo 5 visualizaciones que demuestren las capacidades analíticas del modelo Direct Lake, incluyendo drill-down, filtros cruzados y KPIs de negocio.

**Instrucciones:**

1. En Power BI Desktop, crea una nueva página de informe y nómbrala **"Resumen Ejecutivo"**.

2. **Visualización 1 — Tarjetas KPI (mínimo 3 tarjetas):**
   - Inserta una visualización de tipo **Tarjeta** (Card).
   - Arrastra la medida **`Total Ventas`** al campo Valor.
   - Formatea la tarjeta: título "Ventas Totales", sin decimales.
   - Duplica la tarjeta dos veces y reemplaza la medida por:
     - Segunda tarjeta: **`Margen %`** — título "Margen Bruto %"
     - Tercera tarjeta: **`Ticket Promedio`** — título "Ticket Promedio"
   - Alinea las tres tarjetas en la parte superior del canvas.

3. **Visualización 2 — Gráfico de Barras: Ventas por Categoría:**
   - Inserta un **Gráfico de barras agrupadas** (Clustered bar chart).
   - Eje Y: `silver_productos[categoria]`
   - Valores: `_Medidas[Total Ventas]`
   - Ordena las barras de mayor a menor por Total Ventas.
   - Título: "Ventas por Categoría de Producto"

4. **Visualización 3 — Gráfico de Líneas: Tendencia Temporal:**
   - Inserta un **Gráfico de líneas** (Line chart).
   - Eje X: `dim_fecha[Jerarquía Temporal]` (arrastra la jerarquía completa, no solo el año)
   - Valores: `_Medidas[Total Ventas]` y `_Medidas[Ventas Año Anterior]`
   - Configura las dos líneas con colores distintos y agrega leyenda.
   - Título: "Tendencia de Ventas vs Año Anterior"
   - Habilita el drill-down en el eje X para que los usuarios puedan navegar de Año → Trimestre → Mes → Día.

5. **Visualización 4 — Mapa Geográfico:**
   - Inserta un **Mapa** (Map) o **Mapa de burbujas** (Bubble map).
   - Ubicación: `silver_tiendas[ciudad]`
   - Tamaño de burbuja: `_Medidas[Total Ventas]`
   - Información sobre herramientas (Tooltip): agregar `_Medidas[Margen %]`
   - Título: "Ventas por Ciudad"

   > ⚠️ Si el mapa no reconoce las ciudades automáticamente, agrega también `silver_tiendas[pais]` al campo de ubicación para dar contexto geográfico.

6. **Visualización 5 — Tabla de Detalle con Drill-Through:**
   - Crea una **segunda página** en el informe y nómbrala **"Detalle por Producto"**.
   - En esta página, inserta una **Tabla** (Table visual).
   - Columnas: `silver_productos[nombre_producto]`, `silver_productos[categoria]`, `_Medidas[Total Ventas]`, `_Medidas[Total Unidades]`, `_Medidas[Margen %]`, `_Medidas[Variación % vs Año Anterior]`
   - Ordena la tabla por `Total Ventas` descendente.
   - Configura el **Drill-through**: en el panel de visualización, arrastra `silver_productos[categoria]` al campo **Drill-through**. Esto permitirá que desde la página "Resumen Ejecutivo", el usuario haga clic derecho sobre una categoría y navegue directamente a esta página filtrada.

7. **Agregar Segmentadores (Slicers) en la página "Resumen Ejecutivo":**
   - Inserta un **Segmentador** (Slicer) para `dim_fecha[anio]` — tipo: Lista o Desplegable.
   - Inserta un segundo segmentador para `silver_tiendas[region]`.
   - Posiciona ambos segmentadores en el margen izquierdo del canvas.

8. **Formatear el informe:**
   - Aplica un tema de color consistente (puedes usar el tema predeterminado de Power BI o uno personalizado).
   - Agrega un título de página en la parte superior: "Dashboard de Ventas — Análisis Direct Lake".
   - Asegúrate de que todas las visualizaciones tienen títulos descriptivos.

**Salida Esperada:**

El informe tiene dos páginas: "Resumen Ejecutivo" con 3 tarjetas KPI, gráfico de barras, gráfico de líneas, mapa y 2 segmentadores; y "Detalle por Producto" con la tabla de drill-through configurada. Al hacer clic en las visualizaciones, los filtros cruzados funcionan correctamente entre ellas.

**Verificación:**

✅ La página "Resumen Ejecutivo" contiene mínimo 5 visualizaciones (3 tarjetas + barras + líneas + mapa = 6).  
✅ El gráfico de líneas muestra dos series: "Total Ventas" y "Ventas Año Anterior".  
✅ El drill-down en la jerarquía temporal funciona (Año → Trimestre → Mes).  
✅ El drill-through desde la página "Resumen Ejecutivo" hacia "Detalle por Producto" funciona al hacer clic derecho sobre una categoría.  
✅ Los segmentadores filtran todas las visualizaciones de la página simultáneamente.

---

### Paso 9: Analizar el Rendimiento con Performance Analyzer

**Objetivo:** Usar el Performance Analyzer de Power BI Desktop para medir los tiempos de respuesta de las visualizaciones en modo Direct Lake y comparar el comportamiento con el modo Import, comprendiendo las diferencias arquitectónicas.

**Instrucciones:**

1. En Power BI Desktop, ve al menú **Vista** → **Performance Analyzer** (Analizador de rendimiento).

2. En el panel Performance Analyzer que aparece en el lado derecho, haz clic en **Iniciar grabación** (Start recording).

3. Haz clic en el botón **Actualizar elementos visuales** (Refresh visuals) para forzar la recarga de todas las visualizaciones de la página actual.

4. Espera a que todas las visualizaciones terminen de cargar. Observa los tiempos que aparecen en el Performance Analyzer para cada visual.

5. Expande cada elemento en el Performance Analyzer y anota los siguientes tiempos:
   - **DAX query**: tiempo que tardó el motor DAX en procesar la consulta
   - **Visual display**: tiempo de renderizado del visual
   - **Other**: tiempo adicional de procesamiento

6. Registra los tiempos en la siguiente tabla de comparación (completa la columna Direct Lake con tus observaciones):

   | Visual | DAX Query (ms) | Visual Display (ms) | Total (ms) |
   |---|---|---|---|
   | Tarjeta Total Ventas | ___ | ___ | ___ |
   | Gráfico de Barras | ___ | ___ | ___ |
   | Gráfico de Líneas | ___ | ___ | ___ |
   | Mapa | ___ | ___ | ___ |

7. Haz clic en **Detener grabación** (Stop recording).

8. Para entender el comportamiento de fallback, ejecuta la siguiente consulta DAX en el **DAX Query View** de Power BI Desktop (si está disponible en tu versión) o en el editor de modelo en Fabric:

   ```dax
   -- Consulta de prueba para verificar que Direct Lake está activo
   EVALUATE
   SUMMARIZECOLUMNS(
       silver_productos[categoria],
       "Total Ventas", [Total Ventas],
       "Margen %", [Margen %]
   )
   ORDER BY [Total Ventas] DESC
   ```

9. Observa el tiempo de respuesta de esta consulta. En modo Direct Lake sobre un dataset pequeño/mediano, debería responder en menos de 1-2 segundos.

10. **Discusión de comparación de modos** (reflexión guiada):

    Responde las siguientes preguntas en base a lo observado:
    
    - ¿Los tiempos DAX Query son similares a los que esperarías de un modelo Import (generalmente < 100ms para datasets pequeños)?
    - ¿Observaste algún tiempo de "framing" inicial cuando cargaste el informe por primera vez?
    - ¿Qué ocurre con los tiempos cuando aplicas un segmentador que filtra significativamente los datos?

    > 📌 **Nota sobre el framing**: La primera vez que se ejecuta una consulta sobre una tabla, Direct Lake realiza el "framing" — carga las columnas necesarias desde OneLake al caché en memoria. Las consultas subsiguientes sobre las mismas columnas serán más rápidas porque los datos ya están en caché. Esto explica por qué la primera interacción puede ser ligeramente más lenta que las siguientes.

**Salida Esperada:**

El Performance Analyzer muestra los tiempos de respuesta de cada visual. Los tiempos DAX Query deberían ser comparables a los de un modelo Import (generalmente bajo 500ms para el dataset del curso). Puede observarse un tiempo inicial ligeramente mayor en la primera carga (framing), seguido de tiempos más rápidos en interacciones posteriores.

**Verificación:**

✅ El Performance Analyzer registró tiempos para todas las visualizaciones de la página.  
✅ Tienes anotados los tiempos de DAX Query para al menos 3 visualizaciones.  
✅ La consulta DAX de prueba retorna resultados en menos de 5 segundos (en capacidad Trial con carga variable).  
✅ Puedes explicar la diferencia entre el tiempo de framing inicial y las consultas en caché.

---

### Paso 10: Publicar el Informe en el Workspace de Fabric

**Objetivo:** Publicar el informe de Power BI Desktop en el Workspace de Microsoft Fabric para que esté disponible en el portal web y pueda ser compartido con otros usuarios.

**Instrucciones:**

1. En Power BI Desktop, guarda el archivo del informe localmente:
   - `Archivo` → `Guardar como`
   - Nombre: **`Informe_Ventas_DirectLake.pbix`**
   - Guarda en una carpeta local de fácil acceso.

2. Haz clic en **Publicar** en la cinta de opciones (pestaña **Inicio** → **Publicar**).

3. En el diálogo de selección de destino, busca y selecciona el **Workspace del curso** (el mismo donde está el Lakehouse y el modelo semántico).

4. Haz clic en **Seleccionar**.

5. Power BI Desktop mostrará una barra de progreso de publicación. Espera a que aparezca el mensaje de confirmación: **"Publicación correcta"**.

   > ⚠️ **Importante**: Durante la publicación, Power BI Desktop intentará publicar tanto el informe como el modelo de datos. Como estás usando una Live Connection, el modelo de datos no se duplica — solo se publica el informe que apunta al modelo `SM_Ventas_DirectLake` ya existente en Fabric.

6. Haz clic en el enlace **"Abrir 'Informe_Ventas_DirectLake.pbix' en Power BI"** para abrir el informe publicado en el navegador.

7. En el portal de Fabric, verifica que el informe aparece en el Workspace con el tipo **Report**.

8. Navega por el informe en el navegador y confirma que:
   - Las visualizaciones cargan correctamente
   - Los segmentadores funcionan
   - El drill-through funciona desde la página "Resumen Ejecutivo" hacia "Detalle por Producto"

9. **Opcional — Configurar actualización automática:**
   - En el portal de Fabric, haz clic en los tres puntos (`...`) junto al modelo `SM_Ventas_DirectLake`.
   - Selecciona **Settings** → **Scheduled refresh**.
   - Observa que para modelos Direct Lake, la opción de actualización programada no es necesaria (los datos se leen directamente desde OneLake). Sin embargo, puedes configurar una **actualización de caché** si deseas pre-cargar las columnas en memoria a una hora específica para garantizar la máxima velocidad en las primeras consultas del día.

**Salida Esperada:**

El informe `Informe_Ventas_DirectLake` aparece publicado en el Workspace de Fabric. Al abrirlo en el navegador, todas las visualizaciones cargan correctamente y la interactividad funciona. El informe está conectado al modelo `SM_Ventas_DirectLake` en modo Direct Lake.

**Verificación:**

✅ El ítem `Informe_Ventas_DirectLake` aparece en el Workspace con tipo "Report".  
✅ El informe abre correctamente en el navegador sin errores de conexión.  
✅ Al hacer clic derecho sobre una categoría en el gráfico de barras, aparece la opción "Drill through → Detalle por Producto".  
✅ Los segmentadores de Año y Región filtran todas las visualizaciones de la página simultáneamente.

---

## Validación y Pruebas Finales

Completa las siguientes pruebas para confirmar que el laboratorio está correctamente implementado:

### Lista de Verificación Final

| # | Prueba | Criterio de Éxito | Estado |
|---|---|---|---|
| 1 | Modelo semántico Direct Lake creado | `SM_Ventas_DirectLake` visible en el Workspace con modo Direct Lake | ☐ |
| 2 | Esquema estrella con 4 relaciones | 4 relaciones activas (*:1) en el diagrama del modelo | ☐ |
| 3 | 10 medidas DAX en tabla `_Medidas` | Todas las medidas sin errores, formatos correctos | ☐ |
| 4 | 3 jerarquías definidas | Temporal, Geográfica y Producto con niveles correctos | ☐ |
| 5 | `dim_fecha` marcada como tabla de fechas | Sin errores de validación, funciones de inteligencia de tiempo activas | ☐ |
| 6 | Conexión Live desde Power BI Desktop | Barra de estado muestra "Live connection to: SM_Ventas_DirectLake" | ☐ |
| 7 | Informe con 5+ visualizaciones | 3 tarjetas + barras + líneas + mapa + tabla drill-through | ☐ |
| 8 | Drill-down temporal funcional | Navegación Año → Trimestre → Mes en el gráfico de líneas | ☐ |
| 9 | Drill-through funcional | Navegación desde "Resumen Ejecutivo" a "Detalle por Producto" | ☐ |
| 10 | Informe publicado en Fabric | Ítem visible en Workspace, abre en navegador sin errores | ☐ |

### Prueba de Rendimiento Final

Ejecuta esta consulta DAX en el editor de modelo en Fabric para verificar que el modelo responde correctamente con datos reales:

```dax
EVALUATE
ADDCOLUMNS(
    SUMMARIZECOLUMNS(
        silver_productos[categoria],
        dim_fecha[anio],
        "Total Ventas", [Total Ventas],
        "Ventas Año Anterior", [Ventas Año Anterior],
        "Variación %", [Variación % vs Año Anterior],
        "Margen %", [Margen %]
    ),
    "Ranking Ventas", RANKX(
        ALL(silver_productos[categoria]),
        [Total Ventas]
    )
)
ORDER BY [Total Ventas] DESC
```

**Resultado esperado:** La consulta debe retornar datos con valores numéricos coherentes. Las columnas "Ventas Año Anterior" pueden mostrar BLANK() para el primer año del dataset (comportamiento correcto de `SAMEPERIODLASTYEAR`).

---

## Resolución de Problemas

### Problema 1: El modelo semántico no muestra el modo "Direct Lake" — aparece como "Import" o "DirectQuery"

**Síntomas:**
- Al crear el modelo semántico desde el Lakehouse, la barra de estado o las propiedades del modelo muestran el modo "Import" o "DirectQuery" en lugar de "Direct Lake".
- Las opciones de actualización programada están habilitadas como si fuera un modelo Import normal.
- Al conectarse desde Power BI Desktop, los datos se importan localmente.

**Causa:**
Esto ocurre por una de las siguientes razones:
1. Las tablas seleccionadas no son tablas Delta registradas en el metastore del Lakehouse — son archivos en la carpeta `Files` en lugar de la sección `Tables`.
2. La capacidad Fabric Trial expiró o no tiene suficientes recursos (Direct Lake requiere F2 o superior).
3. Se creó el modelo semántico desde Power BI Desktop usando "Obtener datos → Azure Data Lake Storage" en lugar de la opción de Semantic Model de Fabric.

**Solución:**

```python
# Paso 1: Verificar que las tablas están en el metastore (no solo en Files)
# Ejecutar en un Notebook del Workspace

tablas_en_metastore = spark.catalog.listTables()
print("Tablas registradas en el metastore:")
for tabla in tablas_en_metastore:
    print(f"  - {tabla.name} | tipo: {tabla.tableType} | base de datos: {tabla.database}")

# Si alguna tabla no aparece, registrarla:
spark.sql("""
    CREATE TABLE IF NOT EXISTS silver_ventas
    USING DELTA
    LOCATION 'Tables/silver_ventas'
""")
```

```python
# Paso 2: Verificar que la tabla está en formato Delta (no Parquet sin metadatos)
from delta.tables import DeltaTable

try:
    dt = DeltaTable.forName(spark, "silver_ventas")
    print("✅ silver_ventas es una tabla Delta válida")
    dt.history(3).show()
except Exception as e:
    print(f"❌ Error: {e}")
    print("La tabla no es una tabla Delta válida. Recréala con Lab 03.")
```

Después de verificar y corregir el registro de tablas, elimina el modelo semántico `SM_Ventas_DirectLake` del Workspace y vuelve a crearlo desde el paso 2 de este laboratorio. El modo Direct Lake se asignará automáticamente cuando las tablas sean Delta válidas en el metastore del Lakehouse y la capacidad Fabric esté activa.

---

### Problema 2: Las funciones de inteligencia de tiempo DAX (TOTALYTD, SAMEPERIODLASTYEAR) retornan BLANK() o errores

**Síntomas:**
- Las medidas `Ventas YTD` y `Ventas Año Anterior` muestran BLANK() en todas las celdas del informe.
- Al escribir la medida en el editor DAX, aparece el error: *"La función TOTALYTD requiere una tabla de fechas marcada"* o *"No se puede usar la función de inteligencia de tiempo porque no hay una tabla de fechas"*.
- El gráfico de líneas muestra solo la serie "Total Ventas" pero no "Ventas Año Anterior".

**Causa:**
Existen dos causas posibles:
1. La tabla `dim_fecha` no fue marcada como tabla de fechas en el modelo semántico (Paso 5 de este laboratorio).
2. La columna `fecha` en `dim_fecha` tiene huecos (fechas faltantes) o valores nulos, lo que impide la validación de la tabla de fechas.

**Solución:**

```python
# Verificar la integridad de dim_fecha en un Notebook

from pyspark.sql.functions import col, count, min, max, datediff, sequence, explode
from pyspark.sql.types import DateType

df_fecha = spark.read.format("delta").load("Tables/dim_fecha")

# Verificar rango y huecos
fecha_min = df_fecha.agg({"fecha": "min"}).collect()[0][0]
fecha_max = df_fecha.agg({"fecha": "max"}).collect()[0][0]
total_registros = df_fecha.count()

print(f"Fecha mínima: {fecha_min}")
print(f"Fecha máxima: {fecha_max}")
print(f"Total registros: {total_registros}")

# Calcular días esperados (sin huecos)
dias_esperados = (fecha_max - fecha_min).days + 1
print(f"Días esperados (sin huecos): {dias_esperados}")

if total_registros == dias_esperados:
    print("✅ dim_fecha está completa, sin huecos")
else:
    print(f"❌ Hay {dias_esperados - total_registros} fechas faltantes")
    
    # Regenerar dim_fecha completa
    from pyspark.sql.functions import to_date, lit, year, month, dayofmonth, quarter, weekofyear, date_format
    
    fechas_completas = spark.sql(f"""
        SELECT explode(sequence(
            to_date('{fecha_min}'), 
            to_date('{fecha_max}'), 
            interval 1 day
        )) AS fecha
    """)
    
    dim_fecha_corregida = fechas_completas \
        .withColumn("anio", year("fecha")) \
        .withColumn("trimestre", quarter("fecha")) \
        .withColumn("mes", month("fecha")) \
        .withColumn("dia", dayofmonth("fecha")) \
        .withColumn("numero_mes", month("fecha")) \
        .withColumn("nombre_mes", date_format("fecha", "MMMM")) \
        .withColumn("semana", weekofyear("fecha")) \
        .withColumn("nombre_trimestre", 
                    (quarter("fecha").cast("string").concat(lit("T").concat(year("fecha").cast("string")))))
    
    dim_fecha_corregida.write \
        .format("delta") \
        .mode("overwrite") \
        .option("overwriteSchema", "true") \
        .saveAsTable("dim_fecha")
    
    print("✅ dim_fecha regenerada sin huecos")
    print("Regresa al editor del modelo semántico y actualiza la tabla dim_fecha,")
    print("luego vuelve a marcarla como tabla de fechas.")
```

Después de corregir `dim_fecha`, regresa al editor del modelo semántico en Fabric, selecciona la tabla `dim_fecha`, y repite el procedimiento de "Marcar como tabla de fechas" del Paso 5. Las medidas de inteligencia de tiempo deberían funcionar correctamente después de este proceso.

---

## Limpieza de Recursos

> ⚠️ **NO elimines ningún recurso al finalizar este laboratorio.** El informe publicado y el modelo semántico `SM_Ventas_DirectLake` son requeridos por el **Lab 05** (Configuración de Alertas Automáticas con Data Activator), que usará las métricas del modelo como fuente de eventos para las alertas.

### Acciones de Limpieza Permitidas en este Punto

| Acción | ¿Permitida? | Motivo |
|---|---|---|
| Eliminar el archivo `.pbix` local | ✅ Sí | El informe ya está publicado en Fabric |
| Cerrar Power BI Desktop | ✅ Sí | No afecta el modelo publicado |
| Cerrar el editor de modelo semántico en el navegador | ✅ Sí | Los cambios están guardados |
| Eliminar `SM_Ventas_DirectLake` del Workspace | ❌ No | Requerido por Lab 05 |
| Eliminar `Informe_Ventas_DirectLake` del Workspace | ❌ No | Requerido por Lab 05 |
| Eliminar tablas Silver del Lakehouse | ❌ No | Base de todo el modelo |

### Limpieza Final (Solo después de completar TODOS los laboratorios del curso)

Después de completar el Lab 05, el instructor proporcionará instrucciones para la limpieza completa del entorno. La limpieza final consiste en eliminar el **Workspace completo** del curso, lo que eliminará automáticamente todos los artefactos (Lakehouse, Pipelines, Notebooks, Modelos Semánticos, Informes) y liberará la capacidad Trial.

```
-- Instrucción de limpieza final (ejecutar SOLO después del Lab 05) --
Portal de Fabric → Workspace Settings → Eliminar este Workspace
Esto liberará la capacidad Trial y eliminará todos los recursos del curso.
```

---

## Resumen

### Lo que Construiste en este Laboratorio

En este laboratorio completaste la capa de consumo analítico de tu arquitectura Medallion implementando un modelo semántico Direct Lake de extremo a extremo:

1. **Verificaste el SQL Endpoint** del Lakehouse y confirmaste que las tablas Silver están accesibles tanto para Direct Lake como para el mecanismo de fallback a DirectQuery.

2. **Creaste un modelo semántico personalizado** (`SM_Ventas_DirectLake`) en Microsoft Fabric con modo Direct Lake, seleccionando las 5 tablas Delta de la capa Silver.

3. **Definiste el esquema estrella** con 4 relaciones (*:1) entre la tabla de hechos `silver_ventas` y las dimensiones de clientes, productos, tiendas y fechas.

4. **Creaste 10 medidas DAX** de negocio incluyendo totales, márgenes, variaciones temporales (YTD, vs año anterior) y KPIs, organizadas en una tabla dedicada `_Medidas`.

5. **Configuraste jerarquías** de tiempo, geografía y producto para habilitar el drill-down en los informes, y marcaste `dim_fecha` como tabla de fechas para las funciones de inteligencia de tiempo.

6. **Conectaste Power BI Desktop** al modelo publicado mediante Live Connection, sin importar ni duplicar datos.

7. **Construiste un informe interactivo** con 6+ visualizaciones (tarjetas KPI, gráfico de barras, líneas de tendencia, mapa geográfico y tabla con drill-through) y lo publicaste en el Workspace de Fabric.

8. **Analizaste el rendimiento** con el Performance Analyzer, comprendiendo el mecanismo de framing de Direct Lake y la diferencia entre la primera carga (framing desde OneLake) y las consultas en caché.

### Conceptos Clave Reforzados

| Concepto | Descripción |
|---|---|
| **Direct Lake** | Lee Parquet desde OneLake directamente; sin copia, sin actualización programada, velocidad comparable a Import |
| **Framing** | Carga de columnas bajo demanda desde OneLake al caché en memoria — ocurre en la primera consulta |
| **Fallback a DirectQuery** | Mecanismo de seguridad cuando Direct Lake no puede resolver la consulta desde caché |
| **Live Connection** | Conexión de Power BI Desktop que delega todas las consultas al modelo en Fabric; sin datos locales |
| **Tabla de fechas marcada** | Requisito para funciones de inteligencia de tiempo DAX (TOTALYTD, SAMEPERIODLASTYEAR) |
| **Tabla `_Medidas`** | Buena práctica: centralizar medidas DAX en una tabla dedicada para mejor organización |

### Próximos Pasos

Con el modelo semántico Direct Lake publicado y el informe de Power BI operativo, el **Lab 05** completará la arquitectura implementando alertas automáticas con **Data Activator** de Microsoft Fabric. Configurarás reglas basadas en los datos del modelo semántico — por ejemplo, una alerta cuando las ventas caigan por debajo de un umbral o cuando el margen supere un objetivo — conectando así la capa analítica con operaciones proactivas en tiempo real.

### Recursos Adicionales

| Recurso | URL |
|---|---|
| Documentación oficial Direct Lake | [learn.microsoft.com/es-es/power-bi/enterprise/directlake-overview](https://learn.microsoft.com/es-es/power-bi/enterprise/directlake-overview) |
| Guía de optimización de modelos semánticos Direct Lake | [learn.microsoft.com/es-es/power-bi/enterprise/directlake-analyze-qso](https://learn.microsoft.com/es-es/power-bi/enterprise/directlake-analyze-qso) |
| Referencia DAX — TOTALYTD | [dax.guide/totalytd](https://dax.guide/totalytd) |
| Referencia DAX — SAMEPERIODLASTYEAR | [dax.guide/sameperiodlastyear](https://dax.guide/sameperiodlastyear) |
| Comparación de modos Import, DirectQuery y Direct Lake | [learn.microsoft.com/es-es/power-bi/connect-data/service-dataset-modes-understand](https://learn.microsoft.com/es-es/power-bi/connect-data/service-dataset-modes-understand) |
| Performance Analyzer en Power BI Desktop | [learn.microsoft.com/es-es/power-bi/create-reports/desktop-performance-analyzer](https://learn.microsoft.com/es-es/power-bi/create-reports/desktop-performance-analyzer) |

---
*Lab 04-00-01 — Versión 1.0 | Curso: Microsoft Fabric Lakehouse & Analytics Engineering*
