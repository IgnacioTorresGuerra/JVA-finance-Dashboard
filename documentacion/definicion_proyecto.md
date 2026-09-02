# Definición del proyecto

## Escenario

JVA Servicios SPA es una empresa chilena de servicios industriales: recibe equipos de clientes —bombas, motores eléctricos, rotores, impulsores—, los evalúa, los repara y los despacha. El negocio tiene dos aristas que hasta ahora se miraban por separado: la **operativa** (órdenes de trabajo, qué equipo entró, en qué estado salió) y la **financiera** (qué se facturó, a quién, y si se pagó).

La información vive en planillas Excel que se mantienen a mano. No hay ERP ni base de datos transaccional.

## Problema de negocio

La dirección no tiene una respuesta rápida a tres preguntas que se hace todos los meses:

1. ¿Cómo viene el mes en facturación, comparado con un período equivalente?
2. ¿Quién nos debe, cuánto, y hace cuánto?
3. ¿Los clientes están pagando mejor o peor que antes?

Responderlas implicaba abrir la planilla, filtrar a mano y sacar cuentas. El resultado dependía de quién lo hiciera y cuándo, y no era reproducible.

## Problema analítico

Construir un modelo dimensional sobre las planillas existentes que permita:

- **Análisis descriptivo** — qué pasó: facturación por período, por cliente, por producto; cartera vencida; comportamiento de pago.
- **Análisis diagnóstico** — por qué: qué clientes explican una caída, si un atraso es puntual o una tendencia, si el problema es de volumen o de precio.

Con una restricción de diseño explícita: **la vista de entrada tiene que ser legible en diez segundos, sin interactuar.** Quien la mira habitualmente no va a tocar un slicer.

## Stakeholders

**Principal**

- **Gerencia / dueño.** Necesita el estado del mes en curso de un vistazo, y una alerta cuando algo se sale de lo normal. Es el usuario de la página Resumen.

**Secundarios**

- **Administración y cobranza.** Necesita saber a quién llamar hoy y qué factura vence esta semana. Usa los paneles de cobranza de Resumen.
- **Área comercial.** Necesita entender de dónde vienen los ingresos, qué clientes concentran facturación y cómo evoluciona el ticket. Usa la página Ventas.
- **Quien mantiene las planillas.** Se beneficia indirectamente: los controles de calidad del pipeline le devuelven los registros inconsistentes en vez de dejarlos pasar silenciosamente al reporte.

## Alcance

**Dentro:** facturación (emisión, vencimiento, pago), cartera de clientes, órdenes de trabajo, y el resumen mensual de ventas y compras. Período 2017–2026 en operaciones, 2020–2026 en facturación.

**Fuera:** costos por orden de trabajo, márgenes por servicio, remuneraciones, inventario de repuestos. No están en las fuentes disponibles.

## Preguntas que el dashboard responde

| Pregunta | Dónde se responde |
|---|---|
| ¿Cuánto facturamos este mes? | Resumen — KPI Ventas mensuales |
| ¿Vamos mejor o peor que el año pasado? | Resumen — Variación; Ventas — Variación YoY |
| ¿Qué fracción de lo facturado está vencida? | Resumen — Morosidad |
| ¿Cuánta plata está en juego ahora? | Resumen — Monto en riesgo |
| ¿A quién le cobro hoy? | Resumen — Prioridad de cobranza |
| ¿Qué vence esta semana? | Resumen — Facturas por vencer |
| ¿De dónde vienen los ingresos en el tiempo? | Ventas — gráfico principal |
| ¿Qué clientes concentran la facturación? | Ventas — Top clientes, Participación Top 3 |
| ¿Cuánto tardan en pagarnos? | Cobranzas — DPD y Días de cobro |
| ¿Está mejorando el cumplimiento de pago? | Cobranzas — gráfico principal |
| ¿Qué clientes pagan más tarde? | Cobranzas — ranking por DPD |

## Limitaciones

- **La fuente es una planilla mantenida a mano.** Los problemas de calidad documentados en [`calidad_datos.md`](calidad_datos.md) son estructurales: se mitigan con validaciones en el pipeline, no se eliminan.
- **No hay costos**, así que no hay rentabilidad por servicio ni margen por cliente. El análisis llega hasta la facturación.
- **La cartera vencida real es muy pequeña** en este negocio. Por eso la página de Cobranzas se enfocó en comportamiento de pago histórico y no en un panel de antigüedad de saldos, que habría quedado casi vacío.
- **Los datos de este repositorio son sintéticos.** Ver [`anonimizacion.md`](anonimizacion.md).
