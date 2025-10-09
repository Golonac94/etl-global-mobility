#  ETL de Entregas de Producto — Prueba Técnica

##  Descripción General

Este proyecto implementa un **pipeline ETL automatizado en PySpark**, diseñado para procesar, limpiar y estandarizar registros de entregas de productos provenientes de distintos países.  
El flujo está **controlado mediante un archivo de configuración YAML (OmegaConf)** y **automatizado con GitHub Actions** para desplegar el notebook actualizado en **Databricks Community**, bajo una estructura de carpetas por ambiente (`DEV-GLOBAL-MOBILITY`, `PROD-GLOBAL-MOBILITY`).

El objetivo técnico es cumplir con la **prueba de flujo de datos** establecida, asegurando:
- Control total de parámetros (fechas, país, tipo de entrega, unidades).
- Ejecución modular y trazable (logging estructurado).
- Output estandarizado y auditado en formato **Delta Lake**.
---
 
## Objetivos

El pipeline cumple con los siguientes requerimientos definidos en la prueba técnica:
 
| Requisito | Cumplimiento |
|------------|---------------|
| 1. Lectura del CSV fuente | `paths.raw_csv` en `config.yaml` (lectura con esquema + parseo de fechas) |
| 2. Filtrado dinámico por rango de fechas | `start_date` y `end_date` parametrizados en `config.yaml` (`between(to_date(...))`) |
| 3. Salidas particionadas por `fec_proceso` | Tablas `UDV.data_ventas_depurado` y `UDV.data_ventas_obs`, partición `fec_proceso (DATE)` y `replaceWhere` por país + rango |
| 4. Parametrización con OmegaConf | Se usa OmegaConf para rutas, `params`, `delivery_types` y `unit_factors` (incluye `to_container` para `DictConfig`) |
| 5. Rango de fecha y país único por corrida | Desde OmegaConf se seleccionan `country`, `start_date`, `end_date`; soporta múltiples países y controla reproceso con `proccess: YES/NO` |
| 6. Conversión de unidades (`CS → 20 ST`) | `factor_map` desde `unit_factors` de `config.yaml` (conversión a unidad estándar *ST* y cálculo de `cant_unidad_medida`) |
| 7. Identificación de tipo de entrega | `ind_rutina` y `ind_bonificacion` según `delivery_types` en `config.yaml` (excluyendo otros en depurado) |
| 8. Estandarización de nombres (snake_case) | Nombres normalizados y prefijos coherentes (`cod_*`, `fec_*`) |
| 9. Control de calidad y observaciones | Split en `data_ventas_depurado` / `data_ventas_obs` con `motivo_obs`; logging en `/Volumes/.../etl_run_YYYYMMDD_HHMMSS.log` |
| 10. Columnas adicionales | `precio_unitario_unidades`, `cant_unidad_medida`, `origen_datos`, `fec_actualizacion_registro`, indicadores `ind_*` (según documento) |
| 11. Documentar flujo | README actual (este documento) |


---
##  Arquitectura Técnica del ETL


```
                       ┌──────────────────────────────┐
                       │      config/config.yaml      │
                       │  Parámetros globales del ETL │
                       │  (paths, params, factors...) │
                       └──────────────┬───────────────┘
                                      │
                                      ▼
                          ┌────────────────────────┐
                          │      Lectura CSV       │
                          │ Validación y tipado    │
                          │ (raw_csv → DataFrame)  │
                          └──────────┬─────────────┘
                                      │
                                      ▼
                          ┌─────────────────────────┐
                          │       Filtrado          │
                          │ country / fechas /      │
                          │ columnas requeridas     │
                          └───────────┬─────────────┘
                                      │
                                      ▼
                    ┌──────────────────────────────────────┐
                    │ Transformación (UDV - Silver Layer)  │
                    │ - Conversión unidades (factor_map)   │
                    │ - Indicadores rutina/bonif           │
                    │ - Cálculo precio_unitario_unidades   │
                    └──────────────────────────────────────┘
                                      │
                                      ▼
                          ┌────────────────────────┐
                          │       Filtrado         │
                          │ country / fechas /     │
                          │ columnas requeridas    │
                          └──────────┬─────────────┘
                                      │
                                      ▼
                  ┌─────────────────────────────────────────┐
                  │     Control de Calidad (UDV)            │
                  │ - Split válidos vs observados           │
                  │ - Log info / errores                    │
                  └───────────────┬─────────────────────────┘
                                  │
                 ┌────────────────┼─────────────────┐
                 │                                  │
                 ▼                                  ▼
  ┌─────────────────────────────┐     ┌─────────────────────────────┐
  │ UDV.data_ventas_depurado    │     │ UDV.data_ventas_obs         │
  │ (Registros válidos)         │     │ (Observaciones / errores)   │
  └──────────────┬──────────────┘     └─────────────────────────────┘
                 │                                   
                 |
                 ▼
 ┌──────────────────────────────────┐
 │   /Volumes/workspace/.../data/   │
 │   processed/ (Delta Outputs)     │
 │   Partición: fec_proceso (DATE)  │
 └──────────────────────────────────┘

```

---

##  Configuración con OmegaConf

Archivo: **`config/config.yaml`**

```yaml
paths:
  raw_csv: /Volumes/workspace/global_mobility/data/raw/global_mobility_data_entrega_productos.csv
  output_root: /Volumes/workspace/global_mobility/data/processed

params:
  - country: PE
    start_date: '2025-01-01'
    end_date: '2025-06-30'
    proccess: 'NO'

delivery_types:
  routine:
    - ZPRE
    - ZVE1
  bonus:
    - Z04
    - Z05

unit_factors:
  CS: 20
  ST: 1
```

###  Validaciones incluidas:
- Logging detallado de validación (errores e info).


  <img src='https://casaromo.duckdns.org/apps/files_sharing/publicpreview/5HMeHGqEnNfQSAY?file=/log.png&fileId=348824&x=2560&y=1440&a=true&etag=e8921e3e05b6309000613d311dd245f5' width='600'>

- Validación completa del archivo config.yaml
  -Estructura completa de secciones (`paths`, `params`, `delivery_types`, `unit_factors`).
  - Tipos de dato correctos (`params` es lista, fechas en formato ISO).
  - `unit_factors` convertido con `OmegaConf.to_container` para aceptar `DictConfig`.
  - Control de  valores nulos o negativos y registro en la tabla de obs.

  <img src='https://casaromo.duckdns.org/apps/files_sharing/publicpreview/5HMeHGqEnNfQSAY?file=/config_validacion.png&fileId=348834&x=2560&y=1440&a=true&etag=d1eda172ad22a49569f9813f4b2f96b6' width='600'>

- Validación de data 
  -Toda la data con observación se guarda como STRING en la tabla data_ventas_obs.

    <img src='https://casaromo.duckdns.org/apps/files_sharing/publicpreview/5HMeHGqEnNfQSAY?file=/tabla_obs.png&fileId=348844&x=2560&y=1440&a=true&etag=5c7d760a2c62d39446dfc5b1c84ff555' width='600'>

  - Materiales vacios (`null`). **- Detectada**
  - Registros sin tipo de entrega valida. **- Detectada**
  - Registros sin conversión de unidades valida.
  - Control de  valores nulos o negativos en cantidad y precio de obs. **- Detectada**

    <img src='https://casaromo.duckdns.org/apps/files_sharing/publicpreview/5HMeHGqEnNfQSAY?file=/errores.png&fileId=348843&x=2560&y=1440&a=true&etag=3445c30ed9e21a01de70a633a0c7a254' width='600'>


---


##  Control de Calidad y Logs

- Se genera un archivo log en cada ejecución:
  ```
  /Volumes/workspace/global_mobility/log/etl_run_YYYYMMDD_HHMMSS.log
  ```
- Cada etapa  registra:
  - info de ok.
  - errores con mensajes legibles (`log_error`).

  <img src='https://casaromo.duckdns.org/apps/files_sharing/publicpreview/5HMeHGqEnNfQSAY?file=/log_data.png&fileId=348859&x=2560&y=1440&a=true&etag=367a08e8066e302ffbb0cfe68a1842d7' width='600'>


---

##  Salidas

| Tabla | Descripción | Partición |
|--------|--------------|-----------|
| `RDV.data_ventas` | Carga inicial con los filtros basicos | `fec_proceso (DATE)` | 
| `UDV.data_ventas_depurado` | Registros válidos depurados | `fec_proceso (DATE)` | 
| `UDV.data_ventas_obs` | Registros descartados con `motivo_obs` | `fec_proceso (DATE)` | 
| `/Volumes/workspace/global_mobility/data/processed/` | Registros válidos depurados según solicitud | `fec_proceso (DATE)` | 



  <img src='https://casaromo.duckdns.org/apps/files_sharing/publicpreview/5HMeHGqEnNfQSAY?file=/tablas_finales.png&fileId=348866&x=2560&y=1440&a=true&etag=0c15cbf514b0c92237eb521d104a0308' width='600'>

  <img src='https://casaromo.duckdns.org/apps/files_sharing/publicpreview/5HMeHGqEnNfQSAY?file=/data_carpeta.png&fileId=348867&x=2560&y=1440&a=true&etag=f0cd1ce67bfa2ec2f35778c3296e7601' width='600'>

---

## 🚀 Automatización CI/CD (GitHub → Databricks)

El notebook se despliega automáticamente a la carpeta **`/Users/romarioparedest@outlook.com/PROD-GLOBAL-MOBILITY/notebooks/`** cada vez que se hace **merge de `dev` → `main`** en GitHub.

Workflow: `.github/workflows/update-notebook-on-main.yml`
- Usa los secretos:
  - `DATABRICKS_HOST_DEVELOP` 
  - `DATABRICKS_TOKEN_DEVELOP`

Esto garantiza que la versión en **PROD** siempre refleje el último merge aprobado en GitHub.

  <img src='https://casaromo.duckdns.org/apps/files_sharing/publicpreview/5HMeHGqEnNfQSAY?file=/workflow.png&fileId=348882&x=2560&y=1440&a=true&etag=bd37baaa4521ca9b5787325e4a77edd1' width='650'>


---

##  Columnas finales del dataset depurado

| Campo | Tipo | Descripción |Motivo creación|
|--------|------|-------------|-------------|
| `cod_pais` | STRING | País ISO-2 | Inicial |
| `fec_proceso` | DATE | Fecha proceso (partición) |Inicial |
| `cod_transporte` | STRING | Transporte |Inicial |
| `cod_ruta` | STRING | Ruta de entrega |Inicial |
| `cod_tipo_entrega` | STRING | Tipo entrega (`ZPRE`, `ZVE1`, etc.) |Inicial |
| `cod_material` | STRING | Material entregado |Inicial |
| `precio_unitario_unidades` | DECIMAL | Precio por unidad | Campo para poder evaluar precio por rango de fecha/envio |
| `mto_venta` | DECIMAL | Monto venta |Inicial |
| `cant_uni_medida` | DECIMAL | Cantidad según el cod_uni_medida |Inicial |
| `cod_uni_medida` | STRING | Unidad de medida |Inicial |
| `cant_unidades` | DECIMAL | Cantidad en unidades |Solicitud del test |
| `ind_rutina` | BOOLEAN | true si rutina, false caso contrario |Solicitud del test |
| `ind_bonificacion` | BOOLEAN | true si bonificación, false caso contrario |Solicitud del test |
| `origen_datos` | STRING |Archivo origen de datos |Campo para tracking  |
| `fec_actualizacion_registro` | DATE |Fecha de migracion de datos| Campo para tracking |

Campo adicional en data_ventas_obs

| Campo | Tipo | Descripción |Motivo creación|
|--------|------|-------------|-------------|
| `motivo_obs` | STRING | Motivo de depuración | Poder trackear el motivo de la separación de la data y poder corregir origen o considerar la depuración como regla de negocio|
---