# Calidad de datos: hallazgos y correcciones

Cinco defectos encontrados construyendo el dashboard. Ninguno se buscó: todos aparecieron al validar un número que se veía raro y no cuadraba con lo que el negocio sabía de sí mismo.

Cada uno sigue el mismo formato: **síntoma → causa → corrección → efecto medido.**

---

## 1. La morosidad reportaba 51,5 % en un negocio que cobra bien

**Síntoma.** El indicador de morosidad marcaba 51,5 %. La percepción del negocio era de una décima parte de eso.

**Causa.** Dos problemas superpuestos en la misma medida:

- Contaba como vencidas las facturas **pagadas con atraso**. Una factura pagada tarde ya no es riesgo: es historia.
- Incluía facturas en **litigio** de tres a cinco años de antigüedad, que no son gestión de cobranza corriente sino un proceso legal aparte.

**Corrección.** Se acotó el universo a facturas efectivamente abiertas, excluyendo los estados `En demanda` y `Sin registrar`. Los registros no se borraron: se conservan para respaldo.

**Efecto.** 51,5 % → ~10 %, consistente con lo que el negocio observaba.

---

## 2. Un pipeline ciego a lo que no era un valor de celda

**Síntoma.** Una factura marcada como impaga en la base estaba pagada en la realidad. La comparación automática entre planillas insistía en que ambas coincidían.

**Causa.** Y coincidían — en los **valores**. Quien concilia los pagos a veces anota la fecha real como **comentario de celda de Excel**, y deja el flag visible en `No`. Un script que lee `cell.value` no ve nada: el comentario vive en `cell.comment`, un canal distinto.

Un barrido de la planilla completa encontró **11 casos** con el mismo patrón. También apareció una segunda variante: la fecha de pago tipeada en la columna `Banco`, donde debería ir el nombre del banco.

**Corrección.** `estado_pago_facturacion()` ahora consulta tres fuentes en orden de confiabilidad:

1. Comentario de celda de la columna `Pagada` — es la anotación de quien concilia, la más confiable.
2. Fecha encontrada en la columna `Banco`.
3. El flag manual `Sí`/`No`, que solo decide cuando no hay ninguna otra señal.

**Efecto.** Once facturas pasaron de figurar impagas a su estado real, y el pipeline dejó de depender de un flag que se olvida de marcar.

**Lección transferible:** una comparación de valores entre dos fuentes puede dar "idénticas" y estar ocultando una diferencia real. Los datos de una planilla incluyen comentarios, formato y notas al margen.

---

## 3. Una variación porcentual que comparaba un cliente contra todos

**Síntoma.** Con un cliente seleccionado, la variación interanual daba números imposibles: clientes que claramente habían crecido aparecían con caídas del 90 %.

**Causa.** La medida usaba `FILTER(ALL(Fact_Facturas), …)`. Ese `ALL` no solo limpia los filtros de las columnas propias de la tabla: **también borra el que llega por relación desde `Dim_Clientes`**. El resultado era comparar las ventas *de un cliente* contra las ventas *de todos* el año anterior.

Verificado con datos: en un mes cualquiera, el denominador daba idéntico para todos los clientes — el total de la empresa.

| Cliente | Ventas | Base anterior (mala) | Var. mala | Var. correcta |
|---|---:|---:|---:|---:|
| Cliente A | 3.505.740 | 46.938.514 | −92,5 % | **+84,1 %** |
| Cliente B | 4.563.650 | 46.938.514 | −90,3 % | **+137,0 %** |

**Corrección.** Medida nueva basada en `SAMEPERIODLASTYEAR` sobre la dimensión de fechas, que respeta el contexto de filtro completo.

**Efecto.** La comparación por cliente pasó a ser válida. Las medidas antiguas se conservan marcadas como obsoletas.

---

## 4. Un interanual que comparaba ocho meses contra doce

**Síntoma.** Con el año en curso seleccionado, la variación marcaba **−59,3 %**. La caída real era menor.

**Causa.** Al filtrar por año, el contexto de fecha es el año calendario **completo**. `SAMEPERIODLASTYEAR` lo desplaza entero, así que traía los doce meses del año anterior — mientras el año en curso solo tenía datos hasta agosto. Ocho meses contra doce.

| Cálculo | Resultado |
|---|---:|
| Año en curso (ene–ago) vs. año anterior **completo** | **−59,3 %** |
| Año en curso (ene–ago) vs. año anterior **ene–ago** | **−38,1 %** |

Veintiún puntos porcentuales de diferencia, en el KPI principal de una página.

**Corrección.** Acotar el período anterior a la última fecha con ventas reales:

```dax
VAR _corte = CALCULATE(MAX(Fact_Facturas[Fecha_Emision]), ALL(Fact_Facturas), ALL(Calendario))
RETURN
CALCULATE([Ventas Totales],
    SAMEPERIODLASTYEAR(FILTER(VALUES(Calendario[Fecha]), Calendario[Fecha] <= _corte)))
```

**Efecto.** −59,3 % → −38,1 %. Y un efecto lateral útil: el gráfico dejó de dibujar la línea del año anterior sobre meses futuros sin datos, que sugería visualmente una caída que no existía.

---

## 5. Ocho facturas con el año al que le faltaba un dígito

**Síntoma.** El DPD promedio de un año daba **−25.849 días**. Unos setenta años de pago anticipado.

**Causa.** Comentarios de celda como `"Abonado en bco chile el 7-12-222"`: el año 2022 escrito como `222`, perdiendo el primer dígito al tipear. La expresión regular aceptaba `222` como año, y la corrección de años de dos dígitos (`if y < 100: y += 2000`) no lo cubría porque 222 es mayor que 100. `datetime` lo interpretaba como el año 222 d.C.

Ocho facturas afectadas. Basta una para arruinar un promedio.

| Año | DPD promedio con las corruptas | Sin ellas |
|---|---:|---:|
| 2022 | **−25.849 días** | **+3,9 días** |
| 2023 | **−3.190 días** | **+0,6 días** |

**Corrección.** Dos guardas en el pipeline, aplicadas tanto al comentario como a la fecha de la columna `Banco`:

```python
def _repara_anio(y):
    if y < 100:   return y + 2000          # "26"  -> 2026
    if y < 1000:  return 2000 + (y % 100)  # "222" -> 2022
    return y
```

Más una validación de plausibilidad que descarta cualquier fecha de pago fuera de un rango razonable y deja nota en el log de QA.

**Efecto.** Las ocho fechas se corrigieron y los promedios volvieron a ser interpretables. Con la guarda instalada, el caso no puede repetirse en silencio.

---

## Patrón común

Los cinco tienen la misma forma: **un número que se veía raro y que era más fácil explicar que investigar.**

Una morosidad del 51 % podía ser "el negocio cobra mal". Un −59 % podía ser "el año viene malo". Un DPD de −25.849 días es tan absurdo que salta a la vista, pero estaba escondido dentro de un promedio anual que nadie había mirado desagregado.

El hábito que los encontró todos fue el mismo: **cuando un número sorprende, verificarlo contra la fila que lo produce antes de aceptarlo.**

## Problemas conocidos sin resolver

- **Clientes duplicados por RUT compartido.** Razones sociales distintas comparten RUT, probablemente por error de tipeo. El pipeline los detecta y los anota en la bitácora de QA en vez de fusionarlos automáticamente: adivinar aquí es peor que no hacer nada.
- **Variantes de escritura del mismo cliente.** El mismo cliente aparece tipeado de varias formas en distintos documentos. La columna `Cliente_Registrado` conserva la forma original a propósito, y el modelo une por `Cliente_ID`.
- **Facturas pagadas antes de emitirse.** Un grupo son anticipos legítimos; al menos dos parecen el mismo error de año del hallazgo 5, pero con un año plausible, así que la guarda no los atrapa.
- **Tres jerarquías de fecha automáticas** que el modelo ya no necesita. Ver [`modelo_dimensional.md`](modelo_dimensional.md).
