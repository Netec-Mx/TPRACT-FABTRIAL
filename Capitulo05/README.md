# Eficiencia Operativa: Configuración de alerta automática basada en eventos

## Metadatos del Laboratorio

| Atributo | Valor |
|---|---|
| **Duración estimada** | 75 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (Apply) |
| **Laboratorio número** | 05 de 05 |
| **Dependencia previa** | Labs 01-00-01, 02-00-01, 03-00-01 y 04-00-01 completados |

---

## Descripción General

Este laboratorio cierra el ciclo operativo de la arquitectura Medallion implementando capacidades de observabilidad y respuesta automática mediante **Data Activator** (anteriormente Reflex) en Microsoft Fabric. El estudiante conectará Data Activator a dos fuentes de datos —el informe de Power BI creado en el Lab 4 y las tablas Delta del Lakehouse— para monitorear KPIs de negocio y métricas operacionales del pipeline. Se configurarán reglas de detección con condiciones basadas en umbrales de negocio y se definirán acciones de respuesta automática que incluyen notificaciones por email, mensajes en Microsoft Teams y activación opcional de pipelines correctivos. Al finalizar, el estudiante habrá implementado un flujo completo de observabilidad: desde la detección del evento hasta la respuesta automatizada, sin intervención humana.

---

## Objetivos de Aprendizaje

Al completar este laboratorio, el estudiante será capaz de:

- [ ] Configurar un Reflex de Data Activator conectado a un informe de Power BI para monitorear KPIs de ventas en tiempo real
- [ ] Definir Objetos, Propiedades y Reglas (Triggers) en Data Activator con condiciones basadas en umbrales de negocio
- [ ] Configurar acciones de respuesta automática (email y Microsoft Teams) que se disparan al detectar una anomalía
- [ ] Simular un escenario de alerta para verificar el funcionamiento end-to-end del sistema de observabilidad
- [ ] Documentar el runbook operativo resultante que describe el flujo evento → detección → respuesta

---

## Prerrequisitos

### Conocimiento previo requerido

| Área | Nivel requerido |
|---|---|
| Labs 01 al 04 completados en orden | **Obligatorio** — este lab consume artefactos de todos los anteriores |
| Conceptos de monitoreo operacional y umbrales de alerta | Básico |
| Navegación en Microsoft Fabric (Workspaces, Lakehouse, Power BI) | Intermedio |
| Conceptos de event-driven architecture (eventos, triggers, acciones) | Básico |
| Acceso a Microsoft Teams o cuenta de email activa | **Obligatorio** para recibir notificaciones de prueba |

### Acceso y recursos requeridos

| Recurso | Estado requerido |
|---|---|
| Microsoft Fabric Trial activo | ✅ Activo y con capacidad disponible |
| Workspace `FabricLakehouseLab` (creado en Lab 01) | ✅ Existente con todos los artefactos previos |
| Lakehouse `SalesLakehouse` con tablas Silver | ✅ Tablas `silver_ventas` y `silver_productos` disponibles |
| Informe de Power BI `Reporte_Ventas_Silver` (Lab 04) | ✅ Publicado en el Workspace de Fabric |
| Pipeline `Pipeline_Ingesta_Bronze` (Lab 02) | ✅ Existente y ejecutable |
| Microsoft Teams (canal o chat personal) | ✅ Con permisos para recibir mensajes de aplicaciones |
| Data Activator disponible en la región del tenant | ✅ Verificar antes de comenzar |

> ⚠️ **VERIFICACIÓN PREVIA CRÍTICA**: Antes de comenzar, confirme que Data Activator esté disponible en su región de tenant. Navegue a su Workspace en Fabric, haga clic en **+ New** y busque **Reflex** o **Data Activator** en la lista de artefactos. Si no aparece, consulte al instructor para alternativas según su región.

---

## Entorno del Laboratorio

### Hardware mínimo recomendado

| Componente | Mínimo | Recomendado |
|---|---|---|
| Procesador | Intel Core i5 8ª gen / AMD Ryzen 5 | Intel Core i7 / AMD Ryzen 7 |
| RAM | 8 GB | 16 GB |
| Almacenamiento libre | 500 MB | 1 GB |
| Resolución de pantalla | 1366×768 | 1920×1080 |
| Conexión a Internet | 10 Mbps | 25 Mbps |

### Software requerido

| Software | Versión mínima | Propósito en este lab |
|---|---|---|
| Microsoft Edge o Google Chrome | 110 o superior | Acceso a Microsoft Fabric |
| Microsoft Fabric Trial | Trial activo | Plataforma principal |
| Power BI Desktop | Mayo 2024 o superior | Verificación del modelo semántico |
| Microsoft Teams | Cualquier versión reciente | Recepción de alertas |

### Arquitectura de referencia del laboratorio

```
┌─────────────────────────────────────────────────────────────┐
│                    MICROSOFT FABRIC                          │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │  Lakehouse   │    │   Power BI   │    │ Data Activator│  │
│  │  (OneLake)   │───▶│   Report     │───▶│   (Reflex)    │  │
│  │              │    │              │    │               │  │
│  │ silver_ventas│    │Reporte_Ventas│    │  Objects      │  │
│  │ silver_prods │    │   _Silver    │    │  Properties   │  │
│  └──────────────┘    └──────────────┘    │  Triggers     │  │
│                                          └───────┬───────┘  │
└──────────────────────────────────────────────────┼──────────┘
                                                   │
                    ┌──────────────────────────────┼──────┐
                    ▼                              ▼      ▼
              📧 Email                      💬 Teams  ⚙️ Pipeline
           Notificación                    Mensaje    Re-ejecución
```

---

## Instrucciones Paso a Paso

---

### Paso 1: Verificación del entorno y preparación de métricas en el Lakehouse

**Objetivo**: Confirmar que todos los artefactos de los labs anteriores están disponibles y crear una vista SQL en el Lakehouse que exponga las métricas operacionales que Data Activator monitoreará.

#### Instrucciones

1. Abra su navegador y acceda a [https://app.fabric.microsoft.com](https://app.fabric.microsoft.com). Inicie sesión con sus credenciales del tenant.

2. En el panel izquierdo, haga clic en **Workspaces** y seleccione **FabricLakehouseLab**.

3. Verifique que los siguientes artefactos existen en el Workspace. Anote cualquier elemento faltante antes de continuar:

   | Artefacto | Tipo | Estado esperado |
   |---|---|---|
   | `SalesLakehouse` | Lakehouse | ✅ Con tablas silver_ventas y silver_productos |
   | `Pipeline_Ingesta_Bronze` | Data Pipeline | ✅ Con al menos 1 ejecución exitosa |
   | `Reporte_Ventas_Silver` | Power BI Report | ✅ Publicado y funcional |
   | `Modelo_Ventas_DirectLake` | Semantic Model | ✅ En modo Direct Lake |

4. Haga clic en **SalesLakehouse** para abrirlo.

5. En el panel superior derecho del Lakehouse, cambie la vista a **SQL analytics endpoint** haciendo clic en el selector de modo (esquina superior derecha, donde dice "Lakehouse").

6. En el editor SQL del SQL Analytics Endpoint, abra una nueva consulta y ejecute el siguiente código para verificar que las tablas Silver tienen datos:

```sql
-- Verificación de datos en tablas Silver
SELECT 
    'silver_ventas' AS tabla,
    COUNT(*) AS total_registros,
    MAX(fecha_venta) AS ultima_fecha,
    SUM(monto_total) AS venta_total
FROM silver_ventas

UNION ALL

SELECT 
    'silver_productos' AS tabla,
    COUNT(*) AS total_registros,
    MAX(fecha_actualizacion) AS ultima_fecha,
    NULL AS venta_total
FROM silver_productos;
```

7. Confirme que ambas tablas retornan registros. Si alguna tabla está vacía, detenga el laboratorio y complete los labs anteriores.

8. Ahora cree una vista que exponga las métricas de ventas diarias que servirán como base para las alertas. Ejecute el siguiente script:

```sql
-- Creación de vista de métricas diarias para monitoreo
-- Esta vista será la fuente de datos para las alertas operacionales

CREATE OR ALTER VIEW vw_metricas_ventas_diarias AS
SELECT
    CAST(fecha_venta AS DATE)          AS fecha,
    COUNT(*)                           AS total_transacciones,
    SUM(monto_total)                   AS venta_total_dia,
    AVG(monto_total)                   AS ticket_promedio,
    COUNT(DISTINCT id_cliente)         AS clientes_unicos,
    COUNT(DISTINCT id_producto)        AS productos_vendidos
FROM silver_ventas
GROUP BY CAST(fecha_venta AS DATE);
```

9. Ejecute una consulta de validación sobre la vista recién creada:

```sql
-- Validación de la vista de métricas
SELECT TOP 10 *
FROM vw_metricas_ventas_diarias
ORDER BY fecha DESC;
```

10. Tome nota del valor de `venta_total_dia` del día más reciente. Lo necesitará para definir el umbral de alerta en el Paso 3.

#### Salida esperada

La consulta de verificación devuelve dos filas con conteos de registros mayores a cero. La vista `vw_metricas_ventas_diarias` se crea exitosamente y retorna filas con fechas, montos y conteos de transacciones.

#### Verificación

✅ Ambas tablas Silver contienen registros  
✅ La vista `vw_metricas_ventas_diarias` existe y retorna datos  
✅ Todos los artefactos del Workspace están presentes  

---

### Paso 2: Creación del artefacto Data Activator (Reflex) en el Workspace

**Objetivo**: Crear un nuevo artefacto de tipo Reflex (Data Activator) en el Workspace de Fabric que servirá como contenedor para todos los objetos de monitoreo del laboratorio.

#### Instrucciones

1. Regrese al Workspace **FabricLakehouseLab** haciendo clic en su nombre en la ruta de navegación (breadcrumb) superior.

2. En la barra superior del Workspace, haga clic en el botón **+ New** (Nuevo).

3. En el panel de creación de artefactos, busque **Reflex** en el campo de búsqueda. Si no aparece como "Reflex", busque también **Data Activator**.

   > 📝 **Nota**: Microsoft Fabric está en evolución continua. El artefacto puede aparecer como "Reflex" o "Data Activator" dependiendo de la versión de la interfaz en su tenant. Ambos nombres se refieren al mismo componente.

4. Haga clic en la tarjeta **Reflex** para iniciar la creación.

5. En el diálogo de nombre, ingrese exactamente:
   ```
   Alertas_Operacionales_Ventas
   ```

6. Haga clic en **Create** (Crear).

7. Espere a que el artefacto se inicialice. Verá la interfaz de Data Activator con tres paneles principales:
   - **Panel izquierdo**: árbol de Objetos (Objects)
   - **Panel central**: editor de Propiedades (Properties) y Reglas (Rules/Triggers)
   - **Panel derecho**: configuración de detalles

8. Familiarícese con la interfaz identificando los siguientes elementos:
   - Botón **Get data** (Obtener datos) en la barra superior — para conectar fuentes
   - Sección **Objects** en el panel izquierdo — donde aparecerán los objetos de monitoreo
   - Área de trabajo central — donde se definirán propiedades y reglas

#### Salida esperada

El artefacto `Alertas_Operacionales_Ventas` aparece en el Workspace y se abre en la interfaz de Data Activator. La pantalla muestra un entorno vacío con el mensaje de bienvenida o instrucciones para agregar datos.

#### Verificación

✅ El artefacto `Alertas_Operacionales_Ventas` aparece en el listado del Workspace  
✅ La interfaz de Data Activator se carga sin errores  
✅ Los tres paneles principales son visibles  

---

### Paso 3: Conexión de Data Activator al informe de Power BI

**Objetivo**: Conectar el artefacto Data Activator al informe `Reporte_Ventas_Silver` de Power BI para monitorear KPIs de ventas y crear el primer objeto de monitoreo basado en el visual de ventas totales.

#### Instrucciones

**Parte A: Configurar la alerta desde el informe de Power BI**

1. Abra una nueva pestaña del navegador y navegue al Workspace **FabricLakehouseLab**.

2. Haga clic en el informe **Reporte_Ventas_Silver** para abrirlo en modo de lectura (no en Power BI Desktop).

3. Identifique en el informe el visual de tipo **Tarjeta (Card)** que muestra el total de ventas o el KPI principal de ventas. Si tiene múltiples visuals, busque el que muestre `Venta Total` o `Total de Ventas`.

   > 📝 **Nota**: Si su informe tiene una estructura diferente a la esperada, seleccione cualquier visual de tipo Card o KPI que muestre un valor numérico relevante para el negocio.

4. Haga clic en los **tres puntos (...)** que aparecen en la esquina superior derecha del visual al pasar el cursor sobre él.

5. En el menú contextual, seleccione **Set alert** (Configurar alerta) o **Alertar** según el idioma de la interfaz.

   > ⚠️ **Importante**: Si no ve la opción "Set alert", asegúrese de estar viendo el informe en el servicio de Fabric (browser), no en Power BI Desktop. La opción solo está disponible en el servicio publicado.

6. Fabric abrirá automáticamente un panel lateral derecho de configuración de alerta. Revise los campos disponibles.

7. En el panel de configuración de alerta, configure los siguientes parámetros:

   | Campo | Valor a configurar |
   |---|---|
   | **Nombre de la alerta** | `Alerta_VentaTotal_Umbral_Minimo` |
   | **Condición** | `Becomes less than` (Se vuelve menor que) |
   | **Umbral (Threshold)** | Ingrese el 80% del valor de `venta_total_dia` que anotó en el Paso 1 (ej: si el valor era 50,000, ingrese 40,000) |
   | **Frecuencia de verificación** | `Daily` o la opción de menor frecuencia disponible para el trial |

8. En la sección de **Acciones (Actions)**, configure:
   - **Notificarme (Notify me)**: Asegúrese de que su email aparezca como destinatario
   - Si ve la opción de **Teams**, actívela también

9. Haga clic en **Start my alert** (Iniciar mi alerta) o **Crear** según la versión de la interfaz.

10. Fabric mostrará un mensaje de confirmación indicando que la alerta fue creada y que puede verla en Data Activator.

**Parte B: Verificar la creación del objeto en Data Activator**

11. Regrese a la pestaña donde tiene abierto el artefacto **Alertas_Operacionales_Ventas** en Data Activator.

    > 📝 **Nota**: Es posible que la alerta creada desde Power BI haya generado un Reflex separado con nombre automático. En ese caso, navegue al Workspace y abra el Reflex que se creó automáticamente. Puede renombrarlo o trabajar con él directamente.

12. En el panel izquierdo de Data Activator, debería ver un nuevo **Objeto** creado automáticamente basado en el visual de Power BI. El nombre generalmente refleja el nombre del visual o del informe.

13. Haga clic en el objeto para expandirlo y revise:
    - La **Propiedad** creada automáticamente (que corresponde al valor del visual)
    - La **Regla** (Rule/Trigger) con la condición que configuró

14. Haga clic en la **Regla** para ver sus detalles en el panel central. Confirme que:
    - La condición refleja `valor < [umbral configurado]`
    - La acción de notificación está configurada correctamente

#### Salida esperada

En Data Activator aparece un objeto de monitoreo conectado al visual de Power BI. La regla muestra la condición de umbral configurada y la acción de notificación por email. El estado de la regla aparece como **Active** (Activo).

#### Verificación

✅ La opción "Set alert" fue accesible desde el visual del informe de Power BI  
✅ El objeto aparece en el panel de Data Activator con al menos una propiedad y una regla  
✅ La regla muestra estado "Active"  
✅ La condición de umbral está correctamente configurada  

---

### Paso 4: Creación de un objeto de monitoreo operacional con condición compuesta

**Objetivo**: Crear manualmente un segundo objeto de monitoreo en Data Activator que monitoree el volumen de transacciones diarias directamente desde los datos del Lakehouse, con una condición de alerta compuesta y acción de notificación a Microsoft Teams.

#### Instrucciones

**Parte A: Preparar el dataset de métricas en Power BI para conectar a Data Activator**

Para monitorear métricas del Lakehouse desde Data Activator, la forma más práctica en el contexto de este laboratorio es crear un visual adicional en el informe de Power BI basado en los datos de la vista `vw_metricas_ventas_diarias`.

1. Abra **Power BI Desktop** en su computadora local.

2. Conéctese al modelo semántico `Modelo_Ventas_DirectLake` usando **Obtener datos → Power BI datasets** y seleccione el modelo del Workspace.

   > 📝 **Alternativa**: Si prefiere trabajar directamente en el servicio, puede editar el informe `Reporte_Ventas_Silver` en el navegador usando el modo de edición.

3. Agregue un nuevo visual de tipo **Tarjeta (Card)** al informe con la medida de **Total de Transacciones** (count de registros de ventas). Si la medida no existe, créela:

```dax
-- Medida DAX para total de transacciones del día más reciente
Total_Transacciones_Hoy = 
CALCULATE(
    COUNTROWS(silver_ventas),
    DATESINPERIOD(
        silver_ventas[fecha_venta],
        LASTDATE(silver_ventas[fecha_venta]),
        -1,
        DAY
    )
)
```

4. Agregue también una tarjeta con la medida de **Promedio de Ventas 7 días**:

```dax
-- Medida DAX para promedio de ventas de los últimos 7 días
Promedio_Ventas_7dias = 
CALCULATE(
    AVERAGE(silver_ventas[monto_total]),
    DATESINPERIOD(
        silver_ventas[fecha_venta],
        LASTDATE(silver_ventas[fecha_venta]),
        -7,
        DAY
    )
)
```

5. Guarde y publique el informe actualizado al Workspace **FabricLakehouseLab** con el mismo nombre `Reporte_Ventas_Silver` (reemplazando el existente).

**Parte B: Crear alerta sobre el volumen de transacciones**

6. En el servicio de Fabric (navegador), abra el informe `Reporte_Ventas_Silver` actualizado.

7. Localice el nuevo visual de tarjeta **Total_Transacciones_Hoy**.

8. Haga clic en los **tres puntos (...)** del visual y seleccione **Set alert**.

9. Configure la alerta con los siguientes parámetros:

   | Campo | Valor |
   |---|---|
   | **Nombre** | `Alerta_Volumen_Transacciones_Bajo` |
   | **Condición** | `Becomes less than` |
   | **Umbral** | `100` (simula un umbral de negocio: menos de 100 transacciones diarias es anómalo) |
   | **Frecuencia** | La más baja disponible |

10. En la sección de acciones, configure:
    - **Email**: Su dirección de correo
    - **Teams**: Active esta opción si está disponible y seleccione su canal o chat personal

11. Haga clic en **Start my alert**.

**Parte C: Personalizar la regla en Data Activator**

12. Navegue al artefacto **Alertas_Operacionales_Ventas** en el Workspace (o al Reflex que se creó automáticamente).

13. En el panel izquierdo, localice el nuevo objeto creado para el visual de transacciones.

14. Haga clic en el objeto y luego en la **Regla** asociada para editarla.

15. Busque la opción para **editar la condición** y agregue una condición adicional si la interfaz lo permite:
    - Condición principal: `Total_Transacciones_Hoy < 100`
    - Si hay opción de condición secundaria (AND): `Total_Transacciones_Hoy < Promedio_Ventas_7dias * 0.8`

    > 📝 **Nota**: La disponibilidad de condiciones compuestas depende de la versión de Data Activator en su tenant. Si no está disponible, la condición simple es suficiente para el objetivo del laboratorio.

16. En la sección de **Acciones** de la regla, verifique que esté configurada la acción de **Teams**. Si no está, haga clic en **Add action** y seleccione **Microsoft Teams**.

17. Configure el mensaje de Teams:

    | Campo | Valor |
    |---|---|
    | **Canal/Destinatario** | Su canal de Teams o chat personal |
    | **Mensaje personalizado** | `⚠️ ALERTA OPERACIONAL: El volumen de transacciones de ventas ha caído por debajo del umbral mínimo. Valor detectado: {value}. Revisar pipeline de ingesta inmediatamente.` |

18. Haga clic en **Save** (Guardar) para guardar los cambios en la regla.

#### Salida esperada

El artefacto Data Activator contiene ahora dos objetos de monitoreo activos: uno para ventas totales y otro para volumen de transacciones. Ambos tienen reglas activas con acciones configuradas. La regla de transacciones muestra la acción de Teams configurada con el mensaje personalizado.

#### Verificación

✅ Dos objetos de monitoreo visibles en el panel izquierdo de Data Activator  
✅ La regla de transacciones tiene acción de Teams configurada  
✅ El mensaje personalizado de Teams está guardado  
✅ Ambas reglas muestran estado **Active**  

---

### Paso 5: Configuración de alerta para monitoreo del pipeline de ingesta

**Objetivo**: Crear una tercera regla de alerta que monitoree el estado del pipeline `Pipeline_Ingesta_Bronze` para detectar fallos o ejecuciones ausentes, configurando una acción de re-ejecución automática del pipeline como respuesta.

#### Instrucciones

**Parte A: Crear métrica de estado del pipeline en el modelo**

1. En Power BI Desktop, agregue la siguiente medida DAX al modelo semántico para simular una métrica de estado del pipeline:

```dax
-- Medida para detectar si hay datos recientes (simula monitoreo de pipeline)
-- Retorna 1 si hay datos de las últimas 24 horas, 0 si no hay datos recientes
Pipeline_Datos_Recientes = 
VAR UltimaFecha = LASTDATE(silver_ventas[fecha_venta])
VAR HorasDesdeUltimoDato = 
    DATEDIFF(UltimaFecha, TODAY(), HOUR)
RETURN
    IF(HorasDesdeUltimoDato <= 24, 1, 0)
```

2. Agregue un visual de **Tarjeta** con esta medida al informe y publíquelo nuevamente.

3. En el servicio de Fabric, abra el informe y cree una alerta sobre este nuevo visual:

   | Campo | Valor |
   |---|---|
   | **Nombre** | `Alerta_Pipeline_Sin_Datos_Recientes` |
   | **Condición** | `Becomes equal to` o `Is less than` |
   | **Umbral** | `1` (alerta cuando el valor sea 0, indicando ausencia de datos recientes) |

4. Configure la acción de email con el siguiente asunto y cuerpo:
   - **Asunto**: `[CRÍTICO] Pipeline de ingesta sin ejecución en las últimas 24 horas`
   - **Cuerpo**: Incluya el valor detectado y la hora de detección

**Parte B: Configurar activación de pipeline como acción correctiva (opcional avanzado)**

5. En el artefacto Data Activator, navegue a la regla `Alerta_Pipeline_Sin_Datos_Recientes`.

6. Haga clic en **Add action** (Agregar acción).

7. Si está disponible en su versión de Fabric, seleccione **Start a Fabric item** o **Run a pipeline** como tipo de acción.

   > 📝 **Nota**: La capacidad de activar pipelines directamente desde Data Activator puede estar en preview o no disponible en todas las regiones. Si no está disponible, configure en su lugar una acción de **Power Automate** con el siguiente flujo:

```
Trigger: Data Activator webhook
Action 1: HTTP POST a la API de Fabric para ejecutar el pipeline
   URL: https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/items/{pipelineId}/jobs/instances
   Method: POST
   Body: {"executionData": {}}
Action 2: Enviar email de confirmación de re-ejecución iniciada
```

8. Si usa la activación directa de pipeline, configure:

   | Campo | Valor |
   |---|---|
   | **Workspace** | `FabricLakehouseLab` |
   | **Pipeline** | `Pipeline_Ingesta_Bronze` |
   | **Descripción de la acción** | `Re-ejecución automática por ausencia de datos` |

9. Guarde todos los cambios.

**Parte C: Revisar el estado consolidado de todas las reglas**

10. En el panel izquierdo de Data Activator, expanda todos los objetos y revise el árbol completo. Debería tener una estructura similar a:

```
📦 Alertas_Operacionales_Ventas (Reflex)
├── 📊 Objeto: Ventas_KPI_Total
│   └── 📏 Propiedad: venta_total
│       └── 🔔 Regla: Alerta_VentaTotal_Umbral_Minimo [ACTIVE]
│           └── ✉️ Acción: Email
├── 📊 Objeto: Ventas_Volumen_Transacciones  
│   └── 📏 Propiedad: total_transacciones_hoy
│       └── 🔔 Regla: Alerta_Volumen_Transacciones_Bajo [ACTIVE]
│           ├── 💬 Acción: Teams
│           └── ✉️ Acción: Email
└── 📊 Objeto: Pipeline_Estado_Ingesta
    └── 📏 Propiedad: pipeline_datos_recientes
        └── 🔔 Regla: Alerta_Pipeline_Sin_Datos_Recientes [ACTIVE]
            ├── ✉️ Acción: Email
            └── ⚙️ Acción: Pipeline (si disponible)
```

11. Tome una captura de pantalla del árbol de objetos completo para incluirla en el runbook operativo.

#### Salida esperada

Tres objetos de monitoreo activos en Data Activator, cada uno con al menos una regla y una acción configurada. La estructura jerárquica Objeto → Propiedad → Regla → Acción está correctamente implementada.

#### Verificación

✅ Tres objetos de monitoreo visibles en Data Activator  
✅ La regla de pipeline tiene acción de email configurada  
✅ La acción de pipeline correctivo está configurada (o documentada la alternativa con Power Automate)  
✅ Todas las reglas muestran estado **Active**  

---

### Paso 6: Simulación de escenarios de alerta y verificación end-to-end

**Objetivo**: Simular las condiciones que disparan las alertas configuradas para verificar que el flujo completo de detección → notificación funciona correctamente en el sistema de observabilidad.

#### Instrucciones

**Parte A: Simulación de alerta de ventas por debajo del umbral**

Para simular una condición de alerta sin modificar los datos reales, ajustaremos temporalmente el umbral de la regla a un valor que los datos actuales ya superan (o viceversa), forzando que la condición se cumpla.

1. En Data Activator, navegue a la regla **Alerta_VentaTotal_Umbral_Minimo**.

2. Haga clic en **Edit** (Editar) en la regla.

3. Cambie temporalmente el umbral al valor **más alto** posible (por encima del valor actual de ventas), de modo que la condición `Becomes less than [umbral_alto]` se evalúe como verdadera con los datos actuales.

   Por ejemplo, si las ventas actuales son 50,000, cambie el umbral a 999,999.

4. Haga clic en **Save** y espere entre 2 y 5 minutos para que el motor de Data Activator evalúe la condición.

5. Revise su **bandeja de entrada de email** para verificar la llegada de la notificación. El email debe incluir:
   - Nombre de la regla disparada
   - Valor detectado
   - Hora de detección
   - Enlace al informe o al Reflex

6. Tome una captura de pantalla del email recibido.

7. Una vez verificada la notificación, **restaure el umbral original** (el valor del 80% calculado en el Paso 1) y guarde.

**Parte B: Verificación de la alerta de Teams**

8. En Data Activator, navegue a la regla **Alerta_Volumen_Transacciones_Bajo**.

9. Ajuste temporalmente el umbral de transacciones a un valor superior al actual (ej: si hay 500 transacciones, suba el umbral a 10,000).

10. Guarde y espere 2-5 minutos.

11. Revise el **canal o chat de Microsoft Teams** configurado para verificar la llegada del mensaje de alerta.

12. Confirme que el mensaje contiene:
    - El texto personalizado configurado en el Paso 4
    - El valor detectado (`{value}` reemplazado por el valor real)
    - La hora de detección

13. Tome una captura de pantalla del mensaje de Teams recibido.

14. **Restaure el umbral original** (100 transacciones) y guarde.

**Parte C: Revisión del historial de alertas disparadas**

15. En Data Activator, busque la sección de **Historial** o **History** en la barra superior o en el menú del artefacto.

16. Revise el registro de eventos disparados durante la simulación. Debe mostrar:
    - Fecha y hora de cada disparo
    - Nombre de la regla
    - Valor que causó el disparo
    - Acción ejecutada y su estado (enviado/fallido)

17. Exporte o anote la información del historial para incluirla en el runbook operativo.

**Parte D: Verificación desde el Monitor Hub de Fabric**

18. En el panel de navegación izquierdo de Fabric, busque el icono de **Monitor** (reloj o ícono de monitoreo) para acceder al **Monitor Hub**.

19. En el Monitor Hub, revise las ejecuciones recientes del pipeline `Pipeline_Ingesta_Bronze` para confirmar que no fue activado accidentalmente durante las pruebas.

20. Verifique también el estado de los items de Data Activator en el Monitor Hub.

#### Salida esperada

- Email recibido con detalles de la alerta de ventas simulada
- Mensaje de Teams recibido con el texto personalizado y el valor detectado
- Historial de Data Activator muestra los dos disparos de prueba con estado "Sent" (Enviado)
- Monitor Hub no muestra ejecuciones no deseadas del pipeline

#### Verificación

✅ Email de alerta recibido con información correcta del evento  
✅ Mensaje de Teams recibido con formato y contenido esperado  
✅ Historial de Data Activator registra los eventos de prueba  
✅ Umbrales restaurados a sus valores operacionales originales  

---

### Paso 7: Documentación del Runbook Operativo

**Objetivo**: Crear un documento de runbook operativo que describa el sistema de alertas implementado, los procedimientos de respuesta para cada tipo de alerta y las instrucciones de mantenimiento del sistema de observabilidad.

#### Instrucciones

1. Abra un editor de texto (puede usar el Notebook de Fabric, Visual Studio Code o cualquier procesador de texto).

2. Cree el runbook operativo con la siguiente estructura. Complete cada sección con la información específica de su implementación:

```markdown
# RUNBOOK OPERATIVO: Sistema de Alertas Automáticas - FabricLakehouseLab
## Versión: 1.0 | Fecha: [fecha actual] | Autor: [su nombre]

---

## 1. RESUMEN EJECUTIVO

Este runbook describe el sistema de monitoreo y alertas automáticas 
implementado sobre la arquitectura Medallion de Microsoft Fabric para 
el dataset de ventas. El sistema detecta automáticamente anomalías en 
KPIs de negocio y métricas operacionales del pipeline, notificando al 
equipo de ingeniería de datos sin intervención manual.

---

## 2. INVENTARIO DE ALERTAS CONFIGURADAS

| ID | Nombre de Alerta | Objeto Monitoreado | Condición | Umbral | Acción | Frecuencia |
|----|---|---|---|---|---|---|
| A01 | Alerta_VentaTotal_Umbral_Minimo | Visual Power BI - Venta Total | < umbral | [valor] | Email | [freq] |
| A02 | Alerta_Volumen_Transacciones_Bajo | Visual Power BI - Transacciones | < 100 | 100 | Email + Teams | [freq] |
| A03 | Alerta_Pipeline_Sin_Datos_Recientes | Visual Power BI - Pipeline Status | = 0 | 1 | Email + Pipeline | [freq] |

---

## 3. PROCEDIMIENTOS DE RESPUESTA (PLAYBOOKS)

### ALERTA A01: Ventas por debajo del umbral mínimo

**Síntoma**: Email de alerta con asunto "Alerta_VentaTotal_Umbral_Minimo"
**Causa probable**: 
  - Datos no cargados en la última ejecución del pipeline
  - Problema en la fuente de datos (archivos no llegaron)
  - Día festivo o baja estacional normal

**Pasos de respuesta**:
1. Verificar el Monitor Hub para el estado del último run del pipeline
2. Revisar los logs del Pipeline_Ingesta_Bronze en busca de errores
3. Verificar disponibilidad de la fuente de datos (Azure Storage / archivos)
4. Si el pipeline falló: ejecutar manualmente desde Fabric
5. Si los datos son correctos pero bajos: escalar a negocio para análisis

**Tiempo máximo de respuesta**: 2 horas
**Escalación**: [nombre del responsable] si no se resuelve en 2 horas

---

### ALERTA A02: Volumen de transacciones bajo

**Síntoma**: Mensaje de Teams con texto "ALERTA OPERACIONAL: El volumen de transacciones..."
**Causa probable**:
  - Ingesta parcial de datos
  - Problema de calidad en la capa Bronze (registros rechazados)
  - Problema en la transformación Silver del Notebook

**Pasos de respuesta**:
1. Verificar el conteo de registros en bronze.ventas_raw vs silver_ventas
2. Revisar el Notebook de transformación para errores en la última ejecución
3. Verificar la vista vw_metricas_ventas_diarias para confirmar el valor
4. Si hay discrepancia Bronze/Silver: re-ejecutar el Notebook de transformación
5. Documentar el incidente en el log de operaciones

**Tiempo máximo de respuesta**: 4 horas
**Escalación**: [nombre del responsable]

---

### ALERTA A03: Pipeline sin datos recientes

**Síntoma**: Email con asunto "[CRÍTICO] Pipeline de ingesta sin ejecución..."
**Causa probable**:
  - Pipeline no programado correctamente
  - Fallo silencioso en la ejecución del pipeline
  - Problema de conectividad con la fuente de datos

**Pasos de respuesta**:
1. Verificar el Monitor Hub para confirmar la última ejecución
2. Si el pipeline no ejecutó en 24h: ejecutar manualmente
3. Verificar la programación (schedule) del pipeline
4. Si hay error de conectividad: verificar credenciales y SAS tokens
5. Si se activó el pipeline correctivo automáticamente: verificar su resultado

**Tiempo máximo de respuesta**: 1 hora (crítico)
**Escalación**: INMEDIATA si el pipeline no puede ejecutarse

---

## 4. MANTENIMIENTO DEL SISTEMA DE ALERTAS

### Revisión mensual recomendada:
- [ ] Verificar que todos los umbrales siguen siendo relevantes para el negocio
- [ ] Revisar el historial de alertas del mes anterior
- [ ] Ajustar umbrales si los patrones de datos han cambiado (estacionalidad)
- [ ] Verificar que los destinatarios de email/Teams siguen siendo correctos
- [ ] Probar que las acciones de notificación siguen funcionando

### Ajuste de umbrales:
Los umbrales deben revisarse cada trimestre. Para ajustar:
1. Navegar a Fabric > Workspace FabricLakehouseLab
2. Abrir Alertas_Operacionales_Ventas (Reflex)
3. Seleccionar la regla a modificar
4. Editar el valor del umbral
5. Guardar y verificar con una prueba controlada

---

## 5. ARQUITECTURA DEL SISTEMA DE OBSERVABILIDAD

[Incluir captura de pantalla del árbol de objetos de Data Activator]

Fuentes de datos → Data Activator → Condiciones → Acciones
Power BI Report → Objeto A01 → Venta < umbral → Email
Power BI Report → Objeto A02 → Trans < 100 → Email + Teams  
Power BI Report → Objeto A03 → Pipeline = 0 → Email + Pipeline

---

## 6. CONTACTOS Y ESCALACIÓN

| Rol | Nombre | Email | Teams |
|---|---|---|---|
| Ingeniero de Datos | [nombre] | [email] | [handle] |
| Responsable de Negocio | [nombre] | [email] | [handle] |
| Administrador de Fabric | [nombre] | [email] | [handle] |
```

3. Guarde el runbook con el nombre `Runbook_Alertas_Operacionales_v1.0.md` o `.docx`.

4. Si desea almacenarlo en Fabric, puede crear un **Notebook** en el Workspace con el contenido del runbook en celdas de tipo Markdown.

#### Salida esperada

Un documento de runbook completo con las secciones: inventario de alertas, procedimientos de respuesta para cada alerta, plan de mantenimiento y arquitectura del sistema. El documento está guardado localmente o en el Workspace de Fabric.

#### Verificación

✅ El runbook contiene las tres alertas configuradas con sus umbrales reales  
✅ Cada alerta tiene un procedimiento de respuesta documentado  
✅ El plan de mantenimiento incluye revisiones periódicas  
✅ El documento está guardado y accesible  

---

## Validación y Pruebas del Laboratorio

Ejecute la siguiente lista de verificación completa para confirmar que el laboratorio fue completado exitosamente:

### Lista de verificación de validación final

```
VALIDACIÓN DEL LABORATORIO 05-00-01
=====================================

SECCIÓN 1: Artefactos de Fabric
[ ] 1.1 Artefacto "Alertas_Operacionales_Ventas" (Reflex) existe en el Workspace
[ ] 1.2 El Reflex contiene mínimo 2 objetos de monitoreo activos
[ ] 1.3 Cada objeto tiene al menos 1 propiedad y 1 regla configurada
[ ] 1.4 Al menos 1 regla tiene acción de Email configurada
[ ] 1.5 Al menos 1 regla tiene acción de Teams configurada
[ ] 1.6 Todas las reglas muestran estado "Active"

SECCIÓN 2: Conexión con Power BI
[ ] 2.1 Las alertas están conectadas al informe "Reporte_Ventas_Silver"
[ ] 2.2 Los umbrales reflejan valores de negocio relevantes (no valores de prueba)
[ ] 2.3 El informe de Power BI sigue funcionando correctamente después de los cambios

SECCIÓN 3: Verificación de notificaciones
[ ] 3.1 Email de alerta recibido durante la simulación del Paso 6
[ ] 3.2 Mensaje de Teams recibido durante la simulación del Paso 6
[ ] 3.3 El historial de Data Activator registra los eventos de prueba
[ ] 3.4 Los umbrales fueron restaurados a valores operacionales después de las pruebas

SECCIÓN 4: Documentación
[ ] 4.1 Runbook operativo creado con las tres alertas documentadas
[ ] 4.2 Cada alerta tiene procedimiento de respuesta definido
[ ] 4.3 Captura de pantalla del árbol de objetos de Data Activator incluida

SECCIÓN 5: Verificación de integridad del entorno
[ ] 5.1 El Lakehouse "SalesLakehouse" sigue accesible con sus tablas Silver
[ ] 5.2 El pipeline "Pipeline_Ingesta_Bronze" no fue ejecutado accidentalmente
[ ] 5.3 El modelo semántico "Modelo_Ventas_DirectLake" sigue funcional
[ ] 5.4 El Monitor Hub no muestra errores no esperados

RESULTADO: _____ de 18 ítems completados
APROBADO: 16/18 ítems mínimo requeridos
```

### Consulta de validación final en el Lakehouse

Ejecute esta consulta en el SQL Analytics Endpoint para confirmar que los datos no fueron alterados durante el laboratorio:

```sql
-- Validación final: integridad de datos post-laboratorio
SELECT 
    'Registros Silver Ventas'   AS metrica, 
    COUNT(*)                    AS valor
FROM silver_ventas

UNION ALL

SELECT 
    'Registros Silver Productos' AS metrica, 
    COUNT(*)                     AS valor
FROM silver_productos

UNION ALL

SELECT 
    'Días con datos (vista métricas)' AS metrica,
    COUNT(DISTINCT fecha)             AS valor
FROM vw_metricas_ventas_diarias;
```

Los valores deben ser consistentes con los verificados al inicio del laboratorio en el Paso 1.

---

## Solución de Problemas

### Problema 1: La opción "Set alert" no aparece en el visual de Power BI

**Síntomas**:
- Al hacer clic en los tres puntos (...) del visual en el informe publicado, la opción "Set alert" o "Configurar alerta" no aparece en el menú contextual
- El menú solo muestra opciones como "Exportar datos", "Ver datos" o "Spotlight"

**Causa**:
Este problema ocurre por una de dos razones: (a) el informe se está visualizando en Power BI Desktop en lugar del servicio de Fabric en el navegador, o (b) el tipo de visual seleccionado no es compatible con alertas de Data Activator. Data Activator solo puede conectarse a visuals de tipo **Card (Tarjeta)**, **KPI** o **Gauge** que muestren un único valor numérico. Los visuals de tabla, matriz o gráficos de barras no soportan alertas directas.

**Solución**:
1. Confirme que está accediendo al informe a través del navegador en `app.fabric.microsoft.com`, no desde Power BI Desktop.
2. Verifique que el visual seleccionado es de tipo **Card (Tarjeta)** o **KPI**. Si no tiene este tipo de visual en su informe, abra el informe en modo edición, agregue una tarjeta con la medida deseada, publique y luego intente nuevamente.
3. Si el problema persiste, cree el Reflex directamente desde el Workspace (**+ New → Reflex**) y use la opción **Get data → Power BI** dentro de Data Activator para conectarlo manualmente al dataset, sin usar el atajo desde el visual.

---

### Problema 2: Las notificaciones de alerta no llegan al email o Teams después de la simulación

**Síntomas**:
- La regla en Data Activator muestra estado "Active" y la condición debería estar cumpliéndose (umbral ajustado por encima del valor actual)
- Han pasado más de 10 minutos desde el ajuste del umbral y no hay email ni mensaje de Teams
- El historial de Data Activator no muestra ningún disparo reciente

**Causa**:
Existen tres causas comunes: (a) el intervalo de evaluación de Data Activator cuando la fuente es Power BI no es en tiempo real, sino por intervalos que pueden ser de 15 a 30 minutos dependiendo de la carga del tenant y la configuración; (b) el email o la cuenta de Teams usada para la acción no tiene permisos correctamente configurados en el tenant; (c) el dataset de Power BI no se ha actualizado desde que se ajustó el umbral, por lo que el valor que Data Activator está evaluando es el valor anterior al cambio.

**Solución**:
1. **Para el problema de latencia**: Espere al menos 15-20 minutos después de ajustar el umbral. El trial de Fabric tiene recursos compartidos que pueden aumentar la latencia de evaluación. Si después de 20 minutos no hay notificación, continúe con el laboratorio y documente la configuración como válida; la latencia no indica un error de configuración.
2. **Para el problema de permisos de Teams**: Verifique que la cuenta usada en la acción de Teams tiene permisos para enviar mensajes al canal seleccionado. Si usa un canal de equipo (no chat personal), el bot de Fabric debe estar agregado al canal. Alternativamente, use el chat personal (1:1 con usted mismo) para las pruebas.
3. **Para el problema de actualización del dataset**: En el informe de Power BI, haga clic en el botón de **Actualizar** (refresh) para forzar una actualización del dataset. Luego espere que Data Activator evalúe el nuevo valor. También puede verificar en la configuración de la regla si existe un botón de **Test** o **Preview** que permita ver el valor actual que Data Activator está leyendo.

---

## Limpieza de Recursos

> ⚠️ **INSTRUCCIÓN CRÍTICA**: Este es el **último laboratorio** de la serie. Después de completar este lab, puede proceder con la limpieza completa del entorno. **NO elimine recursos si planea revisar o demostrar el trabajo realizado** en los próximos días.

### Limpieza al finalizar toda la serie de laboratorios

Cuando esté listo para liberar la capacidad Trial de Fabric, siga estos pasos en orden:

1. **Documentar antes de eliminar**: Exporte el runbook operativo y tome capturas de pantalla de todos los artefactos para su portafolio personal.

2. **Desactivar las reglas de Data Activator** (para evitar notificaciones durante la limpieza):
   - Abra `Alertas_Operacionales_Ventas` en Data Activator
   - Para cada regla, cambie el estado a **Paused** o **Inactive**
   - Esto evita recibir alertas espurias durante la eliminación de recursos

3. **Eliminar el Workspace completo** (método recomendado — elimina todos los artefactos de una vez):
   - Navegue a **Workspaces** en el panel izquierdo de Fabric
   - Haga clic en los tres puntos (...) junto a **FabricLakehouseLab**
   - Seleccione **Workspace settings** (Configuración del Workspace)
   - En la sección inferior, haga clic en **Delete this workspace** (Eliminar este Workspace)
   - Confirme la eliminación escribiendo el nombre del Workspace cuando se solicite
   - Haga clic en **Delete**

   > ⚠️ Esta acción es **irreversible** y eliminará permanentemente: el Lakehouse y todos sus datos, los pipelines, notebooks, modelos semánticos, informes de Power BI y el artefacto Data Activator.

4. **Verificar la liberación de capacidad**:
   - Navegue a **[Portal de administración de Fabric](https://app.fabric.microsoft.com/admin-portal)**
   - Verifique que la capacidad Trial muestra los recursos liberados

5. **Limpieza opcional de recursos externos** (si aplica):
   - Si creó un Azure Storage Account durante los labs, elimínelo desde el Portal de Azure
   - Revoque cualquier SAS Token que haya compartido durante el curso

---

## Resumen del Laboratorio

### Logros alcanzados

En este laboratorio completó la implementación de un sistema completo de observabilidad operacional sobre la arquitectura Medallion de Microsoft Fabric. Los logros concretos incluyen:

| Logro | Artefacto creado |
|---|---|
| Sistema de monitoreo de KPIs de negocio | Reflex `Alertas_Operacionales_Ventas` |
| Alerta de ventas por debajo del umbral | Regla `Alerta_VentaTotal_Umbral_Minimo` |
| Alerta de volumen de transacciones bajo | Regla `Alerta_Volumen_Transacciones_Bajo` |
| Alerta de pipeline sin datos recientes | Regla `Alerta_Pipeline_Sin_Datos_Recientes` |
| Notificaciones automáticas por email | Acciones de email en 3 reglas |
| Notificaciones automáticas por Teams | Acción de Teams en regla A02 |
| Documentación operacional | Runbook `Runbook_Alertas_Operacionales_v1.0` |

### Arquitectura Medallion completa: ciclo cerrado

Con este laboratorio, la arquitectura completa implementada a lo largo de los cinco labs queda así:

```
┌─────────────────────────────────────────────────────────────────────┐
│                 ARQUITECTURA MEDALLION COMPLETA                      │
│                                                                      │
│  [Lab 01]          [Lab 02]         [Lab 03]                         │
│  Lakehouse    →    Pipeline    →    Notebook                         │
│  OneLake           Bronze           PySpark/SQL                      │
│  Shortcuts         Ingesta          Bronze → Silver                  │
│                                                                      │
│                    [Lab 04]         [Lab 05]                         │
│                    Modelo      →    Data Activator                   │
│                    Direct Lake      Alertas Automáticas              │
│                    Power BI         Email + Teams + Pipeline         │
│                                                                      │
│  ◄──────────── CICLO OPERATIVO COMPLETO ────────────────────────►   │
│   Ingestar → Transformar → Visualizar → Monitorear → Reaccionar     │
└─────────────────────────────────────────────────────────────────────┘
```

### Conceptos clave reforzados

- **Data Activator** opera sobre tres pilares: **Objeto** (qué se monitorea), **Propiedad** (qué se mide) y **Regla** (cuándo y cómo se actúa)
- La conexión desde **visuals de Power BI** es el camino más rápido para implementar alertas sobre KPIs de negocio existentes
- Las **acciones de respuesta** escalan desde notificaciones simples (email, Teams) hasta automatizaciones complejas (pipelines, Power Automate)
- Un **runbook operativo** es el complemento indispensable de cualquier sistema de alertas: define qué hacer cuando la alerta se dispara
- La **simulación controlada** de alertas es la única forma de verificar el funcionamiento end-to-end antes de pasar a producción

### Recursos de referencia adicionales

| Recurso | URL |
|---|---|
| Documentación oficial de Data Activator | [learn.microsoft.com/fabric/data-activator](https://learn.microsoft.com/es-es/fabric/data-activator/data-activator-introduction) |
| Alertas desde Power BI con Data Activator | [learn.microsoft.com — get-data-power-bi](https://learn.microsoft.com/es-es/fabric/data-activator/data-activator-get-data-power-bi) |
| Integración con Power Automate | [learn.microsoft.com — power-automate-flows](https://learn.microsoft.com/es-es/fabric/data-activator/data-activator-trigger-power-automate-flows) |
| Eventstream en Microsoft Fabric | [learn.microsoft.com — eventstreams overview](https://learn.microsoft.com/es-es/fabric/real-time-intelligence/event-streams/overview) |
| Monitoreo en tiempo real con Fabric | [learn.microsoft.com — real-time-intelligence](https://learn.microsoft.com/es-es/fabric/real-time-intelligence/overview) |

---

> 🎓 **¡Felicitaciones!** Ha completado exitosamente los cinco laboratorios de la serie. Ha implementado una arquitectura Medallion completa en Microsoft Fabric, desde la ingesta de datos en la capa Bronze hasta el monitoreo automatizado con alertas basadas en eventos. Este es el ciclo completo de ingeniería de datos moderna: **ingestar, transformar, modelar, visualizar y observar**.
