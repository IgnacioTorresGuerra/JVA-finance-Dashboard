# Modelo dimensional

Esquema estrella: una dimensión de clientes y una de fechas, dos tablas de hechos, y dos tablas de medidas sin datos.

```
                    ┌──────────────────┐
                    │   Dim_Clientes   │
                    │  Cliente_ID (PK) │
                    └────────┬─────────┘
                       1     │     1
            ┌────────────────┴────────────────┐
            │ *                             * │
  ┌─────────┴──────────┐          ┌───────────┴────────┐
  │  Fact_Operaciones  │          │   Fact_Facturas    │
  │   grano: una OT    │          │ grano: una factura │
  └────────────────────┘          └───────────┬────────┘
                                          *   │
                                              │  Fecha_Emision  (ACTIVA)
                                              │  Fecha_Vencimiento (inactiva)
                                        1     │
                                    ┌─────────┴────────┐
                                    │    Calendario    │
                                    │   Fecha (PK)     │
                                    └──────────────────┘

  Fact_VentasCompras_Mensual  — sin relaciones, resumen contable independiente
```

## Tablas

| Tabla | Tipo | Grano | Filas |
|---|---|---|---|
| `Dim_Clientes` | dimensión | un cliente | 213 |
| `Calendario` | dimensión de fechas | un día | 2017–2028 |
| `Fact_Facturas` | hechos | una factura | 1.169 (carga 1.117) |
| `Fact_Operaciones` | hechos | una orden de trabajo | 1.722 |
| `Fact_VentasCompras_Mensual` | hechos | un mes | 84 |
| `_Medidas Ventas`, `_Medidas Cobranza` | contenedores | — | sin datos |
| `RefreshDate` | utilitaria | marca de refresco | 1 |

## Relaciones

| Desde | Hacia | Cardinalidad | Estado |
|---|---|---|---|
| `Fact_Facturas[Cliente_ID]` | `Dim_Clientes[Cliente_ID]` | * : 1 | activa |
| `Fact_Operaciones[Cliente_ID]` | `Dim_Clientes[Cliente_ID]` | * : 1 | activa |
| `Fact_Facturas[Fecha_Emision]` | `Calendario[Fecha]` | * : 1 | **activa** |
| `Fact_Facturas[Fecha_Vencimiento]` | `Calendario[Fecha]` | * : 1 | **inactiva** |

Todas de dirección simple, de hechos hacia dimensión. Filtrado cruzado bidireccional en ninguna: introduce ambigüedad y no hace falta.

## La decisión central: emisión como eje activo

Entre `Fact_Facturas` y `Calendario` hay dos caminos posibles —fecha de emisión y fecha de vencimiento— y Power BI solo admite uno activo. Está activa **la de emisión**.

**Por qué.** La mayoría de las preguntas del negocio son sobre cuándo se *vendió*: cuánto facturamos en marzo, cómo viene el año contra el anterior, qué cliente creció. Todas esas se responden por fecha de emisión. El vencimiento importa en menos casos, y en esos se activa explícitamente:

```dax
DPD Ultimos 3M =
VAR UltimaFecha = CALCULATE(MAX(Fact_Facturas[Fecha_Vencimiento]))
RETURN
CALCULATE(
    [DPD Promedio],
    DATESINPERIOD(Calendario[Fecha], UltimaFecha, -3, MONTH),
    USERELATIONSHIP(Fact_Facturas[Fecha_Vencimiento], Calendario[Fecha])
)
```

Antes de este cambio, `Calendario` estaba relacionada **solo** con vencimiento y era una tabla de una única columna, usada por una sola medida. Las ventas colgaban de la jerarquía de fechas automática de Power BI, lo que impide escribir inteligencia de tiempo limpia: no hay forma decente de usar `SAMEPERIODLASTYEAR` sobre una jerarquía autogenerada y oculta.

Convertirla en dimensión real fue el prerrequisito de las páginas de Ventas y Cobranzas.

## `Calendario`

Tabla calculada, un día por fila entre 2017 y 2028.

| Columna | Definición | Visible |
|---|---|---|
| `Fecha` | `CALENDAR(...)` | sí |
| `Año` | `YEAR([Fecha])` | sí |
| `Mes` | nombre en español, ordenado por `MesNum` | sí |
| `MesNum` | `MONTH([Fecha])` | no — solo ordena |
| `AñoMes` | `Año × 100 + MesNum` | no — ordena cruzando años |
| `Periodo` | `"Ene-2026"`, ordenada por `AñoMes` | sí |

**`Mes` se construye con `SWITCH`, no con `FORMAT([Fecha], "MMMM")`.** `FORMAT` depende de la configuración regional del archivo: si cambia, los nombres salen en inglés. `SWITCH` es determinista.

**`Periodo` existe por una razón concreta.** El eje temporal anterior era `Fact_Facturas[Mes_Emision_Label]`, una columna de la tabla de hechos. Un eje que vive en la tabla de hechos no entrega contexto de fecha a la dimensión, así que `SAMEPERIODLASTYEAR` no tiene sobre qué operar. Con el eje en `Calendario[Periodo]`, la comparación interanual funciona.

## Jerarquías de fecha automáticas

El modelo conserva tres `LocalDateTable_*` que Power BI genera solo, sobre emisión, vencimiento y fecha de pago. **No se usan en las páginas nuevas** — quedaron porque los slicers originales de Resumen cuelgan de ellas.

Es deuda técnica conocida: lo correcto sería apagar la fecha automática y migrar esos slicers a `Calendario`. Se documenta en vez de esconderse.

## Consecuencia a tener presente

Como `Calendario` filtra por **emisión**, un slicer de fecha en una página de cobranza acota por cuándo se emitió la factura, no por cuándo vencía. Para la página Cobranzas eso es correcto: la pregunta es "las facturas que emitimos en 2024, ¿cómo se pagaron?". Si alguna vez se necesita lo contrario, hay que envolver la medida en `USERELATIONSHIP` como hace `DPD Ultimos 3M`.
