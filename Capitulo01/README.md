# Configuración de Lakehouse y conectividad externa mediante Shortcuts

---

## 1. Metadatos

| Atributo | Detalle |
|---|---|
| **Duración estimada** | 75 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (*Apply*) |
| **Módulo** | Capítulo 1 — Fundamentos de Lakehouse en Microsoft Fabric |
| **Dependencias** | Ninguna (es el primer laboratorio de la serie) |
| **Produce artefactos para** | Lab 02, Lab 03, Lab 04, Lab 05 |

---

## 2. Descripción General

En este laboratorio configurarás los cimientos de toda la arquitectura de datos del curso. Comenzarás creando un **Workspace dedicado** en Microsoft Fabric con capacidad Trial habilitada y aprovisionar un **Lakehouse** siguiendo la estructura de carpetas de la arquitectura Medallion (Bronze / Silver / Gold). La parte central del laboratorio se enfoca en la creación de **Shortcuts** hacia fuentes externas en Azure Data Lake Storage Gen2 o Azure Blob Storage, configurando las credenciales de acceso mediante SAS Token. Finalizarás validando la conectividad ejecutando consultas SQL sobre los datos expuestos por los Shortcuts a través del **SQL Endpoint** del Lakehouse, sin haber copiado un solo byte de datos al almacenamiento de OneLake.

> ⚠️ **DEPENDENCIA CRÍTICA:** Los artefactos creados en este laboratorio (Workspace, Lakehouse y Shortcuts) son consumidos por todos los laboratorios posteriores. **No elimines ningún recurso** al terminar esta práctica.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Crear y configurar un Workspace de Microsoft Fabric con capacidad Trial habilitada y los permisos necesarios para soportar una arquitectura Medallion completa.
- [ ] Aprovisionar un Lakehouse en Microsoft Fabric y organizar manualmente la estructura de carpetas para las capas Bronze, Silver y Gold dentro del área `Files`.
- [ ] Crear Shortcuts hacia fuentes de datos externas (Azure Data Lake Storage Gen2 y/o Azure Blob Storage) para habilitar acceso federado sin duplicación de datos.
- [ ] Validar la conectividad a los datos externos ejecutando consultas `SELECT` sobre los Shortcuts mediante el SQL Endpoint del Lakehouse.

---

## 4. Prerrequisitos

### Conocimientos previos

| Área | Nivel requerido |
|---|---|
| Conceptos generales de Data Lake y almacenamiento en la nube | Básico |
| Navegación en portales web (Azure Portal, Microsoft 365) | Básico |
| SQL (sentencias `SELECT`, `FROM`, `WHERE`) | Básico |
| Conceptos de arquitectura Medallion (Bronze / Silver / Gold) | Conceptual (cubierto en la lección teórica 1.1) |

### Acceso y credenciales necesarios

| Recurso | Detalle |
|---|---|
| **Tenant de Microsoft Fabric** | Trial activo (60 días) con permisos para crear Workspaces |
| **Cuenta Microsoft / Azure AD** | Cuenta organizacional o cuenta personal con Fabric Trial activado |
| **SAS Token o credenciales de Storage** | Proporcionados por el instructor. Deben incluir acceso de lectura a los contenedores de práctica |
| **Archivos de datos de práctica** | Dataset del curso (v1.0) — cargado por el instructor en el Storage Account de demostración |
| **Navegador web** | Microsoft Edge o Google Chrome versión 110 o superior |

> 📋 **Nota para el instructor:** Antes de iniciar el laboratorio, asegúrate de haber distribuido a cada estudiante:
> - La URL del Storage Account (`https://<nombre_cuenta>.dfs.core.windows.net` o `https://<nombre_cuenta>.blob.core.windows.net`)
> - El nombre del contenedor de práctica (por ejemplo: `ventas-raw`)
> - Un SAS Token con permisos de lectura y listado (`rl`) con vigencia mínima de 30 días
> - Los nombres de las carpetas dentro del contenedor que los estudiantes deben mapear como Shortcuts

---

## 5. Entorno del Laboratorio

### Hardware recomendado

| Componente | Mínimo | Recomendado |
|---|---|---|
| Procesador | Intel Core i5 8ª gen / AMD Ryzen 5 | Intel Core i7 / AMD Ryzen 7 |
| Memoria RAM | 8 GB | 16 GB |
| Almacenamiento libre | 2 GB | 5 GB |
| Resolución de pantalla | 1366 × 768 | 1920 × 1080 |
| Conexión a Internet | 10 Mbps | 25 Mbps |

### Software requerido

| Software | Versión mínima | Uso en este lab |
|---|---|---|
| Microsoft Edge / Google Chrome | 110+ | Acceso al portal de Fabric |
| Microsoft Fabric Portal | N/A (SaaS) | Creación de Workspace, Lakehouse y Shortcuts |
| Power BI Desktop | Mayo 2024+ | No requerido en este lab (se usa en Lab 04) |
| Azure Storage Explorer (opcional) | 1.34+ | Verificación visual del contenido del Storage Account |

### Configuración inicial del entorno

Antes de comenzar los pasos del laboratorio, realiza las siguientes verificaciones:

**Verificación 1 — Confirmar acceso a Microsoft Fabric:**

1. Abre tu navegador y navega a: `https://app.fabric.microsoft.com`
2. Inicia sesión con tu cuenta organizacional o personal.
3. Confirma que en la esquina superior derecha aparece el ícono de tu cuenta y que **no** ves un mensaje de "Fabric no disponible" o "Sin licencia".
4. Si ves una pantalla de bienvenida con opción de "Iniciar prueba", activa el Trial haciendo clic en **Iniciar prueba gratuita** y sigue las instrucciones en pantalla.

**Verificación 2 — Confirmar recepción de credenciales del instructor:**

Antes de continuar, asegúrate de tener a mano (en un bloc de notas o documento de texto):

```
URL del Storage Account : https://_________________________.dfs.core.windows.net
Nombre del contenedor   : _________________________
SAS Token               : ?sv=2023-...&sig=...
Carpeta Bronze          : ventas/bronze/ (o el nombre indicado por el instructor)
Carpeta Silver          : ventas/silver/ (o el nombre indicado por el instructor)
```

> 💡 **Alternativa sin Azure Storage:** Si no cuentas con acceso a un Storage Account externo, el instructor puede proporcionarte URLs de SAS Token preconfiguradas para el storage de demostración del curso. En ese caso, usa exactamente las URLs proporcionadas en los pasos de creación de Shortcuts.

---

## 6. Instrucciones Paso a Paso

---

### Paso 1 — Crear el Workspace de Microsoft Fabric

**Objetivo:** Aprovisionar un Workspace dedicado con capacidad Trial que servirá como contenedor de todos los artefactos del curso.

#### Instrucciones

1. Abre tu navegador y navega a `https://app.fabric.microsoft.com`. Inicia sesión si se te solicita.

2. En el panel de navegación izquierdo, localiza el ícono de **Workspaces** (ícono de cuadrícula o carpetas apiladas) y haz clic en él.

3. En la parte inferior del panel de Workspaces, haz clic en **+ Nuevo workspace**.

4. En el panel de configuración del Workspace que aparece a la derecha, completa los campos:

   | Campo | Valor |
   |---|---|
   | **Nombre** | `Fabric_MedallionLab_[TusIniciales]` (ejemplo: `Fabric_MedallionLab_JGR`) |
   | **Descripción** | `Workspace del curso de arquitectura Medallion en Microsoft Fabric` |

5. Expande la sección **Avanzado** (o **Licencia**) para verificar la configuración de capacidad:
   - Selecciona **Trial** en el tipo de licencia.
   - Si Trial no aparece como opción, selecciona **Fabric capacity** y elige la capacidad Trial disponible en el desplegable.

6. Haz clic en **Aplicar** o **Crear**.

7. Espera a que el Workspace se cree (generalmente menos de 10 segundos). Serás redirigido automáticamente al nuevo Workspace vacío.

#### Resultado esperado

- El Workspace aparece en el panel izquierdo con el nombre que asignaste.
- La pantalla principal muestra un Workspace vacío con el mensaje "Este workspace no tiene contenido todavía" o similar.
- En la URL del navegador aparece el ID del Workspace: `https://app.fabric.microsoft.com/groups/<workspace-id>/...`

#### Verificación

Confirma que el Workspace usa capacidad Trial:

1. Haz clic en el ícono de configuración del Workspace (⚙️ engranaje) en la parte superior derecha, o accede a **Configuración del workspace** desde el menú `...` junto al nombre del workspace.
2. En la sección **Licencia** o **Premium**, verifica que aparece **Trial** o **Fabric (Trial)** como tipo de capacidad.
3. Si aparece **Pro** o **Free**, el Workspace no tiene capacidad Fabric habilitada. Contacta al instructor antes de continuar.

---

### Paso 2 — Aprovisionar el Lakehouse principal

**Objetivo:** Crear el Lakehouse `lh_ventas` que servirá como repositorio central de datos para todos los laboratorios del curso.

#### Instrucciones

1. Desde el Workspace recién creado, haz clic en el botón **+ Nuevo elemento** (o **+ New item**) en la parte superior de la pantalla.

2. En el panel de selección de elementos, busca y selecciona **Lakehouse**.
   - Si no ves la opción directamente, usa el campo de búsqueda y escribe "Lakehouse".
   - Asegúrate de seleccionar **Lakehouse** (bajo la categoría *Data Engineering*), no *SQL Warehouse* ni *KQL Database*.

3. En el cuadro de diálogo de creación, ingresa el nombre:

   ```
   lh_ventas
   ```

4. Haz clic en **Crear**.

5. Espera a que Fabric aprovisione el Lakehouse. Este proceso toma entre 15 y 45 segundos. Verás una pantalla de carga con el mensaje "Creando Lakehouse..." o similar.

6. Una vez completado, serás redirigido automáticamente al explorador del Lakehouse `lh_ventas`.

#### Resultado esperado

- La interfaz del Lakehouse muestra dos secciones principales en el panel izquierdo:
  - **Tables** — donde se almacenarán las tablas Delta Lake.
  - **Files** — donde se almacenarán archivos sin procesar y Shortcuts.
- En la parte superior de la pantalla aparece el nombre `lh_ventas`.
- En el panel de navegación del Workspace (izquierda), aparecen dos artefactos nuevos: `lh_ventas` (Lakehouse) y `lh_ventas SQL analytics endpoint` (SQL Endpoint).

#### Verificación

1. Confirma que en el explorador del Lakehouse puedes ver las carpetas `Tables` y `Files` en el panel izquierdo.
2. Haz clic en `Tables` — debe estar vacía (sin tablas aún).
3. Haz clic en `Files` — debe estar vacía (sin archivos ni subcarpetas aún).
4. En la esquina superior derecha del explorador del Lakehouse, verifica que existe un botón o enlace que dice **SQL analytics endpoint** — esto confirma que el endpoint fue aprovisionado automáticamente.

---

### Paso 3 — Crear la estructura de carpetas Medallion en la sección Files

**Objetivo:** Organizar manualmente la estructura de directorios dentro de la carpeta `Files` del Lakehouse siguiendo el patrón Medallion (Bronze / Silver / Gold) para establecer las convenciones de almacenamiento del curso.

#### Instrucciones

1. Desde el explorador del Lakehouse `lh_ventas`, asegúrate de estar en la vista **Lakehouse** (no en SQL Endpoint). Verifica esto en el selector de vista en la parte superior derecha.

2. En el panel izquierdo, haz clic derecho sobre la carpeta **Files**.

3. En el menú contextual, selecciona **Nueva subcarpeta** (o **New subfolder**).

4. Escribe el nombre `Bronze` y presiona **Enter** o haz clic en **Crear**.

5. Repite los pasos 2-4 para crear las carpetas:
   - `Silver`
   - `Gold`

6. Dentro de la carpeta `Bronze`, crea las siguientes subcarpetas para organizar los datos por dominio:
   - Haz clic derecho sobre `Bronze` → **Nueva subcarpeta** → `ventas`
   - Haz clic derecho sobre `Bronze` → **Nueva subcarpeta** → `productos`
   - Haz clic derecho sobre `Bronze` → **Nueva subcarpeta** → `clientes`

7. La estructura final en `Files` debe quedar así:

   ```
   Files/
   ├── Bronze/
   │   ├── ventas/
   │   ├── productos/
   │   └── clientes/
   ├── Silver/
   └── Gold/
   ```

#### Resultado esperado

- El panel izquierdo del explorador del Lakehouse muestra la jerarquía de carpetas bajo `Files`.
- Las carpetas `Bronze`, `Silver` y `Gold` aparecen como subdirectorios de `Files`.
- Dentro de `Bronze` se visualizan tres subcarpetas: `ventas`, `productos` y `clientes`.

#### Verificación

1. Haz clic en cada carpeta para confirmar que se creó correctamente y está vacía.
2. Verifica que los nombres de las carpetas están escritos correctamente (respetan mayúsculas/minúsculas): `Bronze`, `Silver`, `Gold`.

> 📝 **Convención de nombres:** A lo largo del curso usaremos esta estructura de carpetas como referencia. Los pipelines del Lab 02 depositarán datos en `Bronze/ventas/`, los Notebooks del Lab 03 leerán desde `Bronze/` y escribirán en `Silver/`, y así sucesivamente.

---

### Paso 4 — Crear el primer Shortcut hacia Azure Data Lake Storage Gen2 (capa Bronze)

**Objetivo:** Configurar un Shortcut que apunte al contenedor de datos de ventas en ADLS Gen2 proporcionado por el instructor, habilitando acceso federado a los datos sin copiarlos a OneLake.

#### Instrucciones

1. Desde el explorador del Lakehouse `lh_ventas`, en el panel izquierdo, haz clic derecho sobre la carpeta **Files**.

   > 💡 También puedes crear el Shortcut dentro de una subcarpeta específica. Para este ejercicio lo crearemos directamente bajo `Files` para mayor visibilidad.

2. En el menú contextual, selecciona **Nuevo Shortcut** (o **New shortcut**).

3. En el panel de selección de origen del Shortcut, verás las opciones disponibles:
   - **Microsoft OneLake** (interno)
   - **Azure Data Lake Storage Gen2**
   - **Azure Blob Storage**
   - **Amazon S3**
   - **Google Cloud Storage**
   - **Dataverse**

   Selecciona **Azure Data Lake Storage Gen2**.

4. En la pantalla de configuración de conexión, completa los campos con los datos proporcionados por el instructor:

   | Campo | Valor |
   |---|---|
   | **URL** | `https://<nombre_cuenta>.dfs.core.windows.net` |
   | **Tipo de autenticación** | Selecciona **Shared Access Signature (SAS)** |
   | **Token SAS** | Pega el SAS Token proporcionado por el instructor (comienza con `?sv=` o `sv=`) |

   > ⚠️ **Importante:** El SAS Token debe tener permisos de **lectura (r)** y **listado (l)** sobre el contenedor. Si el token comienza con `?`, omite el signo de interrogación al pegarlo si el campo ya lo incluye, o inclúyelo si el campo espera la cadena completa.

5. Haz clic en **Siguiente** (o **Next**).

6. Fabric validará las credenciales y mostrará el explorador de contenedores del Storage Account. Navega hasta el contenedor indicado por el instructor (por ejemplo: `ventas-raw`).

7. Dentro del contenedor, navega hasta la carpeta de datos de ventas (por ejemplo: `ventas/bronze/` o la ruta indicada por el instructor).

8. Selecciona la carpeta que deseas mapear como Shortcut.

9. En el campo **Nombre del Shortcut**, escribe:
   ```
   raw_ventas_ext
   ```

10. Haz clic en **Crear** (o **Create**).

11. Espera a que Fabric valide y cree el Shortcut (5-15 segundos).

#### Resultado esperado

- En el explorador del Lakehouse, bajo `Files`, aparece una nueva entrada llamada `raw_ventas_ext` con un ícono especial que indica que es un Shortcut (generalmente una flecha curva o un ícono de enlace, diferente al ícono de carpeta normal).
- Al hacer clic en `raw_ventas_ext`, se muestran los archivos del contenedor externo como si estuvieran almacenados localmente en el Lakehouse.
- Los archivos de datos (CSV, Parquet, etc.) son visibles en el panel derecho del explorador.

#### Verificación

1. Haz clic en el Shortcut `raw_ventas_ext` para expandirlo.
2. Confirma que puedes ver los archivos de datos del instructor (archivos `.csv` o `.parquet` con nombres relacionados a ventas).
3. Si el Shortcut aparece con un ícono de error (❌) o no muestra contenido, verifica las credenciales y consulta la sección de **Solución de Problemas** al final de este laboratorio.

---

### Paso 5 — Crear el segundo Shortcut hacia Azure Blob Storage (datos de referencia)

**Objetivo:** Configurar un segundo Shortcut apuntando a datos de referencia (catálogo de productos o clientes) almacenados en Azure Blob Storage, demostrando la capacidad de conectar múltiples fuentes heterogéneas desde un mismo Lakehouse.

#### Instrucciones

1. Desde el explorador del Lakehouse `lh_ventas`, haz clic derecho sobre la carpeta **Files** → **Nuevo Shortcut**.

2. En el panel de selección de origen, selecciona **Azure Blob Storage**.

3. Completa los campos de conexión con los datos del segundo recurso de almacenamiento proporcionado por el instructor:

   | Campo | Valor |
   |---|---|
   | **URL** | `https://<nombre_cuenta>.blob.core.windows.net` |
   | **Tipo de autenticación** | **Shared Access Signature (SAS)** |
   | **Token SAS** | SAS Token del segundo recurso (proporcionado por el instructor) |

   > 📝 **Nota:** Si el instructor solo proporcionó un Storage Account, usa el mismo Storage Account pero apuntando a un contenedor diferente (por ejemplo: `productos-ref` o `clientes-ref`). Si se usa el mismo Storage Account, selecciona **Azure Data Lake Storage Gen2** nuevamente con las mismas credenciales pero navegando a un contenedor diferente.

4. Haz clic en **Siguiente**.

5. Navega hasta el contenedor de datos de referencia indicado por el instructor (por ejemplo: `productos-ref`).

6. Selecciona la carpeta raíz del contenedor o la carpeta específica indicada.

7. En el campo **Nombre del Shortcut**, escribe:
   ```
   raw_productos_ext
   ```

8. Haz clic en **Crear**.

#### Resultado esperado

- Bajo `Files`, aparece un segundo Shortcut llamado `raw_productos_ext` con el mismo ícono de enlace.
- Al expandirlo, se visualizan los archivos de referencia de productos o clientes del Storage Account externo.
- Ambos Shortcuts (`raw_ventas_ext` y `raw_productos_ext`) coexisten bajo `Files` sin conflictos.

#### Verificación

1. Confirma que ambos Shortcuts son visibles bajo `Files` en el panel izquierdo.
2. Haz clic en cada uno para verificar que muestran contenido diferente correspondiente a sus fuentes respectivas.
3. La estructura del explorador del Lakehouse debe verse similar a:

   ```
   Files/
   ├── Bronze/
   │   ├── ventas/
   │   ├── productos/
   │   └── clientes/
   ├── Silver/
   ├── Gold/
   ├── raw_ventas_ext     ← Shortcut (ícono de flecha/enlace)
   └── raw_productos_ext  ← Shortcut (ícono de flecha/enlace)
   ```

---

### Paso 6 — Explorar los datos del Shortcut desde el SQL Endpoint

**Objetivo:** Utilizar el SQL Endpoint del Lakehouse para ejecutar consultas SQL sobre los datos accedidos mediante los Shortcuts, verificando que la conectividad funciona correctamente sin necesidad de copiar los datos.

#### Instrucciones

1. En la parte superior derecha del explorador del Lakehouse, haz clic en el selector de vista y selecciona **SQL analytics endpoint** (o accede desde el Workspace haciendo clic en `lh_ventas SQL analytics endpoint`).

2. La interfaz cambia al editor SQL del Endpoint. En el panel izquierdo verás las tablas y esquemas disponibles.

   > 📝 **Nota:** Los Shortcuts bajo `Files` no aparecen automáticamente como tablas SQL. Para consultarlos mediante SQL necesitas referenciarlos con funciones de lectura o crear tablas externas. En este paso usaremos el Notebook de Spark para la validación principal. Sin embargo, si los Shortcuts apuntan a carpetas con archivos **Delta Lake** (formato `.parquet` + `_delta_log`), sí pueden aparecer como tablas en el SQL Endpoint.

3. En el editor SQL, escribe la siguiente consulta para verificar el estado del endpoint y listar los esquemas disponibles:

   ```sql
   -- Verificar los esquemas disponibles en el Lakehouse
   SELECT TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE
   FROM INFORMATION_SCHEMA.TABLES
   ORDER BY TABLE_SCHEMA, TABLE_NAME;
   ```

4. Haz clic en **Ejecutar** (▶️) o presiona `F5`.

5. Observa los resultados. En este punto el Lakehouse no tiene tablas Delta Lake creadas aún (eso ocurrirá en Labs 02 y 03), por lo que el resultado puede estar vacío o mostrar solo el esquema `dbo`.

6. Para explorar los archivos del Shortcut directamente desde el SQL Endpoint usando `OPENROWSET`, ejecuta la siguiente consulta adaptada a los archivos de tu Shortcut:

   > ⚠️ **Importante:** La función `OPENROWSET` puede no estar disponible en todos los SQL Endpoints de Fabric en versiones actuales. Si obtienes un error de función no reconocida, salta al **Paso 7** donde se realiza la validación completa desde un Notebook de Spark.

   ```sql
   -- Consultar archivos CSV desde el Shortcut usando OPENROWSET
   -- Ajusta el path según el nombre de tu Shortcut y los archivos disponibles
   SELECT TOP 10 *
   FROM OPENROWSET(
       BULK 'Files/raw_ventas_ext/*.csv',
       FORMAT = 'CSV',
       HEADER_ROW = TRUE
   ) AS datos_ventas;
   ```

7. Si los archivos son de formato Parquet, usa:

   ```sql
   -- Consultar archivos Parquet desde el Shortcut
   SELECT TOP 10 *
   FROM OPENROWSET(
       BULK 'Files/raw_ventas_ext/*.parquet',
       FORMAT = 'PARQUET'
   ) AS datos_ventas;
   ```

8. Ejecuta la consulta y observa los resultados en el panel inferior.

#### Resultado esperado

- La consulta `INFORMATION_SCHEMA.TABLES` retorna un resultado (posiblemente vacío si no hay tablas Delta aún, lo cual es esperado en este punto).
- Si `OPENROWSET` está disponible y los archivos son accesibles, la consulta retorna las primeras 10 filas del dataset de ventas con las columnas del archivo CSV/Parquet.
- Si `OPENROWSET` no está disponible, el sistema retorna un mensaje de error de sintaxis — esto es normal y la validación se completa en el siguiente paso con Spark.

#### Verificación

- Confirma que el SQL Endpoint responde a consultas sin errores de conectividad.
- Si `OPENROWSET` funcionó, documenta los nombres de las columnas retornadas para usarlos en el Lab 03.

---

### Paso 7 — Validar la conectividad del Shortcut desde un Notebook de Spark

**Objetivo:** Confirmar de forma definitiva que los Shortcuts están correctamente configurados ejecutando código PySpark que lee los datos externos y muestra las primeras filas del dataset.

#### Instrucciones

1. Regresa a la vista **Lakehouse** del `lh_ventas` (usa el selector de vista en la parte superior).

2. En el menú superior o desde el botón **+ Nuevo elemento** del Workspace, crea un nuevo **Notebook**:
   - Haz clic en **Abrir Notebook** → **Nuevo Notebook**, o bien
   - Ve al Workspace → **+ Nuevo elemento** → **Notebook**.

3. En el Notebook, asegúrate de que el Lakehouse `lh_ventas` está adjunto como fuente de datos. Si no lo está:
   - En el panel izquierdo del Notebook, haz clic en **Agregar Lakehouse** (o el ícono de base de datos).
   - Selecciona **Lakehouse existente** → elige `lh_ventas` → **Agregar**.

4. En la primera celda del Notebook, escribe y ejecuta el siguiente código para verificar el Shortcut de ventas:

   ```python
   # Celda 1: Verificar acceso al Shortcut raw_ventas_ext
   # El Shortcut se accede con la misma ruta que cualquier carpeta de Files
   
   ruta_shortcut_ventas = "Files/raw_ventas_ext/"
   
   # Listar los archivos disponibles en el Shortcut
   archivos = mssparkutils.fs.ls(ruta_shortcut_ventas)
   
   print(f"Archivos encontrados en el Shortcut 'raw_ventas_ext': {len(archivos)}")
   for archivo in archivos:
       print(f"  - {archivo.name} | Tamaño: {archivo.size} bytes | Es directorio: {archivo.isDir}")
   ```

5. Haz clic en el botón **Ejecutar celda** (▶️) o presiona `Shift + Enter`.

   > ⏳ **Nota:** La primera ejecución puede tardar entre 1 y 5 minutos mientras se inicializa el clúster Spark. Verás el mensaje "Iniciando sesión de Spark..." — esto es normal.

6. Una vez completada la ejecución, agrega una nueva celda y escribe el siguiente código para leer los datos:

   ```python
   # Celda 2: Leer el dataset de ventas desde el Shortcut
   # Fabric resuelve el Shortcut en tiempo real; los datos NO se copian a OneLake
   
   ruta_shortcut_ventas = "Files/raw_ventas_ext/"
   
   # Detectar el formato de los archivos (ajustar si son .parquet)
   df_ventas = spark.read.format("csv") \
       .option("header", "true") \
       .option("inferSchema", "true") \
       .option("encoding", "UTF-8") \
       .load(ruta_shortcut_ventas)
   
   # Mostrar las primeras 5 filas para confirmar conectividad
   print(f"Total de registros leídos: {df_ventas.count()}")
   print(f"Número de columnas: {len(df_ventas.columns)}")
   print(f"Columnas detectadas: {df_ventas.columns}")
   df_ventas.show(5, truncate=False)
   ```

7. Si los archivos del Shortcut son formato Parquet en lugar de CSV, usa esta versión:

   ```python
   # Celda 2 (alternativa para archivos Parquet):
   ruta_shortcut_ventas = "Files/raw_ventas_ext/"
   
   df_ventas = spark.read.format("parquet") \
       .load(ruta_shortcut_ventas)
   
   print(f"Total de registros leídos: {df_ventas.count()}")
   df_ventas.show(5, truncate=False)
   ```

8. Agrega una tercera celda para validar el segundo Shortcut:

   ```python
   # Celda 3: Verificar el Shortcut raw_productos_ext
   
   ruta_shortcut_productos = "Files/raw_productos_ext/"
   
   df_productos = spark.read.format("csv") \
       .option("header", "true") \
       .option("inferSchema", "true") \
       .load(ruta_shortcut_productos)
   
   print(f"Total de registros de productos: {df_productos.count()}")
   df_productos.show(5, truncate=False)
   ```

9. Ejecuta todas las celdas haciendo clic en **Ejecutar todo** (▶▶) en el menú superior del Notebook.

10. Una vez completada la ejecución, **guarda el Notebook** con el nombre:
    ```
    nb_validacion_shortcuts
    ```
    Usa `Ctrl + S` o el botón **Guardar** en la barra superior.

#### Resultado esperado

- **Celda 1:** Lista los archivos del Shortcut con sus nombres y tamaños. Ejemplo de salida:
  ```
  Archivos encontrados en el Shortcut 'raw_ventas_ext': 3
    - ventas_2023_Q1.csv | Tamaño: 245760 bytes | Es directorio: False
    - ventas_2023_Q2.csv | Tamaño: 312448 bytes | Es directorio: False
    - ventas_2023_Q3.csv | Tamaño: 289792 bytes | Es directorio: False
  ```

- **Celda 2:** Muestra el conteo total de registros, los nombres de las columnas detectadas automáticamente y las primeras 5 filas del dataset de ventas. Ejemplo:
  ```
  Total de registros leídos: 15420
  Número de columnas: 8
  Columnas detectadas: ['fecha_venta', 'id_producto', 'id_cliente', 'cantidad', 'precio_unitario', 'descuento', 'region', 'canal_venta']
  +------------+-----------+-----------+--------+----------------+---------+--------+------------+
  |fecha_venta |id_producto|id_cliente |cantidad|precio_unitario |descuento|region  |canal_venta |
  +------------+-----------+-----------+--------+----------------+---------+--------+------------+
  |2023-01-05  |PROD-001   |CLI-00234  |2       |150.00          |0.05     |Norte   |Online      |
  ...
  ```

- **Celda 3:** Muestra el conteo y primeras filas del catálogo de productos.

#### Verificación

- Confirma que **ninguna celda** termina con un error rojo.
- Verifica que el conteo de registros es mayor a 0 en ambos Shortcuts.
- Confirma que los nombres de las columnas tienen sentido para el dominio de ventas (fechas, IDs, cantidades, precios).

---

## 7. Validación General del Laboratorio

Una vez completados todos los pasos, realiza las siguientes verificaciones finales para confirmar que el laboratorio fue completado exitosamente:

### Lista de verificación final

| # | Verificación | Cómo confirmar | ✅ / ❌ |
|---|---|---|---|
| 1 | Workspace creado con capacidad Trial | Configuración del Workspace → Licencia muestra "Trial" | |
| 2 | Lakehouse `lh_ventas` aprovisionado | Visible en el Workspace con ícono de Lakehouse | |
| 3 | SQL Endpoint aprovisionado automáticamente | `lh_ventas SQL analytics endpoint` visible en el Workspace | |
| 4 | Estructura de carpetas Medallion creada | `Files/Bronze/`, `Files/Silver/`, `Files/Gold/` visibles en el explorador | |
| 5 | Subcarpetas `ventas`, `productos`, `clientes` bajo `Bronze/` | Expandir `Bronze` en el explorador del Lakehouse | |
| 6 | Shortcut `raw_ventas_ext` creado y funcional | Visible en `Files/` con ícono de Shortcut; muestra archivos al expandir | |
| 7 | Shortcut `raw_productos_ext` creado y funcional | Visible en `Files/` con ícono de Shortcut; muestra archivos al expandir | |
| 8 | Notebook `nb_validacion_shortcuts` ejecutado sin errores | Todas las celdas muestran resultados verdes en el historial de ejecución | |
| 9 | Datos leídos desde Shortcuts muestran registros reales | `df_ventas.count()` > 0 y `df_productos.count()` > 0 | |

### Resumen de artefactos creados

Al finalizar este laboratorio, tu Workspace debe contener los siguientes artefactos:

| Artefacto | Tipo | Nombre | Usado en |
|---|---|---|---|
| Workspace | Contenedor | `Fabric_MedallionLab_[Iniciales]` | Todos los labs |
| Lakehouse | Data Engineering | `lh_ventas` | Labs 02, 03, 04, 05 |
| SQL Endpoint | Análisis SQL | `lh_ventas SQL analytics endpoint` | Labs 03, 04 |
| Notebook de validación | Data Engineering | `nb_validacion_shortcuts` | Solo Lab 01 |
| Shortcut ADLS Gen2 | Conectividad | `raw_ventas_ext` | Labs 02, 03 |
| Shortcut Blob Storage | Conectividad | `raw_productos_ext` | Labs 02, 03 |

---

## 8. Solución de Problemas

### Problema 1 — El Shortcut muestra error de autenticación o no lista archivos

**Síntoma:**
- Al hacer clic en el Shortcut `raw_ventas_ext` o `raw_productos_ext`, aparece un ícono de error (❌) o un mensaje como `"Error al acceder al recurso"`, `"AuthorizationPermissionMismatch"` o `"The specified resource does not exist"`.
- En el Notebook, la celda con `mssparkutils.fs.ls()` lanza una excepción `AnalysisException` o `FileNotFoundException`.

**Causa probable:**
El SAS Token proporcionado puede tener uno o más de los siguientes problemas:
- El token ha expirado.
- El token no tiene permisos de **lectura (r)** y/o **listado (l)** sobre el contenedor específico.
- La URL del Storage Account fue ingresada incorrectamente (por ejemplo, usando el endpoint `.blob.core.windows.net` para un recurso ADLS Gen2 que requiere `.dfs.core.windows.net`).
- El nombre del contenedor en la URL no coincide con el contenedor real en el Storage Account.

**Solución:**
1. Verifica la URL del Storage Account: para ADLS Gen2 debe terminar en `.dfs.core.windows.net`; para Blob Storage estándar debe terminar en `.blob.core.windows.net`.
2. Solicita al instructor un nuevo SAS Token y confirma que incluye los permisos `r` (Read) y `l` (List) en su cadena (busca `sp=rl` o `sp=racwdlmeopi` en el token).
3. Elimina el Shortcut con error: haz clic derecho sobre él en el explorador → **Eliminar**. Recrea el Shortcut desde el Paso 4 con las credenciales corregidas.
4. Verifica que la fecha de expiración del token (`se=YYYY-MM-DD`) sea posterior a la fecha actual.
5. Si el problema persiste, solicita al instructor las URLs de SAS Token preconfiguradas del storage de demostración del curso.

---

### Problema 2 — El clúster Spark no inicia o la celda del Notebook queda en estado "En ejecución" indefinidamente

**Síntoma:**
- Al ejecutar la primera celda del Notebook, el estado muestra "Iniciando sesión de Spark..." por más de 15 minutos sin avanzar.
- El indicador de progreso de la celda gira indefinidamente sin mostrar resultados.
- Aparece un mensaje de error como `"SparkException: Spark session failed to start"` o `"The Spark cluster is not available"`.

**Causa probable:**
La capacidad Trial de Microsoft Fabric tiene recursos de cómputo Spark compartidos entre todos los usuarios del tenant. Durante horas de alta demanda (especialmente en entornos de clase con múltiples estudiantes ejecutando Notebooks simultáneamente), el aprovisionamiento del clúster Spark puede demorar significativamente o fallar por falta de capacidad disponible en ese momento.

**Solución:**
1. **Espera adicional:** Si el clúster lleva menos de 10 minutos iniciando, espera. El tiempo de inicialización en capacidad Trial puede ser de hasta 10-15 minutos en condiciones de alta carga.
2. **Cancelar y reintentar:** Si llevas más de 15 minutos esperando, haz clic en el botón **Cancelar** (⏹) de la celda en ejecución, espera 2 minutos y vuelve a ejecutar.
3. **Verificar el estado de la capacidad:** Ve a `https://app.fabric.microsoft.com` → Configuración del Workspace → Capacidad. Si la capacidad muestra un indicador de "Throttled" o "Pausada", espera 10-15 minutos antes de reintentar.
4. **Escalonamiento en clase:** El instructor puede coordinar que los estudiantes ejecuten los Notebooks en grupos de 3-5 personas para evitar saturar la capacidad Trial compartida.
5. **Verificar la sesión de Spark:** En la barra superior del Notebook, verifica que el tipo de sesión sea **Spark** (no **Python** o **SQL**). Si está configurado incorrectamente, cámbialo desde el menú de configuración del Notebook.
6. Si el problema persiste después de 3 intentos, documenta los resultados de los Pasos 1-6 y continúa con la validación mediante el SQL Endpoint (Paso 6). La validación completa de Spark puede realizarse al inicio del Lab 02.

---

## 9. Limpieza de Recursos

> 🚨 **ADVERTENCIA CRÍTICA:** **NO elimines ningún recurso creado en este laboratorio.** El Workspace, el Lakehouse `lh_ventas` y los Shortcuts son consumidos directamente por los Labs 02, 03, 04 y 05. Eliminar cualquiera de estos artefactos interrumpirá la secuencia completa del curso.

### Lo que NO debes hacer al finalizar este laboratorio:
- ❌ No elimines el Workspace `Fabric_MedallionLab_[Iniciales]`
- ❌ No elimines el Lakehouse `lh_ventas`
- ❌ No elimines los Shortcuts `raw_ventas_ext` ni `raw_productos_ext`
- ❌ No elimines las carpetas de la estructura Medallion

### Lo que SÍ puedes hacer:
- ✅ Cerrar el Notebook `nb_validacion_shortcuts` (los resultados quedan guardados)
- ✅ Cerrar las pestañas del navegador relacionadas con Fabric
- ✅ Cerrar sesión en el portal de Fabric si terminas la sesión de trabajo

### Limpieza final (solo al completar TODOS los laboratorios del curso):

Una vez finalizado el Lab 05, el instructor proporcionará una guía de limpieza completa. El proceso general será:

1. Navegar al Workspace `Fabric_MedallionLab_[Iniciales]`.
2. Acceder a **Configuración del Workspace** → **Otros** → **Eliminar este workspace**.
3. Confirmar la eliminación escribiendo el nombre del Workspace.

Esto eliminará todos los artefactos y liberará la capacidad Trial para su reutilización.

---

## 10. Resumen

### Conceptos aplicados en este laboratorio

En este laboratorio aplicaste los conceptos fundamentales de la arquitectura Lakehouse en Microsoft Fabric:

| Concepto | Aplicación en el laboratorio |
|---|---|
| **OneLake como almacenamiento unificado** | Todo el Lakehouse `lh_ventas` reside en OneLake; las carpetas `Files` y `Tables` son parte del mismo almacén jerárquico |
| **Arquitectura Medallion** | Creaste manualmente la estructura `Bronze/Silver/Gold` que guiará todo el flujo de datos del curso |
| **Shortcuts como acceso federado** | Los datos del instructor en ADLS Gen2/Blob Storage son accesibles desde el Lakehouse sin copiarlos físicamente a OneLake |
| **SQL Endpoint automático** | Fabric aprovisionó un endpoint SQL de solo lectura al crear el Lakehouse, sin configuración de infraestructura |
| **PySpark para validación** | Usaste `mssparkutils.fs.ls()` y `spark.read` para confirmar que los Shortcuts resuelven correctamente hacia los datos externos |

### Puntos clave para recordar

1. **Un Lakehouse = tres componentes automáticos:** almacenamiento Delta Lake (Files + Tables), SQL Endpoint y Semantic Model predeterminado.
2. **Los Shortcuts NO copian datos:** cada lectura resuelve la referencia en tiempo real hacia el origen externo. Esto garantiza datos siempre actualizados y elimina costos de duplicación.
3. **La seguridad de los Shortcuts se hereda del Workspace:** quién tiene acceso al Lakehouse tiene acceso a los datos del Shortcut, independientemente de los permisos en el origen.
4. **La estructura de carpetas Medallion es una convención:** Fabric no impone esta estructura, pero adoptarla facilita la gobernanza, el linaje de datos y la colaboración en equipo.
5. **El SQL Endpoint es de solo lectura:** no puedes escribir datos directamente a través del SQL Endpoint; para escribir, debes usar Notebooks (Spark) o Pipelines (Data Factory).

### Próximos pasos

Con el Lakehouse configurado, la estructura Medallion establecida y los Shortcuts como puerta de entrada a las fuentes externas, estás listo para el siguiente laboratorio. En el **Lab 02**, construirás un **pipeline de ingesta automatizada** usando Data Factory dentro de Microsoft Fabric para orquestar la carga de datos desde los orígenes conectados mediante los Shortcuts hacia tablas Delta Lake estructuradas en la capa Bronze, siguiendo los principios de la arquitectura Medallion.

### Recursos adicionales

| Recurso | URL |
|---|---|
| Documentación oficial — Microsoft Fabric Lakehouse | https://learn.microsoft.com/es-es/fabric/data-engineering/lakehouse-overview |
| Introducción a OneLake | https://learn.microsoft.com/es-es/fabric/onelake/onelake-overview |
| Creación y uso de Shortcuts en OneLake | https://learn.microsoft.com/es-es/fabric/onelake/onelake-shortcuts |
| Shortcuts hacia Azure Data Lake Storage Gen2 | https://learn.microsoft.com/es-es/fabric/onelake/create-adls-shortcut |
| Seguridad y control de acceso en Microsoft Fabric | https://learn.microsoft.com/es-es/fabric/security/security-overview |

---

*Lab 01-00-01 — Versión 1.0 | Curso: Arquitectura Medallion en Microsoft Fabric | Duración: 75 minutos*
