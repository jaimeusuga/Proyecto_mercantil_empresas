# Data Lakehouse: Registro Mercantil y SECOP

Proyecto de Data Lakehouse para integrar datos del **Registro Mercantil de Colombia (RUES)** con contratos públicos de **SECOP**, usando arquitectura Medallón (Bronze → Silver → Gold) sobre Databricks con Delta Lake y Unity Catalog.

---

## Fuentes de datos

Ambas fuentes provienen de **Datos Abiertos Colombia** (datos.gov.co) vía API REST (Socrata Open Data):

| Fuente | Tabla Bronze | Volumen | API |
|--------|-------------|---------|-----|
| Registro Mercantil (RUES) | `workspace.bronze.bronze_registro_mercantil` | ~660K registros, 634K NITs únicos | `datos.gov.co/resource/c82u-588k.json` |
| SECOP Contratos | `workspace.bronze.bronze_secop_contratos` | ~660K contratos, 410K proveedores únicos | `datos.gov.co/resource/jbjy-vk9h.json` |

**Llave de unión:** NIT — `numero_identificacion` (Registro Mercantil) / `documento_proveedor` (SECOP)

---

## Notebooks (orden de ejecución)

| # | Notebook | Descripción |
|---|----------|-------------|
| 1 | `00_setup_unity_catalog` | Creación del catálogo `workspace` y esquema `workspace.bronze` en Unity Catalog |
| 2 | `01_proyecto_bronze_completo` | Ingesta de datos desde las APIs de datos.gov.co a las tablas Bronze con PySpark. Incluye paginación HTTP, campos de auditoría (`_ingested_at`, `_source`) y escritura incremental en Delta Lake |
| 3 | `validaciones` | Validaciones de calidad de datos sobre las tablas Bronze: conteo de registros, nulos por columna, duplicados exactos, fechas futuras, valores negativos, outliers y consistencia temporal |

---

## Arquitectura

```
Fuentes externas (datos.gov.co)
    ↓  HTTP GET paginado (PySpark + requests)
Capa Bronze (Delta Lake — inmutable, con auditoría)
    ↓  Limpieza, casteo de tipos, deduplicación
Capa Silver (pendiente)
    ↓  Agregaciones, joins, métricas de negocio
Capa Gold (pendiente)
```

- **Almacenamiento:** Delta Lake (ACID, Time Travel, Schema Evolution)
- **Catálogo:** Unity Catalog — `workspace.bronze`
- **Modo de carga:** Append (incremental, inmutable)
- **Campos de auditoría:** `_ingested_at` (timestamp UTC), `_source` (URL del endpoint)

---

## Prerrequisitos

- Acceso a un workspace de Databricks con Unity Catalog habilitado
- Permisos `CREATE SCHEMA` y `CREATE TABLE` en el catálogo `workspace`
- Compute serverless o cluster con Spark (soporta Python, SQL y sh)
- Conectividad a internet para acceder a las APIs de datos.gov.co

---

## Hallazgos de calidad (capa Bronze)

| Hallazgo | Tabla | Severidad |
|----------|-------|----------|
| 11,461 registros con fecha_matricula > fecha_cancelacion | RM | Media |
| 4,882 filas duplicadas exactas | SECOP | Alta |
| Valor máximo absurdo en valor_del_contrato (~$734 quintillones) | SECOP | Alta |
| 91 contratos con fecha_inicio > fecha_fin | SECOP | Media |
| 28 contratos con fecha de inicio futura | SECOP | Baja |
| Columnas ~90% vacías (liquidación, prórroga) | SECOP | Informativa |
| Columnas ~80% vacías (representante legal, CIIU) | RM | Informativa |

---

**Autor:** Jaime Usuga  
**Fecha:** Septiembre 2026