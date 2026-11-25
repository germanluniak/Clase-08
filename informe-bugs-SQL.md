# Informe de Práctica Clase-08  Investigar bugs con SQL en MiniModa

* **Entorno:** Base SQLite de práctica de **MiniModa v1** (y vista de documentos JSON estilo MongoDB en la colección `pedidos`), con registro de defectos en **Qase**.
* **Reglas de referencia:**
  1. El descuento del cupón es un porcentaje del `subtotal`; el envío no tiene descuento.
  2. Envío: `$3.500` si el `subtotal` es menor a `$60.000`; si no, gratis (`$0`).
  3. `total = subtotal - descuento + envio`.
  4. Un pago (`pago_ref`) corresponde a un solo pedido.
  5. No se vende más de lo que hay: `stock_actual` nunca es negativo.
  6. Un cupón vale entre `vigente_desde` y `vigente_hasta` (`CYBER20` vale hasta el `15/11` a las `23:59` de Argentina). Fechas en UTC (Argentina = UTC-3); montos en pesos.

## 1. Reto 1: Exploración inicial y anomalías detectadas

A partir de las consultas exploratorias ejecutadas sobre la base de práctica, se relevaron datos clave y anomalías directas contra las reglas del negocio:

* **Líder en unidades vendidas (`estado = 'pagado'`):** **Jean Mom Celeste** (`id = 3`) con **32 unidades** vendidas.
* **Pago duplicado (`pago_ref`):** `PAY-7F3K9Q` aparece en **2** pedidos pagados (infringe la Regla 4: un `pago_ref` debe corresponder a un solo pedido).
* **Stock negativo (`stock_actual < 0`):** **Campera Puffer Rosa** (`producto_id = 7`) figura con `stock_actual = -3` (infringe la Regla 5: `stock_actual` nunca es negativo).

## 2. Reto 2: Bug guiado — Descuento excesivo con el cupón `MODA15`

### Síntoma

Finanzas reportó que en noviembre el cupón `MODA15` descontó de más.

### Regla (Oráculo)

Regla 1: el descuento del cupón es un porcentaje del `subtotal` (`ROUND(subtotal * 0.15)` para un 15 %); el envío no tiene descuento.

### Hipótesis

* **Hipótesis inicial:** en algunos pedidos el descuento se calculó sobre algo más que el `subtotal`.
* **Hipótesis por canal:** el error depende exclusivamente del canal (`app` o `web`).
* **Hipótesis por versión + envío:** el error ocurre en la versión `2.4.0` cuando el pedido tiene envío (`envio > 0`).

### Consultas y resultados

#### Consulta 2A — Pedidos con `MODA15` donde `descuento <> ROUND(subtotal * 0.15)`

```sql
SELECT id, canal, version, subtotal, envio, descuento,
       ROUND(subtotal * 0.15) AS esperado,
       ROUND(subtotal * 0.15) - descuento AS diferencia
FROM pedidos
WHERE cupon = 'MODA15'
  AND descuento <> ROUND(subtotal * 0.15);
```

* **Resultado:** **16 filas** (`id` 1074, 1080, 1082, 1095, 1096, 1100, 1101, 1102, 1122, 1136, 1138, 1148, 1154, 1157, 1159 y 1171).
* En las **16 filas**, `version = '2.4.0'`, `envio = 3500`, y la columna `diferencia` (`esperado - descuento`) vale siempre **`-525`** (el descuento aplicado fue `$525` superior al 15 % del `subtotal`, cifra que coincide con el 15 % de `$3.500` de envío: `3500 * 0.15 = 525`).

#### Consulta 2B — Desglose por `version`, `canal` y `con_envio` (`envio > 0`)

```sql
SELECT version, canal, envio > 0 AS con_envio,
       COUNT(*) AS pedidos,
       SUM(descuento <> ROUND(subtotal * 0.15)) AS con_error
FROM pedidos
WHERE cupon = 'MODA15'
GROUP BY version, canal, con_envio;
```

* **Resultado:** **8 filas**:

| version | canal | con_envio | pedidos | con_error |
| :--- | :--- | :--- | :--- | :--- |
| 2.3.1 | app | 0 | 3 | 0 |
| 2.3.1 | app | 1 | 6 | 0 |
| 2.3.1 | web | 0 | 2 | 0 |
| 2.3.1 | web | 1 | 7 | 0 |
| 2.4.0 | app | 0 | 8 | 0 |
| 2.4.0 | app | 1 | 11 | 11 |
| 2.4.0 | web | 0 | 2 | 0 |
| 2.4.0 | web | 1 | 5 | 5 |

### Conclusión (Reto 2)

* **Qué se prueba (evidencia):** todos los pedidos con error en `MODA15` (**16 de 16**) corresponden a la combinación `version = '2.4.0'` y `con_envio = 1` (`envio > 0`), afectando tanto a `app` (`11/11`) como a `web` (`5/5`). En la versión `2.3.1` y en pedidos sin envío (`con_envio = 0`, incluida la versión `2.4.0`), `con_error` es `0`. Esto descarta que el bug dependa solo del canal.
* **Qué no se prueba (suposición):** la implementación exacta en código (por ejemplo, si se suma `envio` al `subtotal` antes de calcular el 15 %) y si ocurre lo mismo con otros cupones como `CYBER20`.

## 3. Reto 3: Bug abierto — Stock negativo en `Campera Puffer Rosa` (producto `7`)

### Síntoma

La **Campera Puffer Rosa** (`producto_id = 7`) figura con `stock_actual = -3`.

### Regla (Oráculo)

Regla 5: no se vende más de lo que hay; `stock_actual` nunca es negativo.

### Hipótesis

Se vendieron más unidades pagadas que el stock inicial disponible (o compras en ráfaga/concurrencia descontaron stock cuando ya estaba agotado).

### Consulta y resultado

```sql
SELECT id, sku, nombre, stock_inicial, stock_actual
FROM productos
WHERE id = 7;

SELECT p.id, p.creado_en, p.estado, i.cantidad
FROM items i
JOIN pedidos p ON p.id = i.pedido_id
WHERE i.producto_id = 7
ORDER BY p.creado_en;
```

* **Resultado:** **12 filas** de pedidos asociados al producto `7`:

| id | creado_en | estado | cantidad | Acumulado pagado | Stock vs. inicial (12) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1012 | 2026-11-02 14:10:11 | pagado | 1 | 1 | 11 |
| 1029 | 2026-11-04 22:41:09 | pagado | 2 | 3 | 9 |
| 1036 | 2026-11-06 13:05:50 | pagado | 1 | 4 | 8 |
| 1056 | 2026-11-08 19:22:31 | cancelado | 1 | 4 (no suma) | 8 |
| 1063 | 2026-11-09 21:47:02 | pagado | 1 | 5 | 7 |
| 1077 | 2026-11-11 16:30:44 | pagado | 2 | 7 | 5 |
| 1094 | 2026-11-13 12:02:19 | pagado | 1 | 8 | 4 |
| 1110 | 2026-11-14 18:55:40 | pagado | 2 | 10 | 2 |
| 1114 | 2026-11-14 23:09:13 | pagado | 2 | 12 | 0 (se agota) |
| 1115 | 2026-11-14 23:15:02 | pagado | 1 | 13 | -1 (se pasa) |
| 1116 | 2026-11-14 23:15:03 | pagado | 1 | 14 | -2 |
| 1117 | 2026-11-14 23:15:05 | pagado | 1 | 15 | -3 |

### Conclusión (Reto 3)

* **Qué se prueba (evidencia):** en los pedidos pagados del producto `7` se acumulan **15 unidades** (excluyendo el pedido `1056`, que está `cancelado`). Dado que `stock_actual` quedó en `-3`, el `stock_inicial` era de **12 unidades**, que se agota en el pedido **`1114`** (`2026-11-14 23:09:13`, acumulado `12`). **El primer pedido que se pasa del stock es el `1115`** (`2026-11-14 23:15:02`, cantidad `1`, acumulado `13`, stock `-1`), seguido por **`1116`** (`23:15:03`, stock `-2`) y **`1117`** (`23:15:05`, stock `-3`). Los pedidos que pasaron el stock (`1115`, `1116`, `1117`) comparten la misma fecha (`2026-11-14`), una ventana de **3 segundos** (`23:15:02` a `23:15:05`), estado `pagado` y `cantidad = 1`.
* **Qué no se prueba (suposición):** la causa técnica exacta en el código (por ejemplo, falta de bloqueo/concurrencia o ausencia de validación contra stock disponible al confirmar el pago).

## 4. Reto extra: Consulta NoSQL en `pedidos`

### Consulta ejecutada

```javascript
db.pedidos.find({ estado: "pagado", canal: "app", total: { $gt: 150000 } })
```

### Resultado (2 documentos)

1. **Pedido `_id: 1058`** (`2026-11-09 18:08:00`): cliente Lautaro Ferreyra (`La Plata`), `canal: "app"`, `version: "2.3.1"`, `estado: "pagado"`, `cupon: null`, `subtotal: 183000`, `descuento: 0`, `envio: 0`, `total: 183000`, `pago_ref: "PAY-9YTWGL"`. Ítems: 2 *Zapatillas Urbanas Blancas* (`$62.000` c/u) y 2 *Buzo Canguro Gris* (`$29.500` c/u).
2. **Pedido `_id: 1156`** (`2026-11-18 14:40:00`): cliente Rocío Cabrera (`Neuquén`), `canal: "app"`, `version: "2.4.0"`, `estado: "pagado"`, `cupon: null`, `subtotal: 169000`, `descuento: 0`, `envio: 0`, `total: 169000`, `pago_ref: "PAY-BASDB8"`. Ítems: 1 *Mochila Urbana Negra* (`$41.000`), 1 *Jean Mom Celeste* (`$38.000`) y 2 *Vestido Lino Arena* (`$45.000` c/u).

Ambos documentos cumplen con `estado: "pagado"`, `canal: "app"`, `total > 150000`, envío gratis (`$0` al tener `subtotal >= $60.000`) y la fórmula `total = subtotal - descuento + envio`.

## 5. Evidencia de defectos registrados en Qase (Reto 4)

Ambos defectos fueron cargados y verificados en **Qase** (`ISSUES > Defects`) bajo el proyecto **MINIMODAS**, en estado **`Open`** y severidad **`Normal`**, reportados y asignados a **Germán Luñak** el **25 Nov 2026**:

### Issue 1 — Cupón `MODA15`

* **Defect title:** Cupón MODA15 aplica descuento extra en versión 2.4.0 con envío (16 pedidos)
* **Estado en Qase:** `Open` | **Severity:** `Normal` | **Prioridad sugerida:** media
* * **Evidencia visual:** [issues/issue-1-Cupon-moda15.png](issues/issue-1-Cupon-moda15.png)
* **Actual result:**
  * **Entorno:** página de práctica de MiniModa, dataset v1.
  * **Requisito (Regla 1):** el descuento del cupón es un porcentaje del `subtotal`; el envío no tiene descuento.
  * **Pasos para reproducir:**
    1. Abrir la página de práctica de MiniModa v1.
    2. Pegar la consulta SQL de evidencia (2A y/o 2B).
    3. Tocar «Ejecutar».
  * **Resultado esperado:** 0 filas en la consulta 2A y 0 pedidos con error en todos los grupos de la consulta 2B.
  * **Resultado obtenido:** 16 filas en la consulta 2A (todos con diferencia de `-525` respecto a `ROUND(subtotal * 0.15)`, `version = '2.4.0'` y `envio = 3500`) y 16 pedidos con error en la consulta 2B (`11/11` en `app` y `5/5` en `web` dentro de `version = '2.4.0'` con `envio > 0`; 0 errores en versión `2.3.1` o sin envío).
  * **Evidencia:** consulta 2A y 16 pedidos afectados (`id` 1074, 1080, 1082, 1095, 1096, 1100, 1101, 1102, 1122, 1136, 1138, 1148, 1154, 1157, 1159 y 1171).
  * **Hipótesis (suposición no probada solo por SQL):** en la versión `2.4.0` con `envio > 0`, el cálculo del 15 % se estaría aplicando sobre `subtotal + envio` (`3500 * 0.15 = 525`).
  * **Falta confirmar con el equipo:** si el código de la versión `2.4.0` suma el envío al base de cálculo del cupón, si ocurre con otros cupones y cómo se ajustarán los 16 pedidos afectados.
  * **Prioridad sugerida:** media. En Qase Free el defecto no tiene campo de prioridad, así que va en este texto.

### Issue 2 — Stock negativo en Campera Puffer Rosa (`producto_id = 7`)

* **Defect title:** Campera Puffer Rosa (producto 7) queda con stock negativo (-3) tras pedidos pagados
* **Estado en Qase:** `Open` | **Severity:** `Normal` | **Prioridad sugerida:** alta
* * **Evidencia visual:** [issues/issue-2-stock-negativo-campera.png](issues/issue-2-stock-negativo-campera.png)
* **Actual result:**
  * **Entorno:** página de práctica de MiniModa, dataset v1.
  * **Requisito (Regla 5):** no se vende más de lo que hay; el stock no puede quedar negativo.
  * **Pasos para reproducir:**
    1. Abrir la página de práctica de MiniModa v1.
    2. Pegar las consultas SQL de evidencia para el producto `id = 7` (`Campera Puffer Rosa`).
    3. Tocar «Ejecutar».
  * **Resultado esperado:** el stock disponible no baja de 0 y no se completan ventas pagadas por encima del stock inicial (12 unidades).
  * **Resultado obtenido:** el producto `7` (`Campera Puffer Rosa`) tiene `stock_inicial = 12` y `stock_actual = -3`. En los pedidos pagados ordenados por `creado_en`, el stock llega a 0 en el pedido `1114` (`2026-11-14 23:09:13`, acumulado 12 unidades) y se pasa por primera vez en el pedido `1115` (`2026-11-14 23:15:02`, acumulado 13 unidades), continuando en los pedidos `1116` (`2026-11-14 23:15:03`) y `1117` (`2026-11-14 23:15:05`) hasta un total de 15 unidades pagadas (el pedido `1056` está cancelado y no suma).
  * **Evidencia:** consultas sobre `productos` (`id = 7`) e `items` + `pedidos` (`producto_id = 7` ordenado por `creado_en`), con las 12 filas y sus acumulados.
  * **Hipótesis (suposición no probada solo por SQL):** falta de validación o bloqueo de stock ante compras rápidas/concurrencia (los pedidos `1115`, `1116` y `1117` ocurren el `2026-11-14` en una ventana de 3 segundos, entre las `23:15:02` y las `23:15:05`) o descuento de stock sin tope mínimo en 0.
  * **Falta confirmar con el equipo:** si el descuento de stock ocurre al crear o al pagar el pedido, cómo se comportan los pedidos `1115`, `1116` y `1117` frente al cliente y si ocurre en otros productos con alta rotación.
  * **Prioridad sugerida:** alta. En Qase Free el defecto no tiene campo de prioridad, así que va en este texto.