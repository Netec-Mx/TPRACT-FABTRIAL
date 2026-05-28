# Ingeniería de datos: Transformación Bronze → Silver con Notebook guiado

---

## 1. Metadatos

| Atributo | Detalle |
|---|---|
| **Duración estimada** | 105 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear (Create) |
| **Laboratorio anterior requerido** | Lab 02-00-01 |
| **Tecnologías principales** | Microsoft Fabric Notebooks, PySpark, Delta Lake, Data Factory Pipeline |

---

## 2. Descripción General

En este laboratorio implementarás el núcleo de ingeniería de datos del curso: la transformación de datos desde la capa **Bronze** hacia la capa **Silver** siguiendo la arquitectura Medallion dentro de Microsoft Fabric. Crearás un Notebook de Spark que leerá las tablas Delta de Bronze, aplicará un conjunto completo de transformaciones de calidad (limpieza de nulos, deduplicación, casting de tipos, normalización de texto y validaciones de negocio), construirá una tabla enriquecida mediante joins y escribirá el resultado en la capa Silver usando el formato Delta con operación MERGE. Al finalizar, integrarás el Notebook como una actividad dentro del Pipeline de Data Factory creado en el Lab 2, completando el flujo automatizado end-to-end Bronze → Silver.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] **Crear y configurar** un Notebook de Microsoft Fabric conectado al Lakehouse del curso como almacenamiento por defecto para acceder a las tablas Delta de la capa Bronze.
- [ ] **Implementar** transformaciones de calidad de datos en PySpark incluyendo deduplicación, manejo de nulos, casting de tipos, estandarización de strings y validaciones de reglas de negocio con aserciones.
- [ ] **Construir** tablas Delta en la capa Silver con esquema explícito, particionamiento por fecha y escritura incremental mediante la operación MERGE (upsert) de Delta Lake.
- [ ] **Orquestar** la ejecución del Notebook desde un Pipeline de Data Factory añadiendo una actividad Notebook que conecte el flujo automatizado Bronze → Silver.

---

## 4. Prerrequisitos

### Conocimiento previo requerido

- Haber completado **Lab 01-00-01** (Lakehouse creado con capas Bronze y Silver configuradas).
- Haber completado **Lab 02-00-01** (Pipeline de ingesta ejecutado correctamente; datos disponibles en la tabla `bronze_ventas` y archivos relacionados en la capa Bronze del Lakehouse).
- Conocimientos básicos de Python y sintaxis de PySpark: creación de DataFrames, uso de transformaciones como `filter`, `withColumn`, `join`.
- Comprensión de los conceptos de calidad de datos: valores nulos, registros duplicados, inconsistencia de tipos.
- Familiaridad con Delta Lake: transacciones ACID, versionado de tablas y operaciones de escritura (`overwrite`, `append`, `merge`).
- Conocimiento de SQL intermedio para validaciones con Spark SQL.

### Acceso requerido

- Cuenta de Microsoft Fabric con capacidad Trial activa (60 días).
- Acceso al Workspace del curso creado en Lab 01.
- Lakehouse del curso con datos en la capa Bronze (tablas: `bronze_ventas`, `bronze_productos`, `bronze_clientes`).
- Permisos de **Contributor** o superior en el Workspace de Fabric.

> ⚠️ **ADVERTENCIA CRÍTICA — Dependencia secuencial:** Este laboratorio requiere que las tablas `bronze_ventas`, `bronze_productos` y `bronze_clientes` existan en el Lakehouse. Si el Lab 02 no se completó correctamente, detente y corrígelo antes de continuar. Ejecutar este lab sin los datos de Bronze producirá errores en cada celda de lectura.

---

## 5. Entorno del Laboratorio

### Requisitos de hardware

| Componente | Mínimo | Recomendado |
|---|---|---|
| RAM | 8 GB | 16 GB |
| Procesador | Intel Core i5 8ª gen / AMD Ryzen 5 | Intel Core i7 / AMD Ryzen 7 |
| Almacenamiento libre | 2 GB | 5 GB |
| Conexión a Internet | 10 Mbps | 25 Mbps |
| Resolución de pantalla | 1366×768 | 1920×1080 |

### Requisitos de software

| Software | Versión mínima | Notas |
|---|---|---|
| Microsoft Edge o Google Chrome | 110 o superior | Navegador principal para Fabric |
| Microsoft Fabric (Trial) | Trial activo | Portal: `app.fabric.microsoft.com` |
| Power BI Desktop | Mayo 2024 o superior | Requerido en Lab 04; no en este lab |

### Nota sobre tiempos de Spark

> ⏱️ **Gestión de tiempos de cómputo:** En capacidades Trial, la inicialización del clúster Spark puede tomar entre **3 y 8 minutos** en la primera ejecución de celda. Las ejecuciones posteriores dentro de la misma sesión serán más rápidas. Planifica un buffer de tiempo adicional para las secciones 6.3 y 6.5 de este laboratorio. **No cierres el navegador ni el Notebook durante la inicialización.**

### Verificación del entorno antes de comenzar

Antes de iniciar el laboratorio, verifica que el entorno esté listo ejecutando los siguientes pasos de comprobación:

1. Navega a `https://app.fabric.microsoft.com` e inicia sesión.
2. Confirma que el Workspace del curso aparece en el panel izquierdo.
3. Abre el Lakehouse del curso y verifica que en la sección **Tables** existen las tablas: `bronze_ventas`, `bronze_productos` y `bronze_clientes`.
4. Si alguna tabla falta, regresa al Lab 02 y ejecuta el Pipeline de ingesta antes de continuar.

---

## 6. Procedimiento Paso a Paso

---

### Paso 1: Crear y Configurar el Notebook de Fabric

**Objetivo:** Crear un nuevo Notebook en Microsoft Fabric, conectarlo al Lakehouse del curso como almacenamiento por defecto y verificar que el entorno de Spark puede leer las tablas Bronze.

---

#### Instrucciones

1. Navega al portal de Microsoft Fabric en `https://app.fabric.microsoft.com` e inicia sesión con tus credenciales del curso.

2. En el panel de navegación izquierdo, selecciona el **Workspace del curso** (el mismo utilizado en Labs 01 y 02).

3. En la barra superior del Workspace, haz clic en el botón **+ Nuevo** y selecciona **Notebook** en el menú desplegable. Si no aparece directamente, busca la opción dentro de **Más opciones de creación → Ingeniería de datos → Notebook**.

4. El Notebook se abrirá con un nombre por defecto (por ejemplo, `Notebook 1`). Haz clic en el nombre en la parte superior izquierda y renómbralo a:
   ```
   NB_Bronze_Silver_Ventas
   ```

5. **Conectar el Lakehouse al Notebook:** En el panel izquierdo del Notebook (sección **Lakehouse**), haz clic en **Agregar Lakehouse** (o el ícono de base de datos con signo `+`).
   - En el diálogo que aparece, selecciona **Lakehouse existente**.
   - Busca y selecciona el **Lakehouse del curso** (el creado en Lab 01, por ejemplo `LH_Ventas_Curso`).
   - Haz clic en **Agregar**.

6. Verifica que el Lakehouse aparece ahora en el panel izquierdo bajo **Lakehouses**. Expande la sección **Tables** y confirma que ves las tablas `bronze_ventas`, `bronze_productos` y `bronze_clientes`.

7. En la primera celda del Notebook (tipo **Code**), escribe el siguiente código para verificar la conexión y el entorno de Spark:

   ```python
   # Celda 1: Verificación del entorno y la sesión de Spark
   print(f"Versión de Spark: {spark.version}")
   print(f"Nombre de la aplicación: {spark.sparkContext.appName}")
   
   # Listar las tablas disponibles en el Lakehouse por defecto
   tablas_disponibles = spark.sql("SHOW TABLES").collect()
   print("\nTablas disponibles en el Lakehouse:")
   for tabla in tablas_disponibles:
       print(f"  - {tabla['tableName']}")
   ```

8. Ejecuta la celda haciendo clic en el botón **▶ Ejecutar celda** (o presiona `Shift + Enter`).

   > ⏱️ Si es la primera ejecución de la sesión, espera entre 3 y 8 minutos mientras el clúster Spark se inicializa. Verás un indicador de progreso en la parte inferior del Notebook.

---

**Salida esperada:**

```
Versión de Spark: 3.4.x (o superior)
Nombre de la aplicación: Microsoft Fabric Notebook

Tablas disponibles en el Lakehouse:
  - bronze_ventas
  - bronze_productos
  - bronze_clientes
```

---

**Verificación:**

- ✅ El Notebook muestra la versión de Spark sin errores.
- ✅ Las tres tablas Bronze aparecen en la lista de tablas disponibles.
- ✅ El panel izquierdo del Notebook muestra el Lakehouse conectado con sus tablas expandidas.

---

### Paso 2: Documentar el Notebook con Celdas Markdown

**Objetivo:** Agregar celdas de documentación en Markdown para estructurar el Notebook de forma profesional y reproducible, siguiendo buenas prácticas de ingeniería de datos.

---

#### Instrucciones

1. Haz clic en el botón **+ Código** debajo de la primera celda y luego cambia el tipo de celda a **Markdown** usando el selector de tipo de celda (generalmente un menú desplegable que dice `Code`; cámbialo a `Markdown`).

2. Escribe el siguiente encabezado de documentación en la celda Markdown:

   ```markdown
   # Notebook: Transformación Bronze → Silver — Ventas
   
   ## Descripción
   Este Notebook implementa el proceso de transformación de datos desde la capa **Bronze** 
   hacia la capa **Silver** dentro de la arquitectura Medallion del curso.
   
   ## Flujo de procesamiento
   1. Lectura de tablas Delta desde Bronze (`bronze_ventas`, `bronze_productos`, `bronze_clientes`)
   2. Limpieza y transformaciones de calidad (deduplicación, nulos, casting, normalización)
   3. Validaciones de reglas de negocio
   4. Join enriquecido entre tablas
   5. Escritura en Silver con operación MERGE (upsert)
   
   ## Autor
   Estudiante del Curso — Microsoft Fabric Lakehouse Engineering
   
   ## Versión
   v1.0 — Lab 03-00-01
   ```

3. Haz clic fuera de la celda Markdown o presiona `Escape` para renderizarla. Verifica que el texto aparece formateado correctamente.

4. Agrega una segunda celda Markdown debajo de la primera celda de código (verificación del entorno) con el siguiente contenido:

   ```markdown
   ## Paso 1: Importación de librerías y configuración global
   
   Importamos las funciones de PySpark necesarias para las transformaciones y definimos 
   las constantes de configuración del pipeline.
   ```

5. Agrega una nueva celda de código debajo de esa Markdown con las importaciones globales del Notebook:

   ```python
   # Celda 2: Importaciones y configuración global
   from pyspark.sql import functions as F
   from pyspark.sql.types import (
       DoubleType, DateType, IntegerType, StringType, TimestampType
   )
   from delta.tables import DeltaTable
   
   # Constantes de configuración
   LAKEHOUSE_NAME = "LH_Ventas_Curso"  # Ajustar si el nombre difiere
   BRONZE_PREFIX = "bronze_"
   SILVER_PREFIX = "silver_"
   FECHA_FORMATO = "yyyy-MM-dd"
   
   # Columnas clave para deduplicación y merge
   CLAVE_VENTAS = ["id_transaccion"]
   CLAVE_PRODUCTOS = ["id_producto"]
   CLAVE_CLIENTES = ["id_cliente"]
   
   print("✅ Librerías importadas y configuración cargada correctamente.")
   ```

6. Ejecuta la celda de importaciones (`Shift + Enter`). Esta ejecución debe ser casi instantánea ya que el clúster ya está iniciado.

---

**Salida esperada:**

```
✅ Librerías importadas y configuración cargada correctamente.
```

---

**Verificación:**

- ✅ Las celdas Markdown se renderizan con formato correcto (encabezados, negritas, listas).
- ✅ La celda de importaciones ejecuta sin errores `ModuleNotFoundError` ni `ImportError`.
- ✅ La estructura del Notebook es legible y documenta el flujo de trabajo.

---

### Paso 3: Leer y Explorar las Tablas Bronze

**Objetivo:** Leer las tres tablas Delta de la capa Bronze en DataFrames de PySpark, explorar su esquema y calidad inicial para fundamentar las decisiones de transformación.

---

#### Instrucciones

1. Agrega una celda Markdown con el encabezado de la sección:

   ```markdown
   ## Paso 2: Lectura y exploración de tablas Bronze
   
   Leemos las tablas Delta desde la capa Bronze y analizamos su estructura y calidad 
   inicial para identificar problemas que deben corregirse antes de escribir en Silver.
   ```

2. Agrega una celda de código para leer las tres tablas Bronze:

   ```python
   # Celda 3: Lectura de tablas Bronze
   print("Leyendo tablas desde la capa Bronze...")
   
   # Lectura de la tabla principal de ventas
   df_bronze_ventas = spark.read.format("delta").load("Tables/bronze_ventas")
   
   # Lectura de la tabla de productos (dimensión)
   df_bronze_productos = spark.read.format("delta").load("Tables/bronze_productos")
   
   # Lectura de la tabla de clientes (dimensión)
   df_bronze_clientes = spark.read.format("delta").load("Tables/bronze_clientes")
   
   print(f"✅ bronze_ventas    → {df_bronze_ventas.count():,} registros")
   print(f"✅ bronze_productos → {df_bronze_productos.count():,} registros")
   print(f"✅ bronze_clientes  → {df_bronze_clientes.count():,} registros")
   ```

3. Ejecuta la celda. Anota los conteos de registros en tu cuaderno de trabajo — los usarás para validar las transformaciones.

4. Agrega una nueva celda de código para explorar el esquema y detectar problemas de calidad:

   ```python
   # Celda 4: Exploración del esquema y calidad inicial de bronze_ventas
   print("=" * 60)
   print("ESQUEMA DE bronze_ventas:")
   print("=" * 60)
   df_bronze_ventas.printSchema()
   
   print("\nPrimeras 5 filas de bronze_ventas:")
   df_bronze_ventas.show(5, truncate=False)
   
   print("\n--- ANÁLISIS DE CALIDAD INICIAL ---")
   total_registros = df_bronze_ventas.count()
   
   # Contar duplicados por clave de negocio
   total_unicos = df_bronze_ventas.dropDuplicates(CLAVE_VENTAS).count()
   duplicados = total_registros - total_unicos
   
   # Contar nulos en columnas críticas
   nulos_id = df_bronze_ventas.filter(F.col("id_transaccion").isNull()).count()
   nulos_monto = df_bronze_ventas.filter(F.col("monto").isNull()).count()
   nulos_fecha = df_bronze_ventas.filter(F.col("fecha_venta").isNull()).count()
   
   print(f"Total registros        : {total_registros:,}")
   print(f"Registros únicos       : {total_unicos:,}")
   print(f"Duplicados detectados  : {duplicados:,}")
   print(f"Nulos en id_transaccion: {nulos_id:,}")
   print(f"Nulos en monto         : {nulos_monto:,}")
   print(f"Nulos en fecha_venta   : {nulos_fecha:,}")
   ```

5. Ejecuta la celda y revisa cuidadosamente la salida. Identifica los tipos de datos actuales en el esquema (especialmente si `monto` aparece como `StringType` o `DoubleType`, y si `fecha_venta` es `StringType` o `DateType`).

6. Agrega una celda adicional para explorar las tablas de dimensiones:

   ```python
   # Celda 5: Exploración rápida de dimensiones
   print("ESQUEMA DE bronze_productos:")
   df_bronze_productos.printSchema()
   df_bronze_productos.show(3, truncate=False)
   
   print("\nESQUEMA DE bronze_clientes:")
   df_bronze_clientes.printSchema()
   df_bronze_clientes.show(3, truncate=False)
   ```

7. Ejecuta la celda y anota los nombres exactos de las columnas clave en cada tabla (por ejemplo: `id_producto` en productos, `id_cliente` en clientes). Estos nombres se usarán en los joins del Paso 5.

---

**Salida esperada:**

```
✅ bronze_ventas    → X,XXX registros
✅ bronze_productos →   XXX registros
✅ bronze_clientes  →   XXX registros

ESQUEMA DE bronze_ventas:
root
 |-- id_transaccion: string (nullable = true)
 |-- id_producto: string (nullable = true)
 |-- id_cliente: string (nullable = true)
 |-- fecha_venta: string (nullable = true)
 |-- monto: string (nullable = true)
 |-- cantidad: string (nullable = true)
 |-- region: string (nullable = true)
 |-- ...

--- ANÁLISIS DE CALIDAD INICIAL ---
Total registros        : X,XXX
Registros únicos       : X,XXX
Duplicados detectados  : X
Nulos en id_transaccion: X
Nulos en monto         : X
Nulos en fecha_venta   : X
```

> 📝 **Nota pedagógica:** Es normal que los datos Bronze tengan columnas de tipo `string` incluso para campos numéricos o de fecha, ya que fueron ingestados tal como llegaron desde el CSV. Esto es precisamente lo que la capa Silver debe corregir.

---

**Verificación:**

- ✅ Los tres DataFrames se leen sin errores `AnalysisException` ni `FileNotFoundException`.
- ✅ El esquema de `bronze_ventas` muestra las columnas esperadas del dataset del curso.
- ✅ El análisis de calidad muestra métricas (aunque sean cero en algunos casos).

---

### Paso 4: Implementar Transformaciones de Calidad de Datos

**Objetivo:** Aplicar el pipeline completo de transformaciones de calidad sobre `bronze_ventas` incluyendo deduplicación, manejo de nulos, casting de tipos, normalización de texto y validaciones de reglas de negocio.

---

#### Instrucciones

1. Agrega una celda Markdown de sección:

   ```markdown
   ## Paso 3: Transformaciones de calidad — Bronze → Silver
   
   Aplicamos el pipeline de limpieza y tipado sobre los datos de ventas. Cada transformación 
   está documentada con su justificación de negocio.
   ```

2. Agrega la celda principal de transformaciones de calidad:

   ```python
   # Celda 6: Pipeline de transformaciones de calidad sobre ventas
   print("Iniciando transformaciones de calidad sobre bronze_ventas...")
   
   df_silver_ventas = (
       df_bronze_ventas
       
       # --- DEDUPLICACIÓN ---
       # Eliminar registros duplicados basados en la clave de negocio única.
       # Los duplicados pueden ocurrir por reintentos del sistema fuente (ERP).
       .dropDuplicates(CLAVE_VENTAS)
       
       # --- ELIMINACIÓN DE NULOS EN COLUMNAS CRÍTICAS ---
       # Registros sin id_transaccion no pueden ser identificados ni trazados.
       .filter(F.col("id_transaccion").isNotNull())
       # Registros sin fecha_venta no pueden asignarse a ningún período de negocio.
       .filter(F.col("fecha_venta").isNotNull())
       
       # --- CASTING DE TIPOS DE DATOS ---
       # Convertir monto de string a double para cálculos numéricos.
       .withColumn("monto", F.col("monto").cast(DoubleType()))
       # Convertir cantidad de string a integer.
       .withColumn("cantidad", F.col("cantidad").cast(IntegerType()))
       # Convertir fecha_venta de string a DateType con el formato del dataset.
       .withColumn("fecha_venta", F.to_date(F.col("fecha_venta"), FECHA_FORMATO))
       
       # --- MANEJO DE NULOS POST-CASTING ---
       # Si el casting falla (valor no convertible), el resultado es null.
       # Rellenamos monto nulo con 0.0 y marcamos para revisión.
       .withColumn("monto",
           F.when(F.col("monto").isNull(), F.lit(0.0))
            .otherwise(F.col("monto"))
       )
       # Rellenamos cantidad nula con 1 (mínimo lógico de negocio).
       .withColumn("cantidad",
           F.when(F.col("cantidad").isNull(), F.lit(1))
            .otherwise(F.col("cantidad"))
       )
       
       # --- NORMALIZACIÓN DE TEXTO ---
       # Eliminar espacios al inicio/fin y convertir a mayúsculas para consistencia.
       .withColumn("region", F.trim(F.upper(F.col("region"))))
       .withColumn("id_producto", F.trim(F.col("id_producto")))
       .withColumn("id_cliente", F.trim(F.col("id_cliente")))
       
       # --- VALIDACIÓN DE REGLAS DE NEGOCIO ---
       # Regla 1: El monto de una venta debe ser mayor o igual a cero.
       .filter(F.col("monto") >= 0)
       # Regla 2: La cantidad vendida debe ser mayor que cero.
       .filter(F.col("cantidad") > 0)
       # Regla 3: La fecha de venta no puede ser futura (más de 1 día de tolerancia).
       .filter(F.col("fecha_venta") <= F.date_add(F.current_date(), 1))
       # Regla 4: La fecha de venta no puede ser anterior al año 2000.
       .filter(F.col("fecha_venta") >= F.lit("2000-01-01").cast(DateType()))
       
       # --- COLUMNAS DE AUDITORÍA ---
       # Registrar cuándo fue procesado este registro para trazabilidad.
       .withColumn("fecha_procesamiento_silver", F.current_timestamp())
       # Indicar el origen del dato para linaje.
       .withColumn("origen_capa", F.lit("bronze_ventas"))
   )
   
   registros_silver = df_silver_ventas.count()
   registros_bronze_original = df_bronze_ventas.count()
   registros_eliminados = registros_bronze_original - registros_silver
   
   print(f"Registros en Bronze (original) : {registros_bronze_original:,}")
   print(f"Registros en Silver (limpios)  : {registros_silver:,}")
   print(f"Registros eliminados/filtrados : {registros_eliminados:,}")
   print(f"Tasa de retención              : {(registros_silver/registros_bronze_original)*100:.1f}%")
   print("\n✅ Transformaciones de calidad aplicadas correctamente.")
   ```

3. Ejecuta la celda. Espera a que complete (puede tomar 1-3 minutos por las múltiples transformaciones encadenadas).

4. Agrega una celda de validación de calidad con aserciones. Esta celda actúa como "guardián" que impide escribir datos corruptos en Silver:

   ```python
   # Celda 7: Validaciones de calidad con aserciones — GUARDIANES DE SILVER
   print("Ejecutando validaciones de calidad antes de escribir en Silver...")
   print("=" * 60)
   
   errores_encontrados = []
   
   # Validación 1: No deben existir valores nulos en id_transaccion
   nulos_id = df_silver_ventas.filter(F.col("id_transaccion").isNull()).count()
   if nulos_id > 0:
       errores_encontrados.append(f"id_transaccion tiene {nulos_id} valores nulos")
   
   # Validación 2: No deben existir valores nulos en fecha_venta
   nulos_fecha = df_silver_ventas.filter(F.col("fecha_venta").isNull()).count()
   if nulos_fecha > 0:
       errores_encontrados.append(f"fecha_venta tiene {nulos_fecha} valores nulos")
   
   # Validación 3: El monto no puede ser negativo
   montos_negativos = df_silver_ventas.filter(F.col("monto") < 0).count()
   if montos_negativos > 0:
       errores_encontrados.append(f"monto tiene {montos_negativos} valores negativos")
   
   # Validación 4: No deben existir duplicados en la clave de negocio
   total = df_silver_ventas.count()
   unicos = df_silver_ventas.dropDuplicates(CLAVE_VENTAS).count()
   if total != unicos:
       errores_encontrados.append(f"Existen {total - unicos} registros duplicados en id_transaccion")
   
   # Validación 5: La tasa de retención no debe caer por debajo del 85%
   tasa_retencion = (df_silver_ventas.count() / df_bronze_ventas.count()) * 100
   if tasa_retencion < 85:
       errores_encontrados.append(
           f"Tasa de retención ({tasa_retencion:.1f}%) por debajo del umbral mínimo (85%)"
       )
   
   # Evaluación final
   if errores_encontrados:
       print("❌ VALIDACIÓN FALLIDA — No se escribirá en Silver.")
       for error in errores_encontrados:
           print(f"   → ERROR: {error}")
       raise ValueError(
           f"Se encontraron {len(errores_encontrados)} error(es) de calidad. "
           "Revisa las transformaciones antes de continuar."
       )
   else:
       print("✅ Todas las validaciones de calidad superadas.")
       print(f"   → Sin nulos en columnas críticas")
       print(f"   → Sin valores negativos en monto")
       print(f"   → Sin duplicados en id_transaccion")
       print(f"   → Tasa de retención: {tasa_retencion:.1f}% (umbral: 85%)")
       print("\n🚀 Datos aprobados para escritura en Silver.")
   ```

5. Ejecuta la celda de validaciones. Si alguna aserción falla, lee el mensaje de error, revisa la celda de transformaciones (Celda 6) y corrige el problema antes de continuar.

---

**Salida esperada:**

```
Ejecutando validaciones de calidad antes de escribir en Silver...
============================================================
✅ Todas las validaciones de calidad superadas.
   → Sin nulos en columnas críticas
   → Sin valores negativos en monto
   → Sin duplicados en id_transaccion
   → Tasa de retención: XX.X% (umbral: 85%)

🚀 Datos aprobados para escritura en Silver.
```

---

**Verificación:**

- ✅ La celda de transformaciones ejecuta sin `AnalysisException` ni `TypeError`.
- ✅ La celda de validaciones no lanza `ValueError` (todas las aserciones pasan).
- ✅ La tasa de retención es superior al 85% (indica que las reglas de negocio son razonables).

---

### Paso 5: Construir la Tabla Enriquecida con Joins

**Objetivo:** Enriquecer el DataFrame de ventas Silver con información de productos y clientes mediante joins, creando una tabla analítica desnormalizada optimizada para consultas.

---

#### Instrucciones

1. Agrega una celda Markdown:

   ```markdown
   ## Paso 4: Enriquecimiento — Join con dimensiones de Productos y Clientes
   
   Combinamos la tabla de hechos de ventas con las dimensiones de productos y clientes 
   para crear una tabla enriquecida que elimine la necesidad de joins en tiempo de consulta.
   ```

2. Agrega la celda de transformaciones para las dimensiones:

   ```python
   # Celda 8: Limpieza de dimensiones (productos y clientes)
   
   # Limpieza de la tabla de productos
   df_silver_productos = (
       df_bronze_productos
       .dropDuplicates(CLAVE_PRODUCTOS)
       .filter(F.col("id_producto").isNotNull())
       .withColumn("nombre_producto", F.trim(F.col("nombre_producto")))
       .withColumn("categoria", F.trim(F.upper(F.col("categoria"))))
       .withColumn("precio_unitario", F.col("precio_unitario").cast(DoubleType()))
   )
   
   # Limpieza de la tabla de clientes
   df_silver_clientes = (
       df_bronze_clientes
       .dropDuplicates(CLAVE_CLIENTES)
       .filter(F.col("id_cliente").isNotNull())
       .withColumn("nombre_cliente", F.trim(F.col("nombre_cliente")))
       .withColumn("segmento", F.trim(F.upper(F.col("segmento"))))
       .withColumn("pais", F.trim(F.upper(F.col("pais"))))
   )
   
   print(f"✅ Productos limpios: {df_silver_productos.count():,} registros")
   print(f"✅ Clientes limpios : {df_silver_clientes.count():,} registros")
   ```

3. Agrega la celda de join enriquecido:

   ```python
   # Celda 9: Join enriquecido — Tabla Silver analítica
   
   # Seleccionar columnas relevantes de cada dimensión para el join
   cols_productos = ["id_producto", "nombre_producto", "categoria", "precio_unitario"]
   cols_clientes  = ["id_cliente", "nombre_cliente", "segmento", "pais"]
   
   df_silver_enriquecido = (
       df_silver_ventas
       # Join con productos (left join para preservar ventas aunque el producto no esté en la dimensión)
       .join(
           df_silver_productos.select(cols_productos),
           on="id_producto",
           how="left"
       )
       # Join con clientes (left join por la misma razón)
       .join(
           df_silver_clientes.select(cols_clientes),
           on="id_cliente",
           how="left"
       )
       # Columna calculada: ingreso total de la transacción
       .withColumn("ingreso_total", F.round(F.col("monto") * F.col("cantidad"), 2))
       # Columna de particionamiento: año-mes para optimizar consultas por período
       .withColumn("anio_venta", F.year(F.col("fecha_venta")))
       .withColumn("mes_venta", F.month(F.col("fecha_venta")))
   )
   
   print(f"✅ Tabla Silver enriquecida: {df_silver_enriquecido.count():,} registros")
   print(f"\nEsquema final de la tabla Silver enriquecida:")
   df_silver_enriquecido.printSchema()
   
   # Muestra de los primeros registros enriquecidos
   print("\nMuestra de datos enriquecidos:")
   df_silver_enriquecido.select(
       "id_transaccion", "fecha_venta", "nombre_producto", 
       "nombre_cliente", "monto", "cantidad", "ingreso_total", "region"
   ).show(5, truncate=False)
   ```

4. Ejecuta ambas celdas y verifica que el join produce el número de registros esperado (debe ser igual al conteo de `df_silver_ventas` ya que usamos `left join`).

---

**Salida esperada:**

```
✅ Productos limpios: XXX registros
✅ Clientes limpios : XXX registros
✅ Tabla Silver enriquecida: X,XXX registros

Esquema final de la tabla Silver enriquecida:
root
 |-- id_transaccion: string (nullable = true)
 |-- id_producto: string (nullable = true)
 |-- id_cliente: string (nullable = true)
 |-- fecha_venta: date (nullable = true)
 |-- monto: double (nullable = true)
 |-- cantidad: integer (nullable = true)
 |-- region: string (nullable = true)
 |-- nombre_producto: string (nullable = true)
 |-- categoria: string (nullable = true)
 |-- precio_unitario: double (nullable = true)
 |-- nombre_cliente: string (nullable = true)
 |-- segmento: string (nullable = true)
 |-- pais: string (nullable = true)
 |-- ingreso_total: double (nullable = true)
 |-- anio_venta: integer (nullable = true)
 |-- mes_venta: integer (nullable = true)
 |-- fecha_procesamiento_silver: timestamp (nullable = true)
 |-- origen_capa: string (nullable = true)
```

---

**Verificación:**

- ✅ El conteo de la tabla enriquecida es igual al conteo de `df_silver_ventas` (left join no elimina filas).
- ✅ Las columnas `nombre_producto`, `nombre_cliente` e `ingreso_total` aparecen en el esquema.
- ✅ La columna `fecha_venta` es de tipo `DateType`, no `StringType`.

---

### Paso 6: Escribir la Tabla Silver con Operación MERGE (Upsert)

**Objetivo:** Persistir la tabla enriquecida en la capa Silver del Lakehouse usando formato Delta Lake con particionamiento por año/mes y la operación MERGE para soportar ejecuciones incrementales idempotentes.

---

#### Instrucciones

1. Agrega una celda Markdown:

   ```markdown
   ## Paso 5: Escritura en Silver — Delta Lake con MERGE (Upsert)
   
   Escribimos la tabla Silver usando la operación MERGE de Delta Lake. Esta operación 
   es idempotente: si un registro ya existe (mismo `id_transaccion`), se actualiza; 
   si no existe, se inserta. Esto permite ejecuciones incrementales seguras.
   ```

2. Agrega la celda de escritura inicial (primera ejecución — crea la tabla si no existe):

   ```python
   # Celda 10: Escritura inicial en Silver con particionamiento
   # Esta celda maneja tanto la creación inicial como las actualizaciones posteriores.
   
   TABLA_SILVER_NOMBRE = "silver_ventas_enriquecidas"
   TABLA_SILVER_PATH   = f"Tables/{TABLA_SILVER_NOMBRE}"
   
   print(f"Verificando existencia de la tabla Silver: {TABLA_SILVER_NOMBRE}")
   
   # Verificar si la tabla Silver ya existe en el Lakehouse
   tabla_existe = spark.catalog.tableExists(TABLA_SILVER_NOMBRE)
   print(f"Tabla existente: {tabla_existe}")
   
   if not tabla_existe:
       # PRIMERA EJECUCIÓN: Crear la tabla Silver desde cero con particionamiento
       print("Primera ejecución detectada. Creando tabla Silver con particionamiento...")
       (
           df_silver_enriquecido
           .write
           .format("delta")
           .mode("overwrite")
           .option("overwriteSchema", "true")
           .partitionBy("anio_venta", "mes_venta")
           .saveAsTable(TABLA_SILVER_NOMBRE)
       )
       print(f"✅ Tabla '{TABLA_SILVER_NOMBRE}' creada exitosamente en Silver.")
       print(f"   Particionada por: anio_venta, mes_venta")
       print(f"   Registros escritos: {df_silver_enriquecido.count():,}")
   
   else:
       # EJECUCIONES POSTERIORES: Usar MERGE para actualizaciones incrementales
       print("Tabla existente detectada. Ejecutando MERGE (upsert) incremental...")
       
       delta_tabla_silver = DeltaTable.forName(spark, TABLA_SILVER_NOMBRE)
       
       (
           delta_tabla_silver.alias("silver")
           .merge(
               df_silver_enriquecido.alias("nuevos"),
               "silver.id_transaccion = nuevos.id_transaccion"
           )
           # Si el registro existe: actualizar todos los campos
           .whenMatchedUpdateAll()
           # Si el registro no existe: insertar como nuevo
           .whenNotMatchedInsertAll()
           .execute()
       )
       
       # Obtener métricas del MERGE
       historial = delta_tabla_silver.history(1).select(
           "operationMetrics"
       ).collect()[0]["operationMetrics"]
       
       print(f"✅ MERGE completado exitosamente.")
       print(f"   Registros insertados : {historial.get('numTargetRowsInserted', 'N/A')}")
       print(f"   Registros actualizados: {historial.get('numTargetRowsUpdated', 'N/A')}")
   ```

3. Ejecuta la celda. En la primera ejecución del laboratorio, el flujo tomará el camino de creación (`tabla_existe = False`).

4. Agrega una celda para verificar la tabla Silver escrita:

   ```python
   # Celda 11: Verificación de la tabla Silver escrita
   print("Verificando la tabla Silver escrita en el Lakehouse...")
   
   df_verificacion = spark.read.format("delta").load(f"Tables/{TABLA_SILVER_NOMBRE}")
   
   print(f"Registros en silver_ventas_enriquecidas: {df_verificacion.count():,}")
   print(f"\nDistribución por año:")
   df_verificacion.groupBy("anio_venta").count().orderBy("anio_venta").show()
   
   print(f"\nEstadísticas de ingreso_total:")
   df_verificacion.select(
       F.min("ingreso_total").alias("min_ingreso"),
       F.max("ingreso_total").alias("max_ingreso"),
       F.avg("ingreso_total").alias("avg_ingreso"),
       F.sum("ingreso_total").alias("total_ingresos")
   ).show()
   
   # Verificar el historial de versiones Delta
   print("\nHistorial de versiones de la tabla Delta Silver:")
   DeltaTable.forName(spark, TABLA_SILVER_NOMBRE).history().select(
       "version", "timestamp", "operation", "operationParameters"
   ).show(5, truncate=False)
   ```

5. Ejecuta la celda de verificación y confirma que los datos fueron escritos correctamente.

6. Agrega una celda final de resumen de métricas del pipeline completo:

   ```python
   # Celda 12: Resumen de métricas del pipeline Bronze → Silver
   print("=" * 60)
   print("RESUMEN DEL PIPELINE BRONZE → SILVER")
   print("=" * 60)
   print(f"Registros Bronze (entrada)     : {df_bronze_ventas.count():,}")
   print(f"Registros Silver (salida)      : {df_verificacion.count():,}")
   print(f"Registros eliminados (calidad) : {df_bronze_ventas.count() - df_silver_ventas.count():,}")
   print(f"Tabla destino                  : {TABLA_SILVER_NOMBRE}")
   print(f"Formato                        : Delta Lake")
   print(f"Particionamiento               : anio_venta, mes_venta")
   print(f"Operación de escritura         : {'CREATE' if not tabla_existe else 'MERGE'}")
   print("=" * 60)
   print("✅ Pipeline Bronze → Silver completado exitosamente.")
   ```

7. Ejecuta la celda de resumen. Guarda el Notebook usando `Ctrl + S` o el botón **Guardar** en la barra de herramientas.

---

**Salida esperada:**

```
Verificando existencia de la tabla Silver: silver_ventas_enriquecidas
Tabla existente: False
Primera ejecución detectada. Creando tabla Silver con particionamiento...
✅ Tabla 'silver_ventas_enriquecidas' creada exitosamente en Silver.
   Particionada por: anio_venta, mes_venta
   Registros escritos: X,XXX

============================================================
RESUMEN DEL PIPELINE BRONZE → SILVER
============================================================
Registros Bronze (entrada)     : X,XXX
Registros Silver (salida)      : X,XXX
...
✅ Pipeline Bronze → Silver completado exitosamente.
```

---

**Verificación:**

- ✅ La tabla `silver_ventas_enriquecidas` aparece en el panel **Tables** del Lakehouse en el panel izquierdo del Notebook.
- ✅ El historial Delta muestra la versión 0 con operación `WRITE` o `CREATE TABLE`.
- ✅ La distribución por año muestra datos en los rangos de fechas del dataset del curso.
- ✅ El Notebook se guarda sin errores.

---

### Paso 7: Verificar la Tabla Silver desde el SQL Endpoint del Lakehouse

**Objetivo:** Confirmar que la tabla Silver es accesible desde el SQL Endpoint automático del Lakehouse, validando que el formato Delta Lake es correctamente reconocido por el motor SQL de Fabric.

---

#### Instrucciones

1. Agrega una celda Markdown:

   ```markdown
   ## Paso 6: Validación con Spark SQL
   
   Verificamos la tabla Silver usando Spark SQL para confirmar accesibilidad 
   y correctitud de los datos desde una perspectiva de consulta analítica.
   ```

2. Agrega una celda de validación con Spark SQL:

   ```python
   # Celda 13: Validación con Spark SQL
   
   # Consulta 1: Ventas totales por categoría de producto
   print("Consulta 1: Ventas totales por categoría")
   spark.sql(f"""
       SELECT 
           categoria,
           COUNT(*) AS num_transacciones,
           SUM(ingreso_total) AS ingresos_totales,
           AVG(ingreso_total) AS ingreso_promedio
       FROM {TABLA_SILVER_NOMBRE}
       WHERE categoria IS NOT NULL
       GROUP BY categoria
       ORDER BY ingresos_totales DESC
   """).show(10, truncate=False)
   
   # Consulta 2: Top 5 regiones por volumen de ventas
   print("\nConsulta 2: Top 5 regiones por volumen")
   spark.sql(f"""
       SELECT 
           region,
           COUNT(*) AS num_ventas,
           SUM(ingreso_total) AS total_ingresos
       FROM {TABLA_SILVER_NOMBRE}
       GROUP BY region
       ORDER BY total_ingresos DESC
       LIMIT 5
   """).show(truncate=False)
   
   # Consulta 3: Verificación de integridad — no deben existir nulos en columnas clave
   print("\nConsulta 3: Verificación de integridad de datos")
   spark.sql(f"""
       SELECT 
           SUM(CASE WHEN id_transaccion IS NULL THEN 1 ELSE 0 END) AS nulos_id,
           SUM(CASE WHEN fecha_venta IS NULL THEN 1 ELSE 0 END) AS nulos_fecha,
           SUM(CASE WHEN monto < 0 THEN 1 ELSE 0 END) AS montos_negativos,
           COUNT(*) AS total_registros
       FROM {TABLA_SILVER_NOMBRE}
   """).show()
   ```

3. Ejecuta la celda. Las consultas SQL deben devolver resultados coherentes con el dataset del curso.

4. Adicionalmente, navega al Lakehouse en el Workspace:
   - Haz clic en el nombre del Lakehouse en el panel izquierdo del Notebook (o navega al Workspace y abre el Lakehouse).
   - En la vista del Lakehouse, haz clic en **SQL Endpoint** en la esquina superior derecha (o en el selector de modo).
   - En el SQL Endpoint, busca la tabla `silver_ventas_enriquecidas` en el panel izquierdo.
   - Haz clic derecho sobre la tabla y selecciona **Vista previa de datos** para confirmar visualmente que los datos son correctos.

---

**Salida esperada:**

```
Consulta 1: Ventas totales por categoría
+------------------+------------------+------------------+------------------+
|categoria         |num_transacciones |ingresos_totales  |ingreso_promedio  |
+------------------+------------------+------------------+------------------+
|ELECTRONICA       |XXX               |XXXXXX.XX         |XXXX.XX           |
...

Consulta 3: Verificación de integridad de datos
+---------+-----------+----------------+---------------+
|nulos_id |nulos_fecha|montos_negativos|total_registros|
+---------+-----------+----------------+---------------+
|0        |0          |0               |X,XXX          |
+---------+-----------+----------------+---------------+
```

---

**Verificación:**

- ✅ Las consultas SQL devuelven resultados sin errores de sintaxis ni de tabla no encontrada.
- ✅ La Consulta 3 muestra **cero** nulos en columnas críticas y **cero** montos negativos.
- ✅ La tabla `silver_ventas_enriquecidas` es visible y consultable desde el SQL Endpoint del Lakehouse.

---

### Paso 8: Integrar el Notebook en el Pipeline de Data Factory

**Objetivo:** Agregar el Notebook `NB_Bronze_Silver_Ventas` como una actividad dentro del Pipeline de Data Factory creado en el Lab 2, completando el flujo automatizado end-to-end: Ingesta → Bronze → Silver.

---

#### Instrucciones

1. Guarda el Notebook con `Ctrl + S` y ciérralo (o mantén la pestaña abierta para referencia).

2. Navega al **Workspace del curso** haciendo clic en el nombre del Workspace en el panel de navegación izquierdo.

3. Localiza el **Pipeline de Data Factory** creado en el Lab 2 (por ejemplo, `PL_Ingesta_Ventas_Bronze`). Haz clic en él para abrirlo en el editor de pipelines.

4. En el lienzo del Pipeline, identifica la última actividad existente (la actividad de copia o la actividad final del flujo del Lab 2). Haz clic en ella para seleccionarla.

5. En el panel de actividades de la izquierda (o en la barra superior), busca la categoría **Notebook** y arrastra una actividad **Notebook** al lienzo, posicionándola a la derecha de la última actividad existente.

6. Conecta la última actividad existente con la nueva actividad Notebook:
   - Pasa el cursor sobre la actividad existente hasta que aparezca una flecha de conexión en su borde derecho.
   - Arrastra la flecha hacia la nueva actividad Notebook.
   - En el diálogo de tipo de dependencia, selecciona **En caso de éxito (On Success)** (representada por una flecha verde).

7. Haz clic en la actividad Notebook para seleccionarla y configurar sus propiedades en el panel inferior:
   - **Nombre de la actividad:** Escribe `ACT_Transformacion_Bronze_Silver`
   - Haz clic en la pestaña **Configuración** del panel de propiedades.
   - En el campo **Notebook**, haz clic en el selector y elige `NB_Bronze_Silver_Ventas` del Workspace actual.
   - Verifica que el **Workspace** seleccionado es el correcto (el del curso).

8. Opcional — Agregar parámetros al Notebook (para ejecuciones parametrizadas futuras):
   - En la pestaña **Configuración** de la actividad, busca la sección **Parámetros base**.
   - Agrega un parámetro: Nombre `ejecutado_desde_pipeline`, Valor `true`.
   
   > 📝 **Nota:** Para que el Notebook acepte parámetros desde el Pipeline, la primera celda del Notebook debe estar marcada como celda de parámetros. Esto es opcional en este laboratorio pero es una buena práctica para pipelines de producción.

9. Haz clic en **Validar** (ícono de check o botón en la barra superior) para verificar que el Pipeline no tiene errores de configuración. Corrige cualquier advertencia que aparezca.

10. Haz clic en **Guardar** (`Ctrl + S`) para persistir los cambios en el Pipeline.

11. Ejecuta el Pipeline completo en modo de prueba:
    - Haz clic en el botón **Ejecutar** (▶ Run) en la barra superior del editor de Pipeline.
    - Selecciona **Depurar** (Debug) si está disponible, o **Ejecutar ahora** (Run now).
    - Observa el panel de monitoreo en la parte inferior del lienzo. Cada actividad mostrará su estado: pendiente (gris), en ejecución (amarillo/naranja), éxito (verde) o fallo (rojo).

12. Espera a que todas las actividades completen. La actividad del Notebook puede tomar entre 5 y 15 minutos dependiendo de la carga de la capacidad Trial.

---

**Salida esperada:**

- Todas las actividades del Pipeline muestran estado **verde (éxito)** en el panel de monitoreo.
- La actividad `ACT_Transformacion_Bronze_Silver` muestra estado **Succeeded**.
- En el panel de detalles de la actividad Notebook (clic en el ícono de detalles), se puede ver la salida del Notebook con los mensajes de éxito del pipeline.

---

**Verificación:**

- ✅ El Pipeline se guarda sin errores de validación.
- ✅ Todas las actividades del Pipeline muestran estado verde tras la ejecución.
- ✅ La actividad `ACT_Transformacion_Bronze_Silver` aparece en el historial de ejecuciones del Pipeline.
- ✅ La tabla `silver_ventas_enriquecidas` en el Lakehouse muestra una nueva versión en su historial Delta (si el Pipeline se ejecutó por segunda vez, el MERGE habrá creado la versión 1).

---

## 7. Validación y Pruebas Finales

Una vez completados todos los pasos, ejecuta las siguientes verificaciones finales para confirmar que el laboratorio fue completado exitosamente:

### Verificación 1: Estado de las tablas en el Lakehouse

1. Navega al Lakehouse del curso desde el Workspace.
2. En la sección **Tables**, confirma que existen las siguientes tablas:
   - `bronze_ventas` (creada en Lab 02)
   - `bronze_productos` (creada en Lab 02)
   - `bronze_clientes` (creada en Lab 02)
   - `silver_ventas_enriquecidas` ✅ **(nueva — creada en este lab)**

### Verificación 2: Historial Delta de la tabla Silver

Ejecuta la siguiente celda en el Notebook (o en una nueva celda al final):

```python
# Verificación final: Historial de versiones de la tabla Silver
print("Historial de versiones de silver_ventas_enriquecidas:")
DeltaTable.forName(spark, "silver_ventas_enriquecidas").history().select(
    "version", "timestamp", "operation", "userName"
).show(10, truncate=False)
```

**Resultado esperado:** Al menos una versión (versión 0 con operación `CREATE TABLE` o `WRITE`). Si el Pipeline se ejecutó dos veces, habrá también una versión 1 con operación `MERGE`.

### Verificación 3: Conteo de registros por capa

```python
# Verificación de conteos por capa
bronze_count = spark.read.format("delta").load("Tables/bronze_ventas").count()
silver_count = spark.read.format("delta").load("Tables/silver_ventas_enriquecidas").count()

print(f"Bronze (registros crudos)    : {bronze_count:,}")
print(f"Silver (registros limpios)   : {silver_count:,}")
print(f"Diferencia (filtrados)       : {bronze_count - silver_count:,}")
print(f"Tasa de retención            : {(silver_count/bronze_count)*100:.1f}%")

assert silver_count > 0, "ERROR: La tabla Silver está vacía"
assert silver_count <= bronze_count, "ERROR: Silver tiene más registros que Bronze (imposible)"
print("\n✅ Verificación final superada. Lab 03 completado correctamente.")
```

### Verificación 4: Accesibilidad desde SQL Endpoint

1. Navega al Lakehouse → **SQL Endpoint**.
2. En el editor de consultas SQL, ejecuta:

```sql
SELECT 
    COUNT(*) AS total_registros,
    COUNT(DISTINCT id_transaccion) AS transacciones_unicas,
    MIN(fecha_venta) AS fecha_minima,
    MAX(fecha_venta) AS fecha_maxima,
    SUM(ingreso_total) AS ingresos_totales
FROM silver_ventas_enriquecidas;
```

**Resultado esperado:** Una fila con valores coherentes (sin nulos en `total_registros`, fechas dentro del rango del dataset, ingresos positivos).

### Verificación 5: Pipeline integrado

1. Navega al Pipeline `PL_Ingesta_Ventas_Bronze` en el Workspace.
2. Confirma que la actividad `ACT_Transformacion_Bronze_Silver` aparece conectada al flujo existente.
3. En la pestaña **Monitor** del Pipeline (o en el hub de monitoreo de Fabric), confirma que existe al menos una ejecución exitosa que incluye la actividad del Notebook.

---

## 8. Resolución de Problemas

### Problema 1: `AnalysisException: Table or view not found: bronze_ventas`

**Síntomas:**
- Al ejecutar la Celda 3 (lectura de tablas Bronze), el Notebook lanza una excepción similar a:
  ```
  AnalysisException: [TABLE_OR_VIEW_NOT_FOUND] The table or view `bronze_ventas` cannot be found.
  ```
- El panel izquierdo del Notebook no muestra tablas en la sección **Tables** del Lakehouse.

**Causa:**
El Lakehouse no está correctamente conectado al Notebook como almacenamiento por defecto, o las tablas Bronze no fueron creadas en el Lab 02. También puede ocurrir si el Notebook fue creado sin seleccionar el Lakehouse correcto en el paso de configuración inicial.

**Solución:**
1. Verifica en el panel izquierdo del Notebook que el Lakehouse del curso aparece bajo la sección **Lakehouses**. Si no aparece, haz clic en **Agregar Lakehouse** y selecciónalo nuevamente.
2. Si el Lakehouse está conectado pero las tablas no aparecen, navega directamente al Lakehouse desde el Workspace y confirma visualmente que las tablas `bronze_ventas`, `bronze_productos` y `bronze_clientes` existen. Si no existen, regresa al Lab 02 y ejecuta el Pipeline de ingesta.
3. Si las tablas existen en el Lakehouse pero no son visibles en el Notebook, intenta usar la ruta explícita en lugar del nombre de tabla:
   ```python
   # Alternativa: usar ruta absoluta de OneLake
   df_bronze_ventas = spark.read.format("delta").load("Tables/bronze_ventas")
   ```
4. Si el problema persiste, reinicia la sesión de Spark (menú **Sesión → Reiniciar sesión**) y vuelve a ejecutar las celdas desde el inicio.

---

### Problema 2: La actividad Notebook en el Pipeline falla con estado rojo y error de timeout o permisos

**Síntomas:**
- Al ejecutar el Pipeline, la actividad `ACT_Transformacion_Bronze_Silver` muestra estado **Failed** (rojo).
- En los detalles del error se observa uno de los siguientes mensajes:
  - `"Activity timeout: The notebook execution exceeded the configured timeout."`
  - `"Unauthorized: The pipeline does not have permission to execute the notebook."`
  - `"SparkException: Job aborted due to stage failure."`

**Causa:**
Existen tres causas posibles según el mensaje de error:
- **Timeout:** La capacidad Trial está saturada y el clúster Spark tardó más de lo esperado en inicializarse o en completar las transformaciones. El timeout por defecto de la actividad Notebook puede ser demasiado corto.
- **Permisos:** El Pipeline fue creado con una identidad diferente a la del propietario del Notebook, o el Notebook está en un Workspace al que el Pipeline no tiene acceso de ejecución.
- **Error de Spark:** Una celda del Notebook contiene un error lógico que no se manifestó durante la ejecución interactiva pero sí en la ejecución orquestada (por ejemplo, una variable no definida en el contexto del Pipeline).

**Solución:**
1. **Para timeout:** En la actividad Notebook del Pipeline, haz clic en **Configuración avanzada** y aumenta el valor de **Timeout** a `01:00:00` (1 hora). Guarda y vuelve a ejecutar el Pipeline.
2. **Para permisos:** Verifica que el Pipeline y el Notebook están en el **mismo Workspace**. Asegúrate de que tu cuenta tiene permisos de **Contributor** o superior en el Workspace. Si el Pipeline usa una identidad de servicio, otórgale acceso al Workspace desde **Configuración del Workspace → Administrar acceso**.
3. **Para errores de Spark:** Abre el Notebook de forma interactiva y ejecútalo completo (menú **Ejecutar → Ejecutar todo**) para identificar la celda que falla. Corrige el error y vuelve a guardar el Notebook antes de re-ejecutar el Pipeline.
4. En todos los casos, revisa los detalles completos del error haciendo clic en el ícono de información (ⓘ) junto a la actividad fallida en el panel de monitoreo del Pipeline.

---

## 9. Limpieza de Recursos

> ⚠️ **IMPORTANTE — No elimines recursos entre laboratorios:** Los artefactos creados en este lab (Notebook `NB_Bronze_Silver_Ventas` y tabla `silver_ventas_enriquecidas`) son **prerrequisitos directos** del Lab 04 (Modelos Semánticos Direct Lake). **No elimines ni renombres estos recursos hasta completar el Lab 05.**

### Acciones de limpieza al finalizar ESTE laboratorio (sin eliminar recursos)

1. **Detener la sesión de Spark activa** para liberar cómputo de la capacidad Trial:
   - En el Notebook, haz clic en el menú **Sesión** (o **Kernel**) en la barra superior.
   - Selecciona **Detener sesión** (Stop session).
   - Confirma la acción en el diálogo.

2. **Guardar el Notebook** con `Ctrl + S` y cerrar la pestaña del Notebook.

3. **Verificar el estado del Pipeline** en el hub de monitoreo y confirmar que no hay ejecuciones pendientes o en cola.

### Limpieza final (solo después de completar el Lab 05)

Después de finalizar todos los laboratorios del curso, el instructor proporcionará una guía de limpieza final que incluirá:

```
1. Eliminar el Workspace completo del curso (esto elimina todos los artefactos: 
   Lakehouse, Notebooks, Pipelines y Modelos Semánticos).
2. Verificar en el portal de Fabric que la capacidad Trial queda liberada.
3. Cancelar la suscripción Trial si no se continuará usando Fabric.
```

> 💡 **Tip de ahorro de capacidad:** Si necesitas pausar el trabajo entre laboratorios por más de 2 horas, detén siempre la sesión de Spark activa. Las sesiones inactivas consumen unidades de capacidad de la Trial incluso sin ejecutar código.

---

## 10. Resumen

### Lo que construiste en este laboratorio

En este laboratorio implementaste el proceso completo de transformación **Bronze → Silver** dentro de una arquitectura Medallion en Microsoft Fabric. Específicamente:

| Componente creado | Descripción |
|---|---|
| `NB_Bronze_Silver_Ventas` | Notebook de Spark con documentación Markdown y 13 celdas de código |
| `silver_ventas_enriquecidas` | Tabla Delta particionada por `anio_venta` y `mes_venta` en la capa Silver |
| Actividad Notebook en Pipeline | `ACT_Transformacion_Bronze_Silver` integrada en el flujo end-to-end |

### Conceptos clave aplicados

- **Arquitectura Medallion:** La capa Silver es el espacio de refinamiento donde los datos crudos de Bronze se convierten en activos confiables con tipos correctos, sin duplicados y sin valores inválidos.
- **Idempotencia:** El uso de la operación MERGE garantiza que el Notebook puede ejecutarse múltiples veces produciendo el mismo resultado, sin duplicar datos ni corromper la tabla Silver.
- **Validaciones como guardianes:** Las aserciones de calidad antes de la escritura en Silver protegen toda la cadena analítica: si los datos no cumplen los estándares, el Notebook falla de forma controlada antes de escribir datos corruptos.
- **Delta Lake:** El formato Delta proporciona versionado, transacciones ACID y capacidad de auditoría (historial de operaciones) en ambas capas del Lakehouse.
- **Orquestación:** La integración del Notebook en el Pipeline de Data Factory convierte el proceso manual en un flujo automatizado y reproducible.

### Próximos pasos

Con la capa Silver consolidada y los datos limpios almacenados en formato Delta Lake, estás listo para el **Lab 04**: construirás un modelo semántico en modo **Direct Lake** sobre la tabla `silver_ventas_enriquecidas` y lo conectarás a Power BI para análisis de alta velocidad sin necesidad de importar datos.

---

### Recursos adicionales

| Recurso | URL |
|---|---|
| Documentación de Notebooks en Microsoft Fabric | https://learn.microsoft.com/es-es/fabric/data-engineering/how-to-use-notebook |
| Arquitectura Medallion en Microsoft Fabric | https://learn.microsoft.com/es-es/azure/databricks/lakehouse/medallion-architecture |
| Referencia de operaciones Delta Lake (MERGE, overwrite) | https://docs.delta.io/latest/delta-batch.html |
| API de funciones PySpark (`pyspark.sql.functions`) | https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/functions.html |
| Delta Lake — Operación MERGE | https://docs.delta.io/latest/delta-update.html |
| Actividad Notebook en Data Factory Pipeline | https://learn.microsoft.com/es-es/fabric/data-factory/notebook-activity |

---
