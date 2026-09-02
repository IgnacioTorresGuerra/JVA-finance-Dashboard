# Diccionario de datos

Describe `data/BASE_DE_DATOS_JVA_demo.xlsx`, el archivo que lee el modelo semántico. Seis hojas; cuatro se cargan al modelo y dos son documentación interna.

En producción este archivo lo genera `scripts/sync_facturacion_a_base.py` a partir de la planilla que se mantiene a mano. La versión de este repositorio es sintética — ver [`anonimizacion.md`](anonimizacion.md).

---

## `Dim_Clientes` — 213 filas

Dimensión de clientes. **Grano: un cliente.**

| Columna | Tipo | Descripción |
|---|---|---|
| `Cliente_ID` | texto | **Llave primaria.** Formato `C-####`. En este repositorio es distinta a la de producción, a propósito. |
| `Nombre_Cliente` | texto | Razón social canónica. Es la que muestra el dashboard. |
| `RUT` | texto | Identificador tributario con dígito verificador. Puede ser `Sin RUT`. |
| `N_Ordenes_Trabajo` | entero | Conteo de OT asociadas. Precalculado por el pipeline. |
| `N_Facturas` | entero | Conteo de facturas asociadas. Precalculado por el pipeline. |

Incluye personas naturales, no solo empresas. Una fila con `Nombre_Cliente = "-"` actúa como cliente no identificado.

---

## `Fact_Facturas` — 1.169 filas

Hechos financieros. **Grano: una factura.** Es la tabla que alimenta Ventas y Cobranzas.

| Columna | Tipo | Descripción |
|---|---|---|
| `Factura_ID` | entero | Llave técnica. |
| `N_Correlativo`, `N_Factura` | entero | Numeración del documento. |
| `Tipo_Documento` | texto | `Factura`, nota de crédito, etc. |
| `Cliente_ID` | texto | **Llave foránea** a `Dim_Clientes`. |
| `Cliente_Registrado` | texto | Nombre **tal como fue tipeado** en el documento. Puede diferir del canónico: es la fuente de las variantes de escritura. |
| `RUT_Registrado` | texto | RUT tal como fue tipeado. |
| `Fecha_Emision` | fecha | **Eje temporal principal.** Define la relación activa con `Calendario`. |
| `Fecha_Vencimiento` | fecha | Base del cálculo de atraso. |
| `Neto`, `IVA`, `Total` | entero | Montos en pesos chilenos. `IVA = Neto × 0,19`, `Total = Neto + IVA`. |
| `Orden_Compra`, `Guia_Despacho` | texto | Documentos de referencia del cliente. Frecuentemente vacíos. |
| `Cedida`, `Factoring` | texto | Marcas de cesión del documento. |
| `Pagada` | texto | **Estado de pago.** Cuatro valores: `Sí`, `No`, `En demanda`, `Sin registrar`. |
| `Banco` | texto | Banco declarado. A veces contiene una fecha por error de tipeo — ver `calidad_datos.md`. |
| `Fecha_Real_Pago` | fecha | Fecha efectiva del pago. Vacía si no se pagó o no se registró. |
| `Banco_Pago` | texto | Banco conciliado por el pipeline. |

### Sobre `Pagada`

Los cuatro estados no son intercambiables y las medidas los tratan distinto:

- **`Sí`** — pagada. Entra en el análisis de comportamiento de pago.
- **`No`** — pendiente. Entra en morosidad y monto en riesgo si está vencida.
- **`En demanda`** — judicializada. **Se excluye** de morosidad: no es gestión de cobranza corriente.
- **`Sin registrar`** — sin estado asignado. **Se excluye** por la misma razón.

Excluir los dos últimos fue una corrección deliberada; ver [`calidad_datos.md`](calidad_datos.md).

### Columnas calculadas del modelo

No están en el Excel: las crea el modelo semántico sobre esta tabla.

| Columna | Definición |
|---|---|
| `Dias_Atraso` | Días entre vencimiento y pago. Si no hay pago, entre vencimiento y hoy. Negativo = pago anticipado. |
| `Anio_Emision`, `Mes_Emision` | Componentes de `Fecha_Emision`. |
| `Rango_Mora` | Tramo de antigüedad: `Al día`, `1-30 días`, `31-60`, `61-90`, `+90 días`, `Pagada`. |
| `Semana_Inicio`, `Semana_Emision` | Lunes de la semana de emisión y su etiqueta. |
| `Es_Mes_Actual`, `Es_Semana_Actual` | Banderas que anclan los paneles de Resumen al período en curso. |

---

## `Fact_Operaciones` — 1.722 filas

Hechos operativos. **Grano: una orden de trabajo (OT).** No se usa en las tres páginas actuales, pero forma parte del modelo.

| Columna | Tipo | Descripción |
|---|---|---|
| `OT_ID`, `N_OT` | entero / texto | Identificadores de la orden. |
| `OT_Multicomponente` | texto | Si la OT agrupa varios equipos. |
| `Alerta_OT_Cliente_Distinto` | texto | Marca de control: el cliente de la OT no coincide con el de la factura. |
| `Fecha_Ingreso` | fecha | Entrada del equipo al taller. Desde 2017. |
| `Cliente_ID` | texto | **Llave foránea** a `Dim_Clientes`. |
| `Cliente_Registrado`, `RUT_Registrado` | texto | Datos tal como fueron tipeados. |
| `Equipo_Componente` | texto | Descripción del equipo recibido. |
| `Motivo_Ingreso` | texto | Motivo normalizado. |
| `Motivo_Ingreso_Original` | texto | Motivo tal como se escribió. |
| `Fecha_Despacho`, `Fecha_Informe` | fecha | Hitos de salida. |
| `Estado`, `Estado_Nota` | texto | Estado de la OT y su comentario. |

---

## `Fact_VentasCompras_Mensual` — 84 filas

Resumen contable mensual, independiente de las otras tablas. **Grano: un mes.**

| Columna | Tipo | Descripción |
|---|---|---|
| `Anio`, `Mes` | entero / texto | Período. |
| `Ventas`, `Compras` | entero | Totales mensuales en pesos. |
| `PPM_5pct`, `PPM_Voluntario` | entero | Pagos provisionales mensuales. |

No se relaciona por `Cliente_ID`. Si se cruza con las otras tablas, es por fecha.

---

## `Diccionario de Datos` y `QA_Calidad_Datos`

Hojas de documentación interna, no se cargan al modelo.

`QA_Calidad_Datos` es la bitácora donde el pipeline escribe sus hallazgos automáticos: clientes con RUT compartido, OT sin RUT válido, facturas que no pudieron asociarse. En este repositorio está reemplazada por una nota genérica, porque en producción nombra clientes y personas reales.

---

## Reglas de integridad

1. Todo `Cliente_ID` en las tablas de hechos existe en `Dim_Clientes`.
2. `Fecha_Vencimiento` ≥ `Fecha_Emision`, salvo documentos emitidos con vencimiento inmediato.
3. `Total = Neto + IVA`, con IVA al 19 %.
4. `Fecha_Real_Pago` no vacía implica `Pagada = Sí` — salvo los casos que documenta `calidad_datos.md`, que fue precisamente lo que hubo que corregir.
5. El modelo solo carga filas con `Fecha_Emision`, `Fecha_Vencimiento` y `Neto` presentes. Por eso carga 1.117 de 1.169 filas.
