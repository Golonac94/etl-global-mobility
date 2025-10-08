#  ETL de Entregas de Producto — Prueba Técnica (Grupo Mariposa)

##  Descripción General

Este proyecto implementa un **flujo ETL parametrizable en PySpark**, desarrollado como parte de una **prueba técnica de ingeniería de datos**.  
El objetivo es procesar y depurar registros de entregas de producto provenientes de distintos países, utilizando una arquitectura flexible controlada mediante **OmegaConf (YAML)** para definir parámetros como fechas, país y rutas de salida.

El flujo se diseñó para operar en entornos **develop / qa / main**, y genera salidas **particionadas por `fecha_proceso`** bajo una estructura estandarizada en formato **Delta Lake**.

---
 
## Objetivos

El pipeline cumple con los siguientes requerimientos definidos en la prueba técnica:

1. Lectura de archivo CSV fuente.  
2. Filtrado por **rango de fechas dinámico** (`start_date`, `end_date`) usando OmegaConf.  
3. Escritura de salidas **particionadas por `fecha_proceso`** en `data/processed/${fecha_proceso}`.  
4. Parametrización por país (`country`).  
5. Estandarización de unidades (`CS` → 20 `ST`).  
6. Identificación de entregas:
   - `ZPRE`, `ZVE1` → entregas de rutina.  
   - `Z04`, `Z05` → entregas con bonificación.  
7. Filtrado de registros no válidos y generación de tabla de observaciones.  
8. Estandarización de nombres de columnas bajo convención `snake_case`.  
9. Detección y eliminación de anomalías.  
10. Columnas adicionales fundamentadas:
    - `precio_unitario_unidades`
    - `ind_rutina`
    - `ind_bonificacion`

---

## ⚙️ Arquitectura General del Flujo

<!-- TODO: Inserta aquí un diagrama tipo "data flow" mostrando las etapas:
CSV → Bronze (lectura y limpieza básica) → Silver (transformaciones / unidad estandarizada / flags) → Gold (salida particionada por fecha_proceso)
Usa draw.io o mermaid y guarda como `docs/etl_diagram.png` -->

**Etapas principales:**

1. **Lectura:**  
   Se lee el dataset CSV desde la ruta indicada en `config.yaml` (`paths.raw_csv`).

2. **Filtrado:**  
   Aplicación dinámica de filtros por país y rango de fechas definidos en `params`.

3. **Transformación:**  
   - Conversión de unidades (`CS → ST * 20`).  
   - Generación de columnas `ind_rutina`, `ind_bonificacion`.  
   - Cálculo de `precio_unitario_unidades = mto_venta / cantidad_en_unidades`.  
   - Normalización de columnas (`snake_case`, trim, upper/lower según tipo).

4. **Control de Calidad:**  
   - Se generan dos DataFrames:
     - `data_ventas_depurado`: registros válidos.  
     - `data_ventas_obs`: registros descartados con motivo de observación.  
   - Ambas tablas se guardan en formato **Delta**, particionadas por `fec_proceso`.

5. **Escritura:**  
   Los resultados se guardan en:
   ```
   data/processed/{fecha_proceso}/data_ventas_depurado
   data/processed/{fecha_proceso}/data_ventas_obs
   ```

---

## 🧾 Configuración (OmegaConf)

El archivo `config/config.yaml` define todos los parámetros del flujo, la conversion unidades y los tipos de delivery validos  :

```yaml

paths:
  raw_csv: "/Volumes/workspace/global_mobility/data/raw/global_mobility_data_entrega_productos.csv"
  output_root: "/Volumes/workspace/global_mobility/data/processed"   

params:    #PARAMETROS EDICION
  start_date: "2025-01-01"        
  end_date:   "2025-06-30"
  country:    "PE"              

delivery_types:
    routine: ["ZPRE", "ZVE1"]
    bonus:   ["Z04", "Z05"]

unit_factors:
  "CS": 20
  "ST": 1

```

El flujo puede ejecutarse para cualquier rango de fechas y país sin modificar el código.

---

## Principales Transformaciones

| Columna origen | Regla aplicada | Columna resultante | Obsevación |
|----------------|----------------|--------------------|--------------------|
| `unidad` | CS = 20 ST, ST = 1 | `cant_unidad_medida` | Cualquier otro valor da `null` y se guarda en obs |
| `mto_venta` / `cant_unidad_medida` | Precio unitario redondeado a 3 decimales | `precio_unitario_unidades` | Cantidad igual 0 dara `null` y se guarda en obs |
| `tipo_entrega` | Listado de config.yml | `ind_rutina`, `ind_bonificacion` |Cantidad igual 0 dara `null` y se guarda en obs |
| — | Anomalías detectadas | `motivo_obs` (en tabla observaciones) |

---

## 🧪 Validaciones y Observaciones

- Se crean **dos salidas**:
  - `data_ventas_depurado`: registros válidos.
  - `data_ventas_obs`: registros descartados con detalle del motivo.

- Ejemplo de escritura:
  ```python
  df_clean.write.format("delta").mode("overwrite").partitionBy("fec_proceso").save(out_path)
  df_obs.write.format("delta").mode("overwrite").partitionBy("fec_proceso").save(obs_path)
  ```

<!-- TODO: agrega una captura de pantalla del resultado en Databricks mostrando las particiones o vista Delta -->

---

## 🧠 Estándar de Columnas Finales

| Campo | Tipo | Descripción |
|--------|------|-------------|
| `cod_pais` | STRING | Código ISO del país |
| `fec_proceso` | DATE | Fecha de procesamiento (partición) |
| `cod_transporte` | STRING | Identificador del transporte |
| `cod_ruta` | STRING | Código de ruta |
| `cod_tipo_entrega` | STRING | Tipo de entrega (ZPRE, Z04, etc.) |
| `cod_material` | STRING | Material entregado |
| `mto_venta` | DECIMAL | Monto de venta |
| `cant_unidad_medida` | DECIMAL | Cantidad estandarizada (ST) |
| `cod_unidad_medida` | STRING | Unidad estandarizada (ST) |
| `precio_unitario_unidades` | DECIMAL(21,3) | Precio por unidad estándar |
| `ind_rutina` | INT | 1 = rutina, 0 = no |
| `ind_bonificacion` | INT | 1 = bonificación, 0 = no |
| `origen_datos` | STRING | Archivo o fuente original |
| `fec_actualizacion_registro` | STRING | Fecha de última actualización |

---
