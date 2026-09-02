# Diccionario de KPIs

38 medidas DAX, en dos tablas de medidas (`_Medidas Ventas` y `_Medidas Cobranza`) organizadas por carpeta de visualización.

Una convención recorre todo el modelo: **las medidas con sufijo `(Mes Actual)` ignoran los slicers a propósito.** Anclan a `TODAY()` y envuelven el cálculo en `REMOVEFILTERS`. Son las que alimentan la página Resumen, que debe mostrar siempre el mes en curso sin importar qué haya filtrado alguien. El resto responde al contexto de filtro con normalidad.

---

## Ventas — base

| Medida | Formato | Qué responde |
|---|---|---|
| `Ventas Totales` | `$#,##0` | Suma de `Total`. Es la medida base de la que dependen casi todas las demás. |
| `Cantidad Vendida` | `#,##0` | Conteo de facturas del contexto. |
| `Ticket promedio` | `$#,##0` | Ventas ÷ cantidad. Distingue si un cambio en facturación viene de volumen o de precio. |
| `Ventas acumuladas` | `$#,##0` | Acumulado dentro del año en curso hasta la fecha del contexto. |

## Ventas — comparación temporal

| Medida | Formato | Qué responde |
|---|---|---|
| `Ventas Año Anterior` | `$#,##0` | Mismo período del año anterior, **acotado a la última fecha con ventas reales**. |
| `Variacion % YoY` | `+0.0%;-0.0%` | Variación contra esa base. Es la KPI de la página Ventas. |
| `Ventas Periodo Anterior` | `$#,##0` | ⚠️ Versión antigua. Ver la nota al final. |
| `Variacion ventas`, `Variacion %` | `$#,##0`, `0.0%` | ⚠️ Derivadas de la anterior. Misma advertencia. |

El acotamiento de `Ventas Año Anterior` no es un detalle: sin él, al filtrar por año se compara un año incompleto contra uno completo. Ver [`calidad_datos.md`](calidad_datos.md), hallazgo 4.

```dax
Ventas Año Anterior =
VAR _corte = CALCULATE(MAX(Fact_Facturas[Fecha_Emision]), ALL(Fact_Facturas), ALL(Calendario))
RETURN
CALCULATE(
    [Ventas Totales],
    SAMEPERIODLASTYEAR(FILTER(VALUES(Calendario[Fecha]), Calendario[Fecha] <= _corte))
)
```

## Ventas — análisis por cliente

| Medida | Formato | Qué responde |
|---|---|---|
| `Participacion Ventas` | `0.0%` | Peso del cliente en la facturación del contexto. |
| `Participacion Cliente` | `0.0%` | Peso del cliente medido en cantidad de facturas, no en monto. |
| `Participacion Top3 Clientes` | `0.0%` | Concentración: cuánto pesan los tres mayores. Es una medida de **riesgo comercial**, no de tamaño. |
| `Participacion Acumulada` | `0.0%` | Acumulado del ranking, para lectura tipo Pareto. |
| `Ranking Cliente` | `0` | Posición por facturación. |

## Ventas — página Resumen (ancladas al mes actual)

| Medida | Formato | Qué responde |
|---|---|---|
| `Ventas Totales (Mes Actual)` | `$#,##0` | Facturación del mes calendario en curso. |
| `Variacion % (Mes Actual)` | `+0.0%;-0.0%` | Contra el mismo mes del año anterior. |
| `Ventas Totales (Semana, Mes Actual)` | `$#,##0` | Alimenta el gráfico semanal, que se reinicia cada mes. |
| `Ventas Totales (Semana Actual)` | `$#,##0` | Solo la semana en curso. |
| `Ventas Totales (Mes Actual, por Cliente)` | `$#,##0` | Ranking de clientes del mes. |

## Textos dinámicos

| Medida | Qué hace |
|---|---|
| `Resumen Ejecutivo` | Genera la frase de alerta del encabezado a partir de la variación del mes. |
| `Resumen Ejecutivo Color` | Color del banner según el signo. |
| `Filtros Aplicados` | Chip que refleja los slicers. Lee la jerarquía de fechas automática — **solo sirve en Resumen**. |
| `Filtros Aplicados (Calendario)` | Variante para Ventas y Cobranzas, que leen `Calendario`. Agrega el caso `Varios` cuando hay más de un valor. |
| `Última Actualización` | Marca de tiempo del último refresco. |

---

## Cobranza — cartera abierta

Miden lo que está **pendiente ahora**. Todas excluyen `En demanda` y `Sin registrar`.

| Medida | Formato | Qué responde |
|---|---|---|
| `% Morosidad` | `0.0%` | Facturas vencidas sobre facturas abiertas. Es un porcentaje, así que detecta cultura de pago con independencia del monto. |
| `Monto en Riesgo` | `$#,##0` | Pesos vencidos y sin pagar. El complemento en monto del anterior. |
| `DPD Promedio` | `0` | Días de atraso promedio de lo vencido. |
| `DPD Maximo` | `0` | Peor caso abierto. |
| `DPD Ultimos 3M` | `0` | DPD de los últimos tres meses, sobre fecha de **vencimiento** vía `USERELATIONSHIP`. |
| `Variacion Tendencia` | `0.0` | Diferencia entre el DPD reciente y el histórico: dice si empeora o mejora. |
| `Ranking Moroso` | `0` | Posición por monto en riesgo. Devuelve vacío si no hay riesgo, lo que hace que la tabla se auto-oculte. |

**`Monto en Riesgo` devuelve vacío a propósito cuando no hay riesgo.** No agregarle `+0`: ese vacío es lo que hace que la tabla de prioridad de cobranza muestre solo a quien efectivamente debe, en vez de 200 filas en cero.

## Cobranza — página Resumen (ancladas al mes actual)

| Medida | Formato | Qué responde |
|---|---|---|
| `% Morosidad (Mes Actual)` | `0.0%` | Morosidad del mes en curso. Lleva `+0` en el numerador para mostrar `0,0%` en vez de `--`. |
| `Monto en Riesgo (Mes Actual)` | `$#,##0` | Monto en riesgo del mes en curso. |
| `Monto (Vence Esta Semana)` | `$#,##0` | Panel preventivo: vence pronto y todavía no está atrasado. |

## Cobranza — comportamiento de pago histórico

Miden cómo se pagó lo que **ya se pagó**. Son la página Cobranzas y no se solapan con las anteriores.

| Medida | Formato | Qué responde |
|---|---|---|
| `DPD Promedio (Pagadas)` | `+0.0;-0.0;0.0` | Días de atraso promedio al momento de pagar. **Negativo = pagan antes del vencimiento.** |
| `% Pagadas con Atraso` | `0.0%` | Proporción pagada después del vencimiento. Es el indicador central de la página. |
| `DPD Maximo (Pagadas)` | `0` | Peor atraso del período filtrado. |
| `Dias Promedio de Cobro` | `0` | Días entre emisión y pago efectivo. Equivalente a DSO. |

`Dias Promedio de Cobro` usa resta de fechas en vez de `DATEDIFF`, para tolerar los pagos anticipados sin error.

---

## Nota sobre medidas obsoletas

`Ventas Periodo Anterior`, `Variacion ventas` y `Variacion %` **contienen un defecto conocido** y se conservan solo porque alguna vista antigua podría referenciarlas. No usarlas en desarrollos nuevos.

Usan `FILTER(ALL(Fact_Facturas), …)`, y ese `ALL` borra también el filtro que llega por relación desde `Dim_Clientes`. Con un cliente seleccionado comparan las ventas *de ese cliente* contra las de *todos* el año anterior. El reemplazo correcto es `Ventas Año Anterior` / `Variacion % YoY`.

Detalle completo en [`calidad_datos.md`](calidad_datos.md), hallazgo 3.
