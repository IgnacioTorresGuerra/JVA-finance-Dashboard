# Anonimización del set de datos

El dashboard se construyó sobre datos reales de una empresa en operación. Lo que se publica en este repositorio es un set sintético, generado por `scripts/anonimizar_demo.py` y comprobado por `scripts/verificar_demo.py`.

Este documento explica las reglas y **por qué cada una existe**. Varias no son obvias, y la que falta es la que produce la filtración.

---

## Cómo correrlo

```bash
python scripts/anonimizar_demo.py    # regenera data/BASE_DE_DATOS_JVA_demo.xlsx
python scripts/verificar_demo.py     # comprueba; sale con código != 0 si hay fuga
```

El primero necesita la planilla de producción, que **no** forma parte de este repositorio. El segundo es la compuerta: se corre antes de cada publicación.

---

## Las siete reglas

### 1. Mapear por llave de fila, no por texto

El mapeo va por `Cliente_ID` de cada fila. Si la fila tiene identificador, su nombre y su RUT se reescriben sin excepción.

**Por qué.** Un mapeo por texto compara contra la tabla de clientes, y en las tablas de hechos los nombres están **tipeados a mano**: abreviaturas, siglas, espaciado distinto alrededor del `&`, la razón social sin su forma legal. Ninguna de esas variantes calza contra el nombre canónico, así que pasan intactas. Es exactamente la clase de fallo que hay que evitar.

### 2. La llave del demo debe ser distinta a la de producción

`Cliente_ID` se reemplaza por un espacio de identificadores nuevo y barajado, sin correspondencia posicional con el original.

**Por qué.** Si la llave se conserva, anonimizar los nombres no anonimiza nada. Basta con que **una** tabla traiga la llave junto a algún rastro real para reconstruir el mapa completo cruzando hojas. La llave compartida es un canal de filtración por sí sola.

### 3. Todas las hojas, no solo las que tienen una columna llamada "nombre"

Los nombres de clientes no viven únicamente en la tabla de clientes. Aparecen en descripciones de equipos, en motivos de ingreso, en notas de estado, en bitácoras de revisión y en los ejemplos de la propia documentación del esquema.

**Por qué.** Anonimizar las hojas evidentes y dar el archivo por bueno deja el resto expuesto. Se recorre el libro completo.

### 4. Las marcas de equipos también pueden ser clientes

Las descripciones de equipos nombran fabricantes de bombas y motores. Varios de esos fabricantes **son además clientes de la empresa**.

**Por qué.** Dejar la marca en la descripción delata al cliente de esa orden de trabajo aunque el nombre esté seudonimizado. La marca es un identificador indirecto.

### 5. El mapa de sustitución se deriva de la fuente, nunca se escribe a mano

El diccionario que traduce nombres reales a ficticios se construye a partir de la planilla de producción en tiempo de ejecución.

**Por qué.** Dos razones, y la segunda es la importante:

- Un mapa manual siempre deja residuos: la marca suelta dentro de una descripción, la sigla, la razón social sin sufijo.
- **Un mapa escrito a mano queda en el código, y el código se publica.** Una versión de este script traía un diccionario de marcas reales para que las sustituciones quedaran más prolijas — con el efecto de que el script encargado de ocultar los nombres los publicaba él mismo.

### 6. Escalar montos, no tocar fechas

Los montos se multiplican por un factor único. Las fechas quedan intactas.

**Por qué.** Un factor único preserva todas las proporciones, así que **cada porcentaje del dashboard sigue siendo exacto**: participaciones, variaciones interanuales, morosidad. Solo los valores absolutos dejan de ser los de la empresa.

Las fechas no se tocan porque los indicadores de comportamiento de pago —días de atraso, días de cobro— **son** la diferencia entre fechas. Alterarlas destruiría el hallazgo principal de la página de Cobranzas.

### 7. Verificar, siempre, con un script independiente

`verificar_demo.py` recorre cada celda de cada hoja, más todos los archivos de texto del repositorio, buscando cualquier nombre o RUT real. Comprueba además que las llaves no se solapen con producción y que ninguna hoja de datos sea idéntica a la de origen.

**Por qué.** No comparte código con el anonimizador, a propósito: si el anonimizador tiene un error de lógica, el verificador debe detectarlo igual. Sale con código distinto de cero cuando encuentra algo, así que sirve como compuerta automática.

---

## Qué comprueba el verificador

| Control | Criterio |
|---|---|
| Nombres reales en cualquier celda | cero coincidencias |
| RUTs reales en cualquier celda | cero coincidencias |
| Solape de llaves con producción | cero |
| Filas idénticas a producción, por hoja | menos del 5 % |
| Montos que coincidan con los reales | cero |
| Nombres reales en archivos de texto | cero |

Los valores centinela —`Sin RUT`, `NULO`, `N/A`— se excluyen del universo de secretos: existen en producción y es correcto que sigan en el demo.

---

## Qué se preserva

La anonimización mantiene todo lo que hace útil al set como demostración:

- **El volumen y la forma.** Mismo número de filas, mismas columnas, mismos tipos.
- **Los problemas de calidad.** Las variantes de escritura del mismo cliente se preservan como variantes del nombre **ficticio**, así que el dashboard sigue mostrando el mismo desafío de normalización.
- **Toda la señal temporal.** Fechas intactas.
- **Todos los porcentajes.** Consecuencia del factor único de escala.
- **La mezcla de tipos de cliente.** Los clientes que en producción son personas naturales siguen siendo personas ficticias, no empresas.

## Qué se sacrifica

- **Los montos absolutos** no son los reales.
- **La bitácora de calidad de datos** se reemplaza por una nota genérica: doscientas filas de prosa sobre clientes y personas no se sanean de forma confiable a punta de sustituciones.
- **Los nombres ficticios no tienen historia.** Un cliente real puede reconocerse por su rubro o su patrón de compra; uno inventado, no.

---

## Nota

Este documento existe porque la primera versión de la anonimización de este proyecto **falló**: cubrió dos de seis hojas y conservó la llave de producción, lo que dejaba las otras dos reversibles por cruce. Se detectó auditando el repositorio antes de una publicación posterior.

Las siete reglas de arriba son la respuesta a ese error. La séptima —verificar con un script independiente— es la que lo habría evitado, y es la que conviene instalar primero.
