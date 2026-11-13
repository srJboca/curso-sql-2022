# Laboratorio Sesión 5: Auditoría Real de Estructuras, Estadísticas e Integridad en Servidores Propios

**Duración:** 50 minutos

**Regla de Seguridad:** Todos los scripts de este laboratorio son **exclusivamente de consulta (`SELECT` y `DBCC` de lectura)**. No alteran esquemas, no borran datos y no reconstruyen índices automáticamente.

**Instrucción inicial para el alumno:** Selecciona en SSMS o Azure Data Studio la base de datos de negocio más importante de tu servidor (`USE [TuBaseDeDatos];`).

---

## Parte 1: Auditoría de Estructuras Físicas (Heaps vs. Clustered y Diseño de Índices)

**Objetivo:** Identificar en una base de datos real qué tablas carecen de índice agrupado (*Heaps*), cuáles sufren de *Forwarded Records*, qué índices tienen claves peligrosas (`GUID`/anchas) y dónde hay oportunidades de mejora.

### Paso 1.1: Detectar tablas sin índice agrupado (*Heaps*) y *Forwarded Records*

Ejecuta esta consulta para descubrir qué tablas de tu base de datos son *Heaps* y si están penalizando el I/O por registros reenviados:

```sql
-- 1.1 Auditoría de Heaps y Forwarded Records en la base de datos actual
SELECT 
    SCHEMA_NAME(t.schema_id) AS Esquema,
    t.name AS TablaHeap,
    p.rows AS FilasAprox,
    ips.page_count AS PaginasEnDisco,
    ips.forwarded_record_count AS RegistrosReenviados,
    ROUND(ips.avg_page_space_used_in_percent, 2) AS DensidadPagina_Pct,
    CASE 
        WHEN ips.forwarded_record_count > 0 THEN 'CRÍTICO: Evaluar Clustered Index o REBUILD del Heap'
        WHEN p.rows > 10000 THEN 'REVISAR: Tabla grande sin Clustered Index'
        ELSE 'OK (Tabla pequeña o staging)'
    END AS Diagnostico
FROM sys.tables t
JOIN sys.indexes i ON t.object_id = i.object_id AND i.index_id = 0 -- index_id 0 = Heap
JOIN sys.partitions p ON i.object_id = p.object_id AND i.index_id = p.index_id
CROSS APPLY sys.dm_db_index_physical_stats(DB_ID(), t.object_id, 0, NULL, 'DETAILED') ips
WHERE t.is_ms_shipped = 0
ORDER BY ips.forwarded_record_count DESC, p.rows DESC;
GO

```

> **Tarea para el alumno:** Anota las 3 tablas con mayor número de `RegistrosReenviados` o filas. ¿Son tablas temporales de carga (*staging*) o tablas transaccionales del negocio que deberían tener una llave primaria agrupada?

### Paso 1.2: Auditar el diseño de los *Clustered Indexes* existentes

Revisamos si los índices agrupados actuales cumplen las 4 reglas de oro (únicos, estrechos, estáticos y crecientes) o si usan tipos de datos problemáticos (`UNIQUEIDENTIFIER`, `VARCHAR`, claves compuestas muy anchas).

```sql
-- 1.2 Revisión de la composición y ancho de las claves Clustered
SELECT 
    SCHEMA_NAME(t.schema_id) AS Esquema,
    t.name AS Tabla,
    i.name AS IndiceClustered,
    i.is_unique AS EsUnico,
    STRING_AGG(c.name + ' (' + ty.name + ')', ', ') WITHIN GROUP (ORDER BY ic.key_ordinal) AS ColumnasClave,
    SUM(c.max_length) AS AnchoTotalBytes,
    CASE 
        WHEN STRING_AGG(ty.name, ',') LIKE '%uniqueidentifier%' THEN 'ALERTA: Posible Page Split por GUID'
        WHEN SUM(c.max_length) > 16 THEN 'REVISAR: Clave agrupada ancha (>16 bytes)'
        WHEN i.is_unique = 0 THEN 'NOTA: No es único (SQL añade Uniquifier de 4 bytes)'
        ELSE 'ÓPTIMO'
    END AS EvaluacionDiseño
FROM sys.tables t
JOIN sys.indexes i ON t.object_id = i.object_id AND i.index_id = 1 -- index_id 1 = Clustered
JOIN sys.index_columns ic ON i.object_id = ic.object_id AND i.index_id = ic.index_id AND ic.is_included_column = 0
JOIN sys.columns c ON ic.object_id = c.object_id AND ic.column_id = c.column_id
JOIN sys.types ty ON c.user_type_id = ty.user_type_id
WHERE t.is_ms_shipped = 0
GROUP BY t.schema_id, t.name, i.name, i.is_unique
ORDER BY AnchoTotalBytes DESC;
GO

```

---

## Parte 2: Auditoría de Estadísticas y Fragmentación Real

**Objetivo:** Detectar estadísticas desactualizadas que están engañando al optimizador y evaluar la fragmentación física real **sin caer en la trampa de reconstruir índices pequeños**.

### Paso 2.1: Detectar estadísticas obsoletas en tablas con alto movimiento

```sql
-- 2.1 Top 20 estadísticas con mayor porcentaje de filas modificadas sin actualizar
SELECT TOP 20
    SCHEMA_NAME(t.schema_id) AS Esquema,
    t.name AS Tabla,
    s.name AS NombreEstadistica,
    s.auto_created AS AutoCreada,
    s.no_recompute AS TieneNoRecompute,
    sp.last_updated AS UltimaActualizacion,
    sp.rows AS FilasTotales,
    sp.rows_sampled AS FilasMuestreadas,
    CAST(100.0 * sp.rows_sampled / NULLIF(sp.rows, 0) AS DECIMAL(5,2)) AS PorcentajeMuestreo,
    sp.modification_counter AS FilasModificadasDesdeUpdate,
    CAST(100.0 * sp.modification_counter / NULLIF(sp.rows, 0) AS DECIMAL(10,2)) AS PorcentajeCambio
FROM sys.tables t
JOIN sys.stats s ON t.object_id = s.object_id
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE t.is_ms_shipped = 0
  AND sp.rows > 1000
ORDER BY PorcentajeCambio DESC, sp.modification_counter DESC;
GO

```

> **Preguntas de análisis para el alumno:**
> 1. ¿Hay alguna estadística con `TieneNoRecompute = 1`? (Si es así, SQL Server nunca la actualizará automáticamente).
> 2. Toma una de las estadísticas críticas de una tabla principal y ejecuta:
> `DBCC SHOW_STATISTICS ('Esquema.Tabla', 'NombreEstadistica') WITH STAT_HEADER;`
> ¿Hace cuántos días o meses fue su última actualización?
> 
> 

### Paso 2.2: Evaluar fragmentación real aplicando filtros operativos (No tocar índices pequeños)

Aplicamos el criterio profesional (similar al de Ola Hallengren): ignorar índices de menos de **1,000 páginas** (~8 MB), porque reconstruirlos no aporta beneficio y solo genera I/O.

```sql
-- 2.2 Diagnóstico realista de fragmentación (> 1,000 páginas)
SELECT 
    SCHEMA_NAME(t.schema_id) AS Esquema,
    t.name AS Tabla,
    i.name AS Indice,
    i.type_desc AS TipoIndice,
    ips.page_count AS Paginas,
    CAST(ips.page_count * 8.0 / 1024 AS DECIMAL(10,2)) AS Tamano_MB,
    ROUND(ips.avg_fragmentation_in_percent, 2) AS Fragmentacion_Pct,
    ROUND(ips.avg_page_space_used_in_percent, 2) AS DensidadPagina_Pct,
    CASE 
        WHEN ips.avg_fragmentation_in_percent >= 30.0 THEN 'REBUILD (Fragmentación >= 30%)'
        WHEN ips.avg_fragmentation_in_percent >= 5.0  THEN 'REORGANIZE (Fragmentación 5% - 30%)'
        ELSE 'IGNORAR (< 5% - Saludable)'
    END AS AccionRecomendada
FROM sys.tables t
JOIN sys.indexes i ON t.object_id = i.object_id
CROSS APPLY sys.dm_db_index_physical_stats(DB_ID(), t.object_id, i.index_id, NULL, 'LIMITED') ips
WHERE t.is_ms_shipped = 0
  AND i.index_id > 0
  AND ips.page_count >= 1000 -- Regla de oro: ignorar índices pequeños (< 8 MB)
ORDER BY ips.avg_fragmentation_in_percent DESC;
GO

```

---

## Parte 3: Detección de Índices Inútiles, Faltantes e Historial de Integridad (`CHECKDB`)

**Objetivo:** Identificar qué índices están consumiendo espacio y penalizando escrituras sin aportar lecturas, evaluar las sugerencias de índices faltantes y auditar cuándo fue el último chequeo de integridad física real.

### Paso 3.1: Encontrar índices sin uso (*Unused Indexes*) que penalizan escrituras

*(Nota: Validar primero cuándo se reinició el servicio de SQL Server, ya que las DMVs se limpian en cada reinicio).*

```sql
-- 3.1 Verificar tiempo de actividad del servidor (Uptime)
SELECT sqlserver_start_time AS UltimoReinicioMotor,
       DATEDIFF(DAY, sqlserver_start_time, GETDATE()) AS DiasEnLinea
FROM sys.dm_os_sys_info;
GO

-- 3.2 Listar índices Nonclustered con CERO lecturas y alto costo de escritura
SELECT 
    SCHEMA_NAME(t.schema_id) AS Esquema,
    t.name AS Tabla,
    i.name AS IndiceSinUso,
    ISNULL(s.user_seeks, 0) + ISNULL(s.user_scans, 0) + ISNULL(s.user_lookups, 0) AS TotalLecturas,
    ISNULL(s.user_updates, 0) AS TotalEscrituras_Penalizacion,
    (SELECT SUM(ps.used_page_count) * 8 / 1024 
     FROM sys.dm_db_partition_stats ps 
     WHERE ps.object_id = i.object_id AND ps.index_id = i.index_id) AS EspacioDesperdiciado_MB
FROM sys.indexes i
JOIN sys.tables t ON i.object_id = t.object_id
LEFT JOIN sys.dm_db_index_usage_stats s 
    ON i.object_id = s.object_id AND i.index_id = s.index_id AND s.database_id = DB_ID()
WHERE t.is_ms_shipped = 0
  AND i.type_desc = 'NONCLUSTERED'
  AND i.is_primary_key = 0
  AND i.is_unique_constraint = 0
  AND (ISNULL(s.user_seeks, 0) + ISNULL(s.user_scans, 0) + ISNULL(s.user_lookups, 0)) = 0
ORDER BY TotalEscrituras_Penalizacion DESC, EspacioDesperdiciado_MB DESC;
GO

```

### Paso 3.3: Evaluar el Top 10 de Índices Faltantes (*Missing Indexes*)

```sql
-- 3.3 Top 10 sugerencias de índices faltantes ordenadas por impacto real
SELECT TOP 10
    OBJECT_NAME(mid.object_id, mid.database_id) AS Tabla,
    CAST(migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans) AS BIGINT) AS ScoreImpacto,
    migs.user_seeks + migs.user_scans AS VecesSolicitado,
    ROUND(migs.avg_user_impact, 2) AS MejoraEstimada_Pct,
    mid.equality_columns AS ColumnasIgualdad,
    mid.inequality_columns AS ColumnasDesigualdad,
    mid.included_columns AS ColumnasInclude
FROM sys.dm_db_missing_index_groups mig
JOIN sys.dm_db_missing_index_group_stats migs ON migs.group_handle = mig.index_group_handle
JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
WHERE mid.database_id = DB_ID()
ORDER BY ScoreImpacto DESC;
GO

```

> **Advertencia del Instructor:** Revisa si dos filas de este resultado pertenecen a la **misma tabla** y comparten columnas de igualdad. Un buen DBA **consolida** ambas sugerencias en un único índice en lugar de crear dos índices duplicados.

### Paso 3.4: Auditar cuándo se ejecutó el último `DBCC CHECKDB` exitoso en cada base

En lugar de lanzar un `CHECKDB` pesado en pleno horario laboral, consultamos la metadata interna (`DBCC DBINFO`) de todas las bases de datos de la instancia para verificar si tienen respaldo de integridad reciente.

```sql
-- 3.4 Auditoría de la fecha del último DBCC CHECKDB limpio en todas las bases de datos
CREATE TABLE #DBInfoTemp (
    ParentObject VARCHAR(255),
    [Object] VARCHAR(255),
    Field VARCHAR(255),
    [Value] VARCHAR(255)
);

CREATE TABLE #ResultadosCheckDB (
    BaseDeDatos SYSNAME,
    UltimoCheckDBLimpo DATETIME NULL
);

DECLARE @DBName SYSNAME;
DECLARE db_cursor CURSOR LOCAL FAST_FORWARD FOR
    SELECT name FROM sys.databases WHERE state_desc = 'ONLINE' AND name <> 'tempdb';

OPEN db_cursor;
FETCH NEXT FROM db_cursor INTO @DBName;

WHILE @@FETCH_STATUS = 0
BEGIN
    TRUNCATE TABLE #DBInfoTemp;
    INSERT INTO #DBInfoTemp EXEC ('DBCC DBINFO ([' + @DBName + ']) WITH TABLERESULTS');

    INSERT INTO #ResultadosCheckDB (BaseDeDatos, UltimoCheckDBLimpo)
    SELECT @DBName, TRY_CAST([Value] AS DATETIME)
    FROM #DBInfoTemp
    WHERE Field = 'dbi_dbccLastKnownGood';

    FETCH NEXT FROM db_cursor INTO @DBName;
END

CLOSE db_cursor;
DEALLOCATE db_cursor;

SELECT 
    BaseDeDatos,
    UltimoCheckDBLimpo,
    DATEDIFF(DAY, UltimoCheckDBLimpo, GETDATE()) AS DiasSinCheckDB,
    CASE 
        WHEN UltimoCheckDBLimpo IS NULL OR UltimoCheckDBLimpo = '1900-01-01' THEN 'CRÍTICO: Nunca se ha ejecutado CHECKDB'
        WHEN DATEDIFF(DAY, UltimoCheckDBLimpo, GETDATE()) > 7 THEN 'ALERTA: Más de 7 días sin validación física'
        ELSE 'OK (Protegida esta semana)'
    END AS EstadoIntegridad
FROM #ResultadosCheckDB
ORDER BY DiasSinCheckDB DESC;

DROP TABLE #DBInfoTemp;
DROP TABLE #ResultadosCheckDB;
GO

```

---

## Entregable del Alumno (Reporte de Hallazgos de su Servidor)

Al finalizar los 50 minutos, cada estudiante debe presentar 4 hallazgos reales extraídos de su base de datos:

1. **Un *Heap* con *Forwarded Records*** (o confirmar que todas sus tablas principales tienen *Clustered Index*).
2. **Una estadística crítica** con alto porcentaje de modificación pendiente (`modification_counter`).
3. **Un índice candidato a eliminación** (0 lecturas vs. miles de escrituras) y **una propuesta consolidada** de *Missing Index*.
4. **El estado de `DBCC CHECKDB**` de su instancia (cuántos días llevan las bases de datos sin validación física).
