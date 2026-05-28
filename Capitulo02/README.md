# Construcción de pipeline de ingesta automatizada hacia capa Bronze

## Metadatos

| Atributo | Detalle |
|---|---|
| **Duración estimada** | 105 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |
| **Laboratorio previo requerido** | Lab 01-00-01 |
| **Tecnología principal** | Microsoft Fabric Data Factory |

---

## Descripción General

En este laboratorio construirás un pipeline de ingesta automatizada en Microsoft Fabric Data Factory que mueve archivos CSV y Parquet desde una fuente externa (Azure Blob Storage o ADLS Gen2) hacia la **capa Bronze** del Lakehouse creado en el Lab 1, siguiendo los principios de la arquitectura Medallion. Implementarás actividades de copia parametrizadas, control de flujo con `ForEach` e `If Condition`, y un trigger de ejecución programada diaria. Al finalizar, validarás el funcionamiento completo revisando el Monitor Hub de Fabric y verificando que los datos aterrizaron correctamente en formato Parquet dentro de la estructura de carpetas particionada por fecha.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:

- [ ] Diseñar y construir un pipeline de datos en Microsoft Fabric Data Factory con múltiples actividades encadenadas (Copy Data, ForEach, If Condition) que ingeste archivos desde una fuente externa hacia la capa Bronze del Lakehouse.
- [ ] Configurar actividades de copia con expresiones dinámicas para parametrizar rutas de origen y destino, habilitando la reutilización del pipeline para múltiples archivos y fechas de ejecución.
- [ ] Implementar lógica de control de flujo y validación básica de calidad de datos dentro del pipeline para detectar ingestas vacías o fallidas de forma temprana.
- [ ] Programar la ejecución automática del pipeline mediante un Scheduled Trigger y monitorear el historial de ejecuciones desde el Monitor Hub de Microsoft Fabric.

---

## Prerrequisitos

### Conocimientos Previos

- Haber completado el **Lab 01-00-01** (Lakehouse configurado con estructura Medallion, shortcuts funcionales y carpetas `bronze/`, `silver/`, `gold/` creadas).
- Comprensión conceptual de ETL/ELT y la diferencia entre transformar en origen vs. en destino.
- Conocimiento básico de formatos de archivo: CSV, Parquet y Delta Lake.
- Familiaridad con conceptos de conectores, servicios vinculados y datasets en herramientas de integración de datos.

### Acceso y Recursos

- Acceso activo a **Microsoft Fabric Trial** con permisos de Member o Admin en el workspace del curso.
- URL de SAS Token para el Azure Blob Storage del instructor (proporcionada antes del laboratorio) **o** acceso a las URLs de demostración alternativas indicadas en la sección de notas del curso.
- Archivos de datos de práctica del curso (dataset de ventas ficticias v1.0) ya disponibles en el storage de origen.
- Power BI Desktop instalado (versión mayo 2024 o superior) — no se usa directamente en este lab, pero debe estar disponible para labs posteriores.

> **⚠️ Nota Importante — Alternativa sin Azure Storage:** Si no tienes acceso a Azure Blob Storage, el instructor habrá cargado los archivos directamente en una subcarpeta `_source_sim/` dentro del Lakehouse de OneLake. En ese caso, sigue las instrucciones marcadas con la etiqueta **[ALT]** en cada paso relevante.

---

## Entorno del Laboratorio

### Requisitos de Hardware

| Componente | Mínimo | Recomendado |
|---|---|---|
| RAM | 8 GB | 16 GB |
| Procesador | Intel Core i5 8ª gen / AMD Ryzen 5 | Intel Core i7 / AMD Ryzen 7 |
| Almacenamiento libre | 2 GB | 5 GB |
| Resolución de pantalla | 1366 × 768 | 1920 × 1080 |
| Conexión a Internet | 10 Mbps | 25 Mbps |

### Requisitos de Software

| Software | Versión Mínima | Notas |
|---|---|---|
| Microsoft Edge / Google Chrome | 110 o superior | Navegador principal para Fabric |
| Microsoft Fabric Trial | Activo (60 días) | Workspace del curso ya creado en Lab 1 |
| Power BI Desktop | Mayo 2024 o superior | Requerido en labs posteriores |
| Azure Storage Explorer (opcional) | 1.34 o superior | Para inspeccionar el storage de origen |

### Verificación del Entorno Antes de Comenzar

Antes de iniciar el laboratorio, verifica que el entorno del Lab 1 está intacto:

1. Abre [https://app.fabric.microsoft.com](https://app.fabric.microsoft.com) en tu navegador.
2. Navega al workspace del curso (ej. `ws_curso_fabric_[tus_iniciales]`).
3. Confirma que el Lakehouse `lh_medallion` existe y tiene las carpetas `bronze/`, `silver/` y `gold/` visibles en la sección **Files**.
4. Confirma que los shortcuts del Lab 1 siguen activos (deben aparecer con el ícono de acceso directo ⇗).

Si alguno de estos elementos no está presente, **detente y contacta al instructor** antes de continuar.

---

## Instrucciones Paso a Paso

---

### Paso 1 — Explorar los Archivos de Origen en Azure Blob Storage

**Objetivo:** Familiarizarte con la estructura y los archivos de datos que ingestará el pipeline antes de construirlo, para tomar decisiones informadas de configuración.

#### Instrucciones

1. Abre **Azure Storage Explorer** en tu computadora (si está instalado). Si no lo tienes, puedes usar la interfaz web del portal de Azure en [https://portal.azure.com](https://portal.azure.com) → busca la cuenta de almacenamiento proporcionada por el instructor.

2. Conéctate al storage del instructor usando el **SAS Token** proporcionado:
   - En Storage Explorer: selecciona *Connect → Blob container or directory → Shared Access Signature (SAS)*.
   - Pega la URL completa con el SAS Token. Ejemplo de formato:
     ```
     https://stcursofabric.blob.core.windows.net/datos-curso?sv=2023-01-03&ss=b&srt=co&sp=rl&se=2024-12-31T00:00:00Z&st=2024-06-01T00:00:00Z&spr=https&sig=XXXXXXXXX
     ```

3. Navega al contenedor `datos-curso` y explora la siguiente estructura de archivos:
   ```
   datos-curso/
   └── ventas/
       ├── ventas_20240101.csv
       ├── ventas_20240102.csv
       ├── ventas_20240103.csv
       └── ... (archivos adicionales)
   └── productos/
       └── productos_catalogo.csv
   └── clientes/
       └── clientes_maestro.csv
   ```

4. Descarga y abre localmente el archivo `ventas_20240101.csv` con Excel o un editor de texto para inspeccionar su estructura. Deberías ver columnas similares a:
   ```
   id_transaccion,fecha_venta,id_cliente,id_producto,cantidad,precio_unitario,total,canal_venta,region
   TXN-001,2024-01-01,CLI-1042,PRD-205,3,150.00,450.00,online,norte
   TXN-002,2024-01-01,CLI-0871,PRD-118,1,89.99,89.99,tienda,centro
   ```

5. Anota el número aproximado de columnas y el separador usado (coma `,`). Esta información la necesitarás al configurar el dataset de origen.

> **[ALT] Sin Azure Storage:** Si usas la alternativa OneLake, el instructor ya habrá copiado los archivos a `Files/_source_sim/ventas/` dentro del Lakehouse `lh_medallion`. Navega al Lakehouse en Fabric y explora esa carpeta para completar los pasos 3-5 de forma equivalente.

#### Resultado Esperado

Tienes una comprensión clara de la estructura de los archivos de origen: formato CSV delimitado por comas, sin encabezado de tipo de datos, con 9 columnas de datos de ventas. Esta información guiará la configuración del pipeline.

#### Verificación

- [ ] Puedes ver al menos 3 archivos CSV en la carpeta `ventas/` del contenedor.
- [ ] El archivo CSV tiene encabezado en la primera fila y usa coma como delimitador.
- [ ] Identificaste el nombre exacto del contenedor y la ruta de la carpeta de ventas.

---

### Paso 2 — Crear el Pipeline Principal en Fabric Data Factory

**Objetivo:** Crear el artefacto Data Pipeline en el workspace de Fabric y familiarizarte con el lienzo de diseño visual.

#### Instrucciones

1. En [https://app.fabric.microsoft.com](https://app.fabric.microsoft.com), navega a tu workspace del curso.

2. En la barra lateral izquierda, haz clic en el ícono **Data Factory** para cambiar a la experiencia de Data Factory (si el workspace muestra otra experiencia activa, usa el selector de experiencias en la esquina inferior izquierda).

3. Haz clic en **+ Nuevo ítem** (o el botón equivalente según la versión actual de Fabric) y selecciona **Data Pipeline** de la lista de opciones.

4. En el cuadro de diálogo de nombre, escribe exactamente:
   ```
   pl_ingesta_ventas_bronze
   ```
   Luego haz clic en **Crear**.

5. Se abrirá el lienzo de diseño del pipeline. Observa los paneles principales:
   - **Lienzo central**: área donde arrastras y conectas actividades.
   - **Panel de actividades** (izquierda o pestaña): lista de actividades disponibles.
   - **Panel de propiedades** (inferior o derecha): configuración de la actividad seleccionada.
   - **Barra de herramientas superior**: botones de Validar, Ejecutar, Publicar.

6. Antes de agregar actividades, crea los **parámetros del pipeline** haciendo clic en el área vacía del lienzo y luego en la pestaña **Parámetros** en el panel inferior. Agrega los siguientes parámetros:

   | Nombre del parámetro | Tipo | Valor predeterminado |
   |---|---|---|
   | `p_contenedor_origen` | String | `datos-curso` |
   | `p_carpeta_origen` | String | `ventas` |
   | `p_fecha_proceso` | String | (dejar vacío — se calculará dinámicamente) |
   | `p_nombre_tabla_destino` | String | `ventas` |

   Para agregar cada parámetro: haz clic en **+ Nuevo**, escribe el nombre, selecciona el tipo y escribe el valor predeterminado si aplica.

7. Haz clic en **Guardar** (ícono de disco o `Ctrl+S`) para preservar los cambios iniciales.

#### Resultado Esperado

El pipeline `pl_ingesta_ventas_bronze` aparece en el workspace con el lienzo vacío y 4 parámetros configurados. El pipeline aún no tiene actividades.

#### Verificación

- [ ] El pipeline aparece en el workspace con el nombre exacto `pl_ingesta_ventas_bronze`.
- [ ] Los 4 parámetros están visibles en la pestaña Parámetros del lienzo.
- [ ] El estado del pipeline muestra "Sin publicar" o similar (cambios no publicados).

---

### Paso 3 — Configurar la Conexión al Azure Blob Storage (Linked Service)

**Objetivo:** Establecer la conexión reutilizable al storage de origen donde residen los archivos CSV de ventas.

#### Instrucciones

1. Dentro del lienzo del pipeline `pl_ingesta_ventas_bronze`, arrastra la actividad **Copy Data** desde el panel de actividades hacia el lienzo. Aparecerá un bloque con el nombre "Copy data1".

2. Con la actividad Copy Data seleccionada, ve al panel inferior y selecciona la pestaña **Origen** (*Source*).

3. Haz clic en el botón **+ Nuevo** junto al campo de conexión de origen para crear una nueva conexión.

4. En el panel de selección de tipo de conector, busca y selecciona **Azure Blob Storage**.

5. Completa el formulario de configuración de la nueva conexión:
   - **Nombre de conexión**: `conn_blob_storage_curso`
   - **Método de autenticación**: `Shared Access Signature (SAS)`
   - **URL de SAS**: Pega la URL completa del SAS Token proporcionada por el instructor.
   - Haz clic en **Probar conexión** para verificar que las credenciales son válidas.
   - Si la prueba es exitosa (aparece el mensaje "Connection successful" o equivalente), haz clic en **Crear**.

6. De vuelta en la configuración del origen, completa los campos restantes:
   - **Formato de archivo**: `DelimitedText` (CSV)
   - **Ruta del archivo — Contenedor**: usa la expresión dinámica:
     ```
     @pipeline().parameters.p_contenedor_origen
     ```
   - **Ruta del archivo — Directorio**: usa la expresión dinámica:
     ```
     @pipeline().parameters.p_carpeta_origen
     ```
   - **Ruta del archivo — Nombre de archivo**: deja en blanco por ahora (se configurará en el ForEach más adelante).
   - **Primera fila como encabezado**: ✅ Activado.
   - **Delimitador de columna**: Coma (`,`).
   - **Delimitador de fila**: Salto de línea predeterminado.
   - **Codificación**: `UTF-8`.

> **[ALT] Sin Azure Storage:** En lugar de Azure Blob Storage, selecciona **OneLake** como tipo de conector y apunta a la ruta `Files/_source_sim/ventas/` dentro del Lakehouse `lh_medallion`. El resto de la configuración de formato CSV es idéntica.

7. Haz clic en **Guardar** (`Ctrl+S`).

#### Resultado Esperado

La actividad Copy Data tiene configurado el origen apuntando al Azure Blob Storage del instructor con autenticación SAS. Los campos de ruta usan referencias a parámetros del pipeline (`@pipeline().parameters.*`), lo que hace la configuración reutilizable.

#### Verificación

- [ ] La conexión `conn_blob_storage_curso` aparece en la lista de conexiones del workspace (puedes verificarlo en **Configuración → Conexiones**).
- [ ] La pestaña Origen de la actividad Copy Data no muestra errores de validación (sin triángulos rojos).
- [ ] Al hacer clic en "Vista previa de datos" (*Data preview*), puedes ver las primeras filas de un archivo CSV de prueba del contenedor.

---

### Paso 4 — Configurar el Destino Bronze en el Lakehouse

**Objetivo:** Definir la ruta de destino dentro del Lakehouse donde los archivos CSV de ventas se almacenarán en formato Parquet con particionamiento por fecha.

#### Instrucciones

1. Con la actividad Copy Data aún seleccionada, ve a la pestaña **Destino** (*Sink*) en el panel inferior.

2. En el campo de tipo de destino, selecciona **Lakehouse** de la lista de conectores disponibles.

3. Haz clic en **Seleccionar** y elige el Lakehouse `lh_medallion` del workspace del curso.

4. En la sección de configuración del destino, completa los campos:
   - **Tipo de destino en Lakehouse**: `Files` (no Tables — en Bronze guardamos archivos crudos, no tablas Delta administradas).
   - **Ruta de la carpeta raíz**: usa la siguiente expresión dinámica para crear la estructura particionada por fecha:
     ```
     bronze/@{pipeline().parameters.p_nombre_tabla_destino}/año=@{formatDateTime(utcNow(),'yyyy')}/mes=@{formatDateTime(utcNow(),'MM')}/dia=@{formatDateTime(utcNow(),'dd')}
     ```
   - **Nombre del archivo de salida**: usa la siguiente expresión para generar un nombre único con timestamp:
     ```
     @{pipeline().parameters.p_nombre_tabla_destino}_@{formatDateTime(utcNow(),'yyyyMMdd_HHmmss')}.parquet
     ```
   - **Formato de archivo de salida**: `Parquet`.
   - **Tipo de compresión Parquet**: `Snappy` (balance óptimo entre velocidad y compresión).
   - **Comportamiento de copia**: `Merge files` (todos los registros del origen en un solo archivo Parquet de salida).

5. La ruta completa resultante para una ejecución del 15 de junio de 2024 a las 14:30:22 sería:
   ```
   Files/bronze/ventas/año=2024/mes=06/dia=15/ventas_20240615_143022.parquet
   ```

6. Haz clic en **Guardar** (`Ctrl+S`).

> **Nota sobre la elección de formato:** Guardamos en **Parquet** en lugar de CSV porque Parquet es un formato columnar que reduce el espacio de almacenamiento entre 60-80% y acelera significativamente las lecturas en Spark durante la fase de transformación Silver. Aunque la capa Bronze almacena datos "crudos", usar Parquet no viola la filosofía Bronze ya que no estamos transformando los valores, solo cambiando el formato de serialización.

#### Resultado Esperado

La actividad Copy Data tiene configurado un destino Parquet en la ruta particionada por fecha dentro de la sección `Files/bronze/ventas/` del Lakehouse `lh_medallion`. Las expresiones dinámicas garantizan que cada ejecución del pipeline crea una nueva carpeta con la fecha del día.

#### Verificación

- [ ] La pestaña Destino no muestra errores de validación.
- [ ] La ruta de destino contiene expresiones dinámicas con `formatDateTime(utcNow(),...)`.
- [ ] El formato de salida está configurado como Parquet con compresión Snappy.

---

### Paso 5 — Agregar Actividad de Lookup para Listar Archivos Pendientes

**Objetivo:** Implementar una actividad Lookup que obtenga la lista de archivos disponibles en el origen para procesarlos dinámicamente con ForEach, en lugar de procesar solo un archivo hardcodeado.

#### Instrucciones

1. En el lienzo del pipeline, **antes** de la actividad Copy Data que ya configuraste, agrega una nueva actividad **Lookup** arrastrándola desde el panel de actividades.

2. Reordena las actividades: la actividad Lookup debe estar a la izquierda y la Copy Data a la derecha. Por ahora **no las conectes** todavía.

3. Renombra la actividad Lookup haciendo doble clic sobre su nombre: escribe `lookup_listar_archivos`.

4. Configura la actividad Lookup:
   - **Tipo de conexión de origen**: Azure Blob Storage (usa la conexión `conn_blob_storage_curso` ya creada).
   - **Formato**: `DelimitedText`.
   - **Ruta — Contenedor**: `@pipeline().parameters.p_contenedor_origen`
   - **Ruta — Directorio**: `@pipeline().parameters.p_carpeta_origen`
   - **Nombre de archivo**: `ventas_20240101.csv` *(temporalmente, para la prueba inicial — luego se parametrizará)*.
   - **Primera fila como encabezado**: ✅ Activado.
   - **Solo primera fila** (*First row only*): ❌ **Desactivado** — queremos obtener todos los registros para contar.

   > **Nota pedagógica:** En un pipeline de producción real, usarías una actividad `Get Metadata` con el argumento `Child items` para obtener la lista de archivos del contenedor. Para este laboratorio, simplificamos usando Lookup para verificar que al menos un archivo existe y tiene datos, lo cual es suficiente para aprender el patrón de control de flujo.

5. Haz clic en **Guardar** (`Ctrl+S`).

#### Resultado Esperado

El lienzo muestra dos actividades no conectadas: `lookup_listar_archivos` y la actividad Copy Data (que renombrarás en el siguiente paso). La actividad Lookup está configurada para leer el primer archivo CSV de ventas.

#### Verificación

- [ ] La actividad Lookup aparece en el lienzo con el nombre `lookup_listar_archivos`.
- [ ] La configuración del Lookup apunta al mismo contenedor y carpeta que el origen del Copy Data.
- [ ] La opción "Solo primera fila" está desactivada.

---

### Paso 6 — Implementar Control de Flujo con If Condition y ForEach

**Objetivo:** Agregar lógica condicional para verificar que el Lookup devolvió datos antes de proceder con la copia, y encapsular la copia dentro de un ForEach para procesar múltiples archivos.

#### Instrucciones

**Parte A — Renombrar y reorganizar actividades**

1. Haz doble clic sobre el nombre de la actividad Copy Data existente y renómbrala: `copy_ventas_a_bronze`.

2. Elimina temporalmente la actividad `copy_ventas_a_bronze` del lienzo principal haciendo clic derecho → Eliminar (la recrearemos dentro del ForEach). **Guarda una captura mental de su configuración** — la replicarás en el paso 6C.

**Parte B — Agregar actividad If Condition**

1. Arrastra una actividad **If Condition** al lienzo, a la derecha de `lookup_listar_archivos`.

2. Conecta `lookup_listar_archivos` → `If Condition` usando la flecha de **éxito** (verde): haz clic en el borde derecho de Lookup y arrastra hasta la actividad If Condition.

3. Renombra la actividad If Condition: `if_archivo_tiene_datos`.

4. En la pestaña **Actividades** del panel inferior de la actividad If Condition, configura la expresión de condición:
   ```
   @greater(activity('lookup_listar_archivos').output.count, 0)
   ```
   Esta expresión evalúa si el Lookup devolvió más de 0 registros.

**Parte C — Configurar la rama "True" (datos encontrados)**

1. Haz clic en el ícono de lápiz ✏️ de la rama **True** (si existe datos) dentro de la actividad If Condition. Se abrirá un sub-lienzo.

2. Dentro del sub-lienzo de la rama True, arrastra una actividad **ForEach** al lienzo.

3. Renombra el ForEach: `foreach_archivos_ventas`.

4. Configura el ForEach:
   - **Items** (elementos a iterar): para este laboratorio, usaremos una lista estática de un archivo como demostración del patrón. En la pestaña **Configuración** del ForEach, escribe en el campo Items:
     ```
     @createArray('ventas_20240101.csv','ventas_20240102.csv','ventas_20240103.csv')
     ```
   - **Secuencial**: ❌ Desactivado (permite ejecución paralela de hasta 20 iteraciones).
   - **Batch count**: `5` (máximo 5 archivos en paralelo — apropiado para capacidad Trial).

5. Haz clic en el ícono de lápiz ✏️ del ForEach para abrir su sub-lienzo interno.

6. Dentro del sub-lienzo del ForEach, agrega una actividad **Copy Data** y nómbrala `copy_archivo_individual_a_bronze`.

7. Configura el **Origen** de esta actividad Copy Data:
   - **Conexión**: `conn_blob_storage_curso`
   - **Formato**: `DelimitedText`
   - **Contenedor**: `@pipeline().parameters.p_contenedor_origen`
   - **Directorio**: `@pipeline().parameters.p_carpeta_origen`
   - **Nombre de archivo**: `@item()` ← Esta es la expresión clave del ForEach: `@item()` representa el elemento actual de la iteración.
   - **Primera fila como encabezado**: ✅ Activado
   - **Delimitador**: Coma

8. Configura el **Destino** de esta actividad Copy Data (igual que el Paso 4, pero con el nombre de archivo derivado del ítem actual):
   - **Lakehouse**: `lh_medallion`
   - **Tipo**: `Files`
   - **Ruta de carpeta**:
     ```
     bronze/@{pipeline().parameters.p_nombre_tabla_destino}/año=@{formatDateTime(utcNow(),'yyyy')}/mes=@{formatDateTime(utcNow(),'MM')}/dia=@{formatDateTime(utcNow(),'dd')}
     ```
   - **Nombre de archivo**:
     ```
     @{replace(item(), '.csv', '')}_@{formatDateTime(utcNow(),'HHmmss')}.parquet
     ```
     Esta expresión toma el nombre del archivo CSV (ej. `ventas_20240101.csv`), elimina la extensión `.csv` y agrega un timestamp de hora para evitar colisiones.
   - **Formato**: `Parquet`
   - **Compresión**: `Snappy`

9. Usa el botón de navegación de migas de pan (*breadcrumb*) en la parte superior del lienzo para regresar al sub-lienzo de la rama True, y luego al lienzo principal del pipeline.

**Parte D — Configurar la rama "False" (sin datos)**

1. De vuelta en el lienzo principal, haz clic en el ícono de lápiz ✏️ de la rama **False** de la actividad `if_archivo_tiene_datos`.

2. Dentro del sub-lienzo de la rama False, arrastra una actividad **Fail** (o **Set Variable** si Fail no está disponible en tu versión) al lienzo.

3. Renómbrala `fail_sin_datos_en_origen`.

4. Configura la actividad Fail:
   - **Mensaje de error**:
     ```
     @concat('No se encontraron datos en el origen para la carpeta: ', pipeline().parameters.p_carpeta_origen, ' en la fecha ', utcNow())
     ```
   - **Código de error**: `1001`

5. Regresa al lienzo principal usando las migas de pan.

6. Haz clic en **Guardar** (`Ctrl+S`).

#### Resultado Esperado

El lienzo principal muestra el flujo: `lookup_listar_archivos` → `if_archivo_tiene_datos`. La rama True contiene el ForEach que itera sobre los archivos y ejecuta la copia individual. La rama False contiene la actividad Fail que termina el pipeline con un error descriptivo si no hay datos.

#### Verificación

- [ ] La actividad If Condition tiene la expresión `@greater(activity('lookup_listar_archivos').output.count, 0)`.
- [ ] El ForEach tiene configurado `@item()` como nombre de archivo en el origen del Copy Data interno.
- [ ] La expresión del nombre de archivo destino usa `@{replace(item(), '.csv', '')}` para derivar el nombre Parquet.
- [ ] La rama False tiene una actividad Fail con mensaje descriptivo.

---

### Paso 7 — Validar y Ejecutar el Pipeline Manualmente

**Objetivo:** Usar la función de validación integrada para detectar errores de configuración y ejecutar el pipeline manualmente para verificar su funcionamiento end-to-end.

#### Instrucciones

1. En la barra de herramientas superior del pipeline, haz clic en el botón **Validar** (ícono de palomita o botón con texto "Validate").

2. Revisa el panel de resultados de validación. Si hay errores:
   - **Errores en rojo**: deben corregirse antes de continuar. Haz clic en cada error para ir directamente a la actividad problemática.
   - **Advertencias en amarillo**: son informativas; el pipeline puede ejecutarse pero revísalas.
   
   Errores comunes en este punto y sus soluciones rápidas:
   - *"Expression is invalid"*: verifica que las comillas en las expresiones sean rectas (`'`) no tipográficas (`'`).
   - *"Connection not found"*: confirma que `conn_blob_storage_curso` está disponible en el workspace.
   - *"Required field missing"*: revisa que todos los campos marcados con asterisco (*) estén completados.

3. Una vez que la validación muestra **0 errores**, haz clic en **Publicar** para guardar el pipeline en el workspace.

   > **Importante:** En Fabric, los cambios no se persisten permanentemente hasta que haces clic en **Publicar**. El botón Guardar (`Ctrl+S`) guarda en borrador temporal, pero Publicar lo confirma en el workspace.

4. Después de publicar, haz clic en **Ejecutar** (botón ▶ o "Run") en la barra de herramientas.

5. Aparecerá un cuadro de diálogo para confirmar los valores de los parámetros. Verifica que los valores predeterminados son correctos y haz clic en **Ejecutar** / **OK**.

6. El pipeline comenzará a ejecutarse. Observa el lienzo: las actividades mostrarán indicadores visuales de estado:
   - **Gris**: en espera
   - **Naranja/Girando**: en ejecución
   - **Verde ✓**: completado con éxito
   - **Rojo ✗**: fallido

7. Espera a que todas las actividades muestren estado verde (puede tomar entre 2-8 minutos dependiendo de la carga del tenant).

8. Una vez completado, navega al **Lakehouse `lh_medallion`** en el workspace y expande la sección **Files → bronze → ventas**. Deberías ver la estructura de carpetas particionada por fecha con los archivos Parquet generados:
   ```
   Files/
   └── bronze/
       └── ventas/
           └── año=2024/
               └── mes=06/
                   └── dia=15/
                       ├── ventas_20240101_143022.parquet
                       ├── ventas_20240102_143025.parquet
                       └── ventas_20240103_143028.parquet
   ```

#### Resultado Esperado

El pipeline ejecuta exitosamente todas las actividades (Lookup → If Condition rama True → ForEach → 3 × Copy Data). Los archivos Parquet aparecen en la estructura de carpetas Bronze del Lakehouse, con nombres derivados de los archivos CSV originales.

#### Verificación

- [ ] La validación del pipeline muestra 0 errores antes de la ejecución.
- [ ] Todas las actividades en el lienzo muestran estado verde tras la ejecución.
- [ ] Los archivos Parquet aparecen en `Files/bronze/ventas/año=XXXX/mes=XX/dia=XX/` en el Lakehouse.
- [ ] El número de archivos Parquet en Bronze coincide con el número de elementos en el array del ForEach (3 archivos).

---

### Paso 8 — Verificar los Datos en Bronze con una Vista Previa

**Objetivo:** Confirmar que los archivos Parquet en Bronze contienen los datos correctos y que el esquema se preservó durante la ingesta.

#### Instrucciones

1. En el workspace de Fabric, abre el Lakehouse `lh_medallion`.

2. En el explorador de archivos del Lakehouse (panel izquierdo), navega hasta uno de los archivos Parquet recién creados en `Files/bronze/ventas/año=XXXX/mes=XX/dia=XX/`.

3. Haz clic derecho sobre el archivo `.parquet` y selecciona **Vista previa** o **Load to Tables** → **Preview** (la opción exacta puede variar según la versión de Fabric).

4. Verifica en la vista previa:
   - El número de columnas coincide con el CSV original (9 columnas).
   - Los nombres de columnas son correctos: `id_transaccion`, `fecha_venta`, `id_cliente`, `id_producto`, `cantidad`, `precio_unitario`, `total`, `canal_venta`, `region`.
   - Los valores de datos se ven correctos (sin truncamientos ni caracteres extraños).
   - No hay filas vacías al inicio (el encabezado CSV fue correctamente excluido del contenido de datos).

5. Opcionalmente, abre un **Notebook** en el Lakehouse y ejecuta el siguiente código para una verificación más detallada:

```python
# Celda 1: Leer los archivos Parquet de Bronze y verificar la ingesta
from pyspark.sql import SparkSession
from pyspark.sql.functions import count, col

spark = SparkSession.builder.getOrCreate()

# Leer todos los archivos Parquet de la carpeta de ventas Bronze
# Fabric infiere el esquema automáticamente desde Parquet
df_bronze_ventas = spark.read.parquet(
    "Files/bronze/ventas/"
)

# Mostrar el esquema inferido
print("=== Esquema de los datos Bronze ===")
df_bronze_ventas.printSchema()

# Contar el total de registros ingestados
total_registros = df_bronze_ventas.count()
print(f"\n=== Total de registros en Bronze: {total_registros} ===")

# Mostrar las primeras 5 filas
print("\n=== Muestra de datos (primeras 5 filas) ===")
df_bronze_ventas.show(5, truncate=False)

# Verificación de calidad básica: columnas nulas
print("\n=== Conteo de nulos por columna ===")
df_bronze_ventas.select(
    [count(col(c)).alias(c) for c in df_bronze_ventas.columns]
).show()
```

```python
# Celda 2: Validación de integridad - confirmar que no hay duplicados de id_transaccion
from pyspark.sql.functions import countDistinct

total = df_bronze_ventas.count()
distintos = df_bronze_ventas.select(
    countDistinct("id_transaccion")
).collect()[0][0]

print(f"Total registros: {total}")
print(f"IDs de transacción únicos: {distintos}")

if total == distintos:
    print("✅ VALIDACIÓN EXITOSA: No hay duplicados de id_transaccion")
else:
    print(f"⚠️ ADVERTENCIA: Hay {total - distintos} registros duplicados")
```

6. Ejecuta ambas celdas y verifica que los resultados son consistentes con los datos originales.

#### Resultado Esperado

Los archivos Parquet en Bronze contienen exactamente los mismos datos que los CSV de origen, con el esquema correcto y sin duplicados. El total de registros en Bronze debe ser la suma de todos los registros en los 3 archivos CSV procesados.

#### Verificación

- [ ] La vista previa del archivo Parquet muestra las 9 columnas esperadas.
- [ ] El código PySpark ejecuta sin errores y muestra el esquema correcto.
- [ ] El total de registros es mayor a 0 y consistente con el origen.
- [ ] La validación de duplicados muestra "VALIDACIÓN EXITOSA".

---

### Paso 9 — Configurar el Scheduled Trigger para Ejecución Automática Diaria

**Objetivo:** Programar la ejecución automática del pipeline para que corra diariamente sin intervención manual, simulando un proceso de ingesta productivo.

#### Instrucciones

1. Regresa al pipeline `pl_ingesta_ventas_bronze` en el workspace (haz clic en su nombre en el workspace o en el breadcrumb si aún está abierto).

2. En la barra de herramientas del pipeline, busca la opción **Programar** (*Schedule*) o **Agregar trigger** → **Nuevo/Editar**. La ubicación exacta puede variar; busca por el ícono de reloj ⏰ o el texto "Schedule".

3. Haz clic en **+ Nuevo trigger** y selecciona **Programado** (*Scheduled*).

4. Completa el formulario del trigger con la siguiente configuración:

   | Campo | Valor |
   |---|---|
   | **Nombre del trigger** | `trigger_ingesta_diaria_ventas` |
   | **Descripción** | Ejecuta la ingesta de ventas diariamente a las 3:00 AM |
   | **Tipo** | Programado (Schedule) |
   | **Fecha de inicio** | Fecha de hoy (en formato de tu zona horaria) |
   | **Zona horaria** | Selecciona la zona horaria de tu organización (ej. `(UTC-06:00) Central Time (US & Canada)`) |
   | **Recurrencia** | Cada `1` día(s) |
   | **Hora de ejecución** | `03:00 AM` |
   | **Fecha de fin** | Sin fecha de fin (o 30 días en el futuro para el laboratorio) |

5. La configuración interna del trigger corresponde a este JSON (solo referencia — no necesitas escribirlo manualmente):
   ```json
   {
     "name": "trigger_ingesta_diaria_ventas",
     "type": "ScheduleTrigger",
     "recurrence": {
       "frequency": "Day",
       "interval": 1,
       "startTime": "2024-06-16T03:00:00",
       "timeZone": "Central Standard Time"
     }
   }
   ```

6. Haz clic en **Guardar** o **Aceptar** para confirmar la configuración del trigger.

7. Aparecerá un mensaje preguntando si deseas **activar** el trigger inmediatamente. Haz clic en **Sí** para activarlo.

   > **Nota:** El trigger no ejecutará el pipeline ahora mismo (son las 3:00 AM del día siguiente). Solo lo ejecutará en el horario programado. Para este laboratorio, la ejecución manual del Paso 7 ya validó el funcionamiento.

8. Puedes verificar que el trigger está activo navegando a **Configuración → Triggers** en el workspace o buscando la sección de triggers en la vista del pipeline.

9. Haz clic en **Publicar** para guardar todos los cambios incluyendo el trigger.

#### Resultado Esperado

El trigger `trigger_ingesta_diaria_ventas` está activo y configurado para ejecutar el pipeline `pl_ingesta_ventas_bronze` diariamente a las 3:00 AM en la zona horaria seleccionada. El estado del trigger muestra "Activo" o "Enabled".

#### Verificación

- [ ] El trigger `trigger_ingesta_diaria_ventas` aparece en la lista de triggers del pipeline.
- [ ] El estado del trigger es "Activo" / "Enabled".
- [ ] La próxima ejecución programada muestra la fecha y hora correctas (mañana a las 3:00 AM).
- [ ] El pipeline fue publicado después de agregar el trigger.

---

### Paso 10 — Monitorear el Pipeline en el Monitor Hub

**Objetivo:** Usar el Monitor Hub de Microsoft Fabric para analizar el historial de ejecuciones del pipeline, revisar tiempos de procesamiento e identificar métricas clave de la ingesta.

#### Instrucciones

1. En la barra de navegación izquierda de Fabric, busca el ícono de **Monitor** (ícono de gráfico de barras o actividad) y haz clic en él para abrir el **Monitor Hub**.

   Alternativa: en el menú superior del workspace, busca **Monitorear** o navega a través del selector de experiencias.

2. En el Monitor Hub, selecciona la pestaña **Ejecuciones de pipeline** (*Pipeline runs*) o el filtro equivalente.

3. Localiza las ejecuciones del pipeline `pl_ingesta_ventas_bronze`. Deberías ver al menos la ejecución manual del Paso 7. Haz clic sobre ella para ver los detalles.

4. En la vista de detalle de la ejecución, examina:
   - **Estado general**: Succeeded / Failed.
   - **Duración total**: tiempo desde inicio hasta fin.
   - **Actividades individuales**: expande cada actividad para ver su duración y estado.
   
5. Haz clic sobre la actividad `foreach_archivos_ventas` para ver las iteraciones individuales del ForEach. Deberías ver 3 iteraciones (una por cada archivo CSV).

6. Para cada iteración del ForEach, observa:
   - **Filas leídas** (*Rows read*): número de registros leídos del CSV de origen.
   - **Filas escritas** (*Rows written*): debe ser igual a filas leídas (sin filtrado en Bronze).
   - **Throughput**: velocidad de transferencia en filas/segundo o MB/segundo.
   - **Duración**: tiempo de la copia individual.

7. Anota los siguientes valores en la tabla de registro del laboratorio:

   | Métrica | Valor observado |
   |---|---|
   | Duración total del pipeline | ___ segundos/minutos |
   | Filas totales escritas en Bronze | ___ registros |
   | Archivo con mayor tiempo de copia | ___ |
   | Estado de la actividad Lookup | Succeeded / Failed |
   | Estado del ForEach | Succeeded / Failed |

8. Regresa al Monitor Hub y haz clic en **Exportar** (si está disponible) para descargar el log de ejecución en formato CSV. Esto simula la práctica de auditoría de pipelines en producción.

9. Opcionalmente, filtra por "Últimas 24 horas" y verifica que no hay otras ejecuciones inesperadas del pipeline.

#### Resultado Esperado

El Monitor Hub muestra el historial completo de ejecución del pipeline con todas las actividades en estado "Succeeded". Las métricas de throughput y conteo de filas son visibles y consistentes con los datos de origen. La tabla de registro del laboratorio está completada con valores reales.

#### Verificación

- [ ] El Monitor Hub muestra al menos 1 ejecución exitosa de `pl_ingesta_ventas_bronze`.
- [ ] Las 3 iteraciones del ForEach se completaron con éxito.
- [ ] Las filas leídas y filas escritas son iguales en cada iteración (sin pérdida de datos).
- [ ] La tabla de métricas del laboratorio está completada con valores reales observados.

---

## Validación y Pruebas Finales

Antes de dar el laboratorio por completado, ejecuta las siguientes validaciones integrales:

### Lista de Verificación Final

```python
# Ejecuta este código en un Notebook del Lakehouse para validación completa del Lab 2

from pyspark.sql import SparkSession
from pyspark.sql.functions import count, col, countDistinct, max, min
import datetime

spark = SparkSession.builder.getOrCreate()

print("=" * 60)
print("VALIDACIÓN FINAL - LAB 02-00-01")
print("=" * 60)

# 1. Verificar existencia de archivos Parquet en Bronze
try:
    df_bronze = spark.read.parquet("Files/bronze/ventas/")
    total_registros = df_bronze.count()
    print(f"\n✅ TEST 1 PASADO: Archivos Parquet encontrados en Bronze")
    print(f"   Total de registros: {total_registros}")
except Exception as e:
    print(f"\n❌ TEST 1 FALLIDO: No se encontraron archivos en Bronze")
    print(f"   Error: {e}")

# 2. Verificar esquema esperado
columnas_esperadas = {
    'id_transaccion', 'fecha_venta', 'id_cliente', 'id_producto',
    'cantidad', 'precio_unitario', 'total', 'canal_venta', 'region'
}
columnas_actuales = set(df_bronze.columns)

if columnas_esperadas.issubset(columnas_actuales):
    print(f"\n✅ TEST 2 PASADO: Esquema correcto - todas las columnas esperadas presentes")
else:
    columnas_faltantes = columnas_esperadas - columnas_actuales
    print(f"\n❌ TEST 2 FALLIDO: Columnas faltantes: {columnas_faltantes}")

# 3. Verificar que no hay registros completamente nulos
registros_nulos = df_bronze.filter(col("id_transaccion").isNull()).count()
if registros_nulos == 0:
    print(f"\n✅ TEST 3 PASADO: No hay registros con id_transaccion nulo")
else:
    print(f"\n⚠️ TEST 3 ADVERTENCIA: {registros_nulos} registros con id_transaccion nulo")

# 4. Verificar rango de fechas en los datos
fecha_min = df_bronze.agg(min("fecha_venta")).collect()[0][0]
fecha_max = df_bronze.agg(max("fecha_venta")).collect()[0][0]
print(f"\n✅ TEST 4 INFO: Rango de fechas en Bronze: {fecha_min} → {fecha_max}")

# 5. Resumen final
print("\n" + "=" * 60)
print(f"RESUMEN: {total_registros} registros ingestados en Bronze")
print(f"Columnas: {len(df_bronze.columns)}")
print(f"Particiones Parquet: {df_bronze.rdd.getNumPartitions()}")
print("=" * 60)
print("\n🎉 Lab 02-00-01 completado exitosamente si todos los TESTSs pasaron.")
```

### Criterios de Aceptación

| Criterio | Estado esperado |
|---|---|
| Pipeline `pl_ingesta_ventas_bronze` existe en el workspace | ✅ Creado y publicado |
| Pipeline tiene 4 parámetros configurados | ✅ Verificado en Paso 2 |
| Conexión `conn_blob_storage_curso` activa | ✅ Verificado en Paso 3 |
| Archivos Parquet en `Files/bronze/ventas/año=*/mes=*/dia=*/` | ✅ Verificado en Paso 7 |
| Total de registros > 0 en Bronze | ✅ Verificado en Paso 8 |
| Trigger `trigger_ingesta_diaria_ventas` activo | ✅ Verificado en Paso 9 |
| Ejecución exitosa visible en Monitor Hub | ✅ Verificado en Paso 10 |

---

## Solución de Problemas

### Problema 1 — La actividad Copy Data falla con error "Access Denied" o "Authentication Failed"

**Síntomas:**
- La actividad Copy Data muestra estado rojo ✗ en el lienzo.
- El mensaje de error en el Monitor Hub dice: `"Access to the specified resource is denied"` o `"The SAS token has expired or is invalid"`.
- El Lookup puede haber pasado pero el Copy Data falla.

**Causa raíz:**
El SAS Token usado en la conexión `conn_blob_storage_curso` ha expirado o fue ingresado incorrectamente. Los SAS Tokens tienen una fecha de expiración definida y son sensibles a mayúsculas/minúsculas. También puede ocurrir si el token fue copiado con espacios al inicio o al final.

**Solución:**
1. Solicita al instructor el SAS Token actualizado si ha expirado.
2. Navega a **Configuración del workspace → Conexiones** (o busca "Manage connections" en Fabric).
3. Localiza `conn_blob_storage_curso` y haz clic en **Editar**.
4. Borra el campo de SAS Token y pega el nuevo token asegurándote de no incluir espacios.
5. Haz clic en **Probar conexión** para confirmar que funciona antes de guardar.
6. Regresa al pipeline y ejecuta nuevamente con el botón ▶.

Si el problema persiste después de actualizar el token, verifica que el SAS Token tiene permisos de **lectura (Read)** y **listado (List)** sobre el contenedor, no solo de escritura.

---

### Problema 2 — El ForEach itera pero los archivos Parquet no aparecen en el Lakehouse

**Síntomas:**
- Todas las actividades en el Monitor Hub muestran estado verde (éxito).
- Las métricas del Copy Data interno al ForEach muestran "Rows written: 0" o un número positivo.
- Sin embargo, al navegar a `Files/bronze/ventas/` en el Lakehouse, la carpeta está vacía o no existe.
- El Notebook de verificación del Paso 8 falla con `Path does not exist`.

**Causa raíz:**
La ruta de destino en la actividad Copy Data interna al ForEach contiene un error en la expresión dinámica (comillas incorrectas, función mal escrita o parámetro con nombre equivocado), lo que hace que Fabric escriba los archivos en una ruta diferente a la esperada. También puede ocurrir si se seleccionó **Tables** en lugar de **Files** como tipo de destino en el Lakehouse.

**Solución:**
1. Abre el pipeline y navega hasta la actividad `copy_archivo_individual_a_bronze` dentro del ForEach.
2. En la pestaña Destino, verifica que el tipo de destino es **Files** (no Tables).
3. Haz clic en el campo de ruta de carpeta y revisa la expresión completa. Copia la expresión y pégala en el **Constructor de expresiones** (*Expression builder*) para validarla:
   ```
   bronze/@{pipeline().parameters.p_nombre_tabla_destino}/año=@{formatDateTime(utcNow(),'yyyy')}/mes=@{formatDateTime(utcNow(),'MM')}/dia=@{formatDateTime(utcNow(),'dd')}
   ```
4. Usa la función **Evaluar expresión** (*Evaluate expression*) del constructor para ver el valor resultante con los parámetros actuales.
5. Si la expresión evalúa correctamente, navega al Lakehouse y usa el buscador de archivos para buscar el nombre del archivo Parquet (puede estar en una ruta ligeramente diferente).
6. Si encontraste los archivos en una ruta incorrecta (ej. `Files/bronze/ventas/` sin las subcarpetas de fecha), corrige la expresión, publica el pipeline y ejecuta nuevamente.
7. Para limpiar archivos en ruta incorrecta: en el Lakehouse, haz clic derecho sobre la carpeta incorrecta → **Eliminar**.

---

## Limpieza de Recursos

> **⚠️ ADVERTENCIA CRÍTICA:** **NO elimines** el Lakehouse `lh_medallion`, el pipeline `pl_ingesta_ventas_bronze` ni los archivos Parquet en Bronze. Estos recursos son **prerrequisitos obligatorios** para el Lab 03-00-01 (transformación Bronze → Silver con PySpark). Eliminarlos requeriría repetir este laboratorio completo.

### Limpieza Permitida en Este Laboratorio

Las siguientes acciones son seguras y opcionales para liberar recursos:

1. **Desactivar el trigger temporalmente** (opcional — para evitar ejecuciones automáticas mientras no estás en clase):
   - Navega al pipeline `pl_ingesta_ventas_bronze`.
   - En la sección de triggers, selecciona `trigger_ingesta_diaria_ventas`.
   - Cambia el estado a **Inactivo** / **Disabled**.
   - Recuerda **reactivarlo** antes del Lab 5 si necesitas datos frescos.

2. **Cerrar el Notebook de verificación** sin guardarlo si no lo necesitas:
   - Si creaste un Notebook temporal para las verificaciones del Paso 8, puedes cerrarlo sin guardar o eliminarlo del workspace.

3. **Limpiar archivos Parquet duplicados** (si ejecutaste el pipeline múltiples veces durante pruebas):
   - En el Lakehouse, navega a `Files/bronze/ventas/`.
   - Si hay múltiples carpetas de fecha del mismo día con archivos duplicados, puedes eliminar las carpetas de fechas de prueba anteriores, conservando solo la más reciente.
   - **No elimines** la carpeta `bronze/` ni sus subcarpetas principales.

### Estado Final Esperado del Workspace

Al finalizar el Lab 2, tu workspace debe contener:

| Artefacto | Estado |
|---|---|
| Lakehouse `lh_medallion` | ✅ Activo con datos en `Files/bronze/ventas/` |
| Pipeline `pl_ingesta_ventas_bronze` | ✅ Publicado |
| Trigger `trigger_ingesta_diaria_ventas` | ✅ Activo (o inactivo temporalmente) |
| Conexión `conn_blob_storage_curso` | ✅ Activa |
| Shortcuts del Lab 1 | ✅ Sin cambios |

---

## Resumen

En este laboratorio construiste un pipeline de ingesta automatizada de extremo a extremo en Microsoft Fabric Data Factory. Los logros principales fueron:

- **Diseñaste** la arquitectura del pipeline siguiendo el principio Bronze de "aterrizar primero, transformar después", preservando los datos crudos en formato Parquet con particionamiento por fecha.
- **Implementaste** una cadena de actividades: Lookup para validar la existencia de datos → If Condition para bifurcar el flujo → ForEach para procesar múltiples archivos en paralelo → Copy Data parametrizado para cada archivo individual.
- **Aplicaste** expresiones dinámicas (`@{formatDateTime(utcNow(),...)}`, `@item()`, `@pipeline().parameters.*`) para hacer el pipeline reutilizable y autónomo en cualquier fecha de ejecución.
- **Automatizaste** la ejecución con un Scheduled Trigger diario a las 3:00 AM, eliminando la necesidad de intervención manual.
- **Monitoreaste** la ejecución desde el Monitor Hub, analizando métricas de throughput, conteo de filas y tiempos de procesamiento.

### Conceptos Clave Reforzados

| Concepto | Aplicación en el Lab |
|---|---|
| Arquitectura Medallion — capa Bronze | Almacenamiento de datos crudos en Parquet particionado por fecha |
| Expresiones dinámicas en Fabric | `@{formatDateTime(...)}`, `@item()`, `@pipeline().parameters.*` |
| Control de flujo | If Condition para validar datos; ForEach para iterar archivos |
| Parametrización de pipelines | 4 parámetros reutilizables para contenedor, carpeta, fecha y tabla |
| Scheduled Triggers | Ejecución automática diaria sin intervención humana |
| Monitor Hub | Auditoría de ejecuciones, métricas y diagnóstico de errores |

### Recursos Adicionales

- [Documentación oficial: Pipelines en Microsoft Fabric Data Factory](https://learn.microsoft.com/es-es/fabric/data-factory/data-factory-overview)
- [Actividad Copy Data — referencia completa de configuración](https://learn.microsoft.com/es-es/fabric/data-factory/copy-data-activity)
- [Expresiones y funciones dinámicas en Fabric / ADF](https://learn.microsoft.com/es-es/azure/data-factory/control-flow-expression-language-functions)
- [Actividad ForEach en pipelines de Fabric](https://learn.microsoft.com/es-es/fabric/data-factory/foreach-activity)
- [Monitor Hub — supervisión de actividad en Microsoft Fabric](https://learn.microsoft.com/es-es/fabric/data-factory/monitor-pipeline-runs)
- [Formato Apache Parquet — documentación oficial](https://parquet.apache.org/docs/)

### Próximo Laboratorio

Con la capa Bronze poblada y el pipeline de ingesta funcionando automáticamente, el **Lab 03-00-01** te llevará al siguiente nivel de la arquitectura Medallion: construirás un Notebook de PySpark que leerá los archivos Parquet desde Bronze, aplicará reglas de limpieza, estandarización y enriquecimiento de datos, y escribirá el resultado en la **capa Silver** en formato Delta Lake, aprovechando las capacidades transaccionales ACID que este formato ofrece dentro de Microsoft Fabric.

---
*Lab 02-00-01 — Microsoft Fabric: Ingeniería de Datos con Arquitectura Medallion*
*Versión 1.0 | Nivel: Avanzado | Duración: 105 minutos*
