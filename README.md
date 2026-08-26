# Análisis de volatilidad de criptomonedas

Análisis del riesgo histórico de 5 criptomonedas (BTC, DOGE, NEO, UNI, ZEN) usando precios diarios OHLC. El objetivo es comparar cuál ofrece mejor relación riesgo/retorno y en qué momento histórico habría sido más conveniente comprar.

---

## Objetivo

Responder dos preguntas de negocio concretas:

1. **¿Cuál de estas cinco criptomonedas es la más eficiente en términos de riesgo asumido por unidad de retorno obtenida?**
2. **¿En qué mes del año ha sido históricamente más barato comprar cada una?**

El análisis no predice precios futuros. Trabaja únicamente con datos históricos y métricas estadísticas estándar usadas en finanzas.

---

## Hallazgos principales

- **UNI tiene el mejor ratio riesgo/retorno (0.097)**, seguida de DOGE (0.087) y BTC (0.071). ZEN es la menos eficiente con un ratio de 0.023, a pesar de tener el retorno diario promedio más alto (41.2%).
- **BTC es la criptomoneda menos volátil** con una desviación estándar de retornos de 21.2, frente a ZEN que alcanza 1800.4 — unas 85 veces más volátil.
- **El drawdown máximo de BTC supera los 9.1 millones de unidades**, el mayor en términos absolutos, reflejo de su precio medio histórico elevado (341,687). ZEN ocupa el segundo lugar con 2.0 millones.
- **Enero es el mejor mes histórico para comprar BTC y UNI**. DOGE tiene su mínimo histórico mensual en marzo, NEO en abril y ZEN en febrero.
- El dataset cubre **16,942 registros diarios** desde julio de 2010 (primer dato disponible de BTC) hasta 2025.

---

## Stack técnico

Python · pandas · numpy · matplotlib

---

## Estructura del proyecto

```
analisis-volatilidad-cripto/
├── datos/
│   ├── crudos/          <- CSVs originales, uno por criptomoneda (BTC, DOGE, NEO, UNI, ZEN)
│   └── procesados/      <- datasets limpios y métricas calculadas listos para consumir
├── notebooks/
│   ├── 01_limpieza_datos.ipynb               <- unifica los 5 CSVs en un solo dataset
│   ├── 02_analisis_metricas.ipynb            <- calcula volatilidad, drawdown y ratio riesgo/retorno
│   ├── 03_recomendaciones_estadisticas.ipynb <- ranking de criptos por eficiencia estadística
│   └── 04_recomendacion_mensual.ipynb        <- identifica el mes óptimo de compra por cripto
├── outputs/
│   └── figures/         <- gráficos exportados (precio histórico, drawdown, retorno acumulado)
├── README.md
└── requirements.txt
```

---

## Metodología

### 1. Limpieza y unificación (`01_limpieza_datos.ipynb`)

Los 5 archivos CSV tienen formatos inconsistentes: distintos separadores, comillas, codificación BOM y separadores de miles con punto. El pipeline de limpieza aplica en orden:

- Detección automática de separador con `sep=None, engine='python'`
- Eliminación de BOM (`﻿`) en nombres de columna
- Estandarización de nombres a español (ticker → cripto_id, date → fecha, etc.)
- Conversión de fechas con `dayfirst=True` y eliminación de fechas inválidas
- Limpieza de precios: elimina separador de miles (`.`), sustituye coma decimal por punto, convierte a float
- Eliminación de duplicados por `(cripto_id, fecha)`

**Por qué este orden:** los tipos se corrigen antes de eliminar nulos para no perder filas por errores de formato que serían corregibles. Los duplicados se eliminan al final porque antes no se puede garantizar que las fechas sean comparables.

Resultado: `datos/procesados/precios_diarios.csv` — 16,942 filas, separador `;`, codificación UTF-8.

---

### 2. Métricas financieras (`02_analisis_metricas.ipynb`)

Sobre el dataset unificado se calculan cuatro métricas por criptomoneda:

**Retorno diario** — variación porcentual del precio de cierre respecto al día anterior:
```python
df["retorno_diario"] = df.groupby("cripto_id")["cierre"].pct_change()
```
Se agrupa por `cripto_id` antes de aplicar `pct_change()` para evitar que el primer día de una cripto compare con el último día de otra cripto diferente en el dataset concatenado.

**Volatilidad** — desviación estándar de los retornos diarios. Mide cuánto oscila el rendimiento día a día. Un valor alto indica mayor incertidumbre.

**Drawdown** — diferencia entre el máximo acumulado histórico y el precio actual. Responde: ¿cuánto ha caído desde su pico más alto?
```python
df["cierre_max"] = df.groupby("cripto_id")["cierre"].cummax()
df["drawdown"] = df["cierre_max"] - df["cierre"]
```

**Ratio riesgo/retorno** — retorno promedio dividido entre la volatilidad. Cuanto más alto, más retorno se obtiene por cada unidad de riesgo asumida. Es el equivalente simplificado del ratio de Sharpe sin tasa libre de riesgo.
```python
ratio = retorno.mean() / retorno.std()
```

---

### 3. Ranking estadístico (`03_recomendaciones_estadisticas.ipynb`)

Las métricas se ordenan por ratio riesgo/retorno descendente. La cripto en primer lugar (UNI) es la que históricamente ha generado más retorno relativo a la volatilidad que introduce. Se identifica además su precio mínimo histórico y la fecha en que ocurrió.

**Por qué ratio riesgo/retorno y no solo retorno:** una cripto puede tener retorno alto pero con oscilaciones tan extremas que hacen imposible saber en qué momento entrar. El ratio penaliza esa incertidumbre.

---

### 4. Estacionalidad mensual (`04_recomendacion_mensual.ipynb`)

Se agrupa por `(cripto_id, mes)` calculando el precio promedio de cierre. El mes con menor promedio histórico por cripto es el candidato para comprar con el menor precio esperado.

**Limitación importante:** este análisis asume que los patrones mensuales históricos se repetirán, lo cual no está garantizado. Los mercados de criptomonedas son especialmente sensibles a eventos externos (regulación, adopción institucional, ciclos de halvings en BTC) que pueden alterar estos patrones.

---

## Cómo ejecutarlo

```bash
# 1. Clonar el repositorio
git clone https://github.com/alerrandroFerreira/crypto-volatility-analysis.git
cd crypto-volatility-analysis

# 2. Crear entorno e instalar dependencias
python -m venv .venv
.venv\Scripts\activate        # Windows
source .venv/bin/activate     # macOS/Linux
pip install -r requirements.txt

# 3. Ejecutar los notebooks en orden
jupyter lab
```

Los notebooks están numerados y deben ejecutarse en orden: `01 → 02 → 03 → 04`. Cada uno depende del CSV generado por el anterior.

---

## Datos

Los CSVs originales en `datos/crudos/` provienen de fuentes públicas de precios históricos. Cada archivo contiene columnas OHLC (apertura, máximo, mínimo, cierre) con frecuencia diaria.

Los datos **no incluyen volumen de negociación** por lo que métricas de liquidez quedan fuera del alcance de este análisis.

---

## Lo que este proyecto no hace

- No predice precios futuros
- No incluye análisis de sentimiento ni datos on-chain
- No recomienda estrategias de inversión

Es un proyecto de análisis histórico con fines de aprendizaje y demostración de habilidades técnicas.
