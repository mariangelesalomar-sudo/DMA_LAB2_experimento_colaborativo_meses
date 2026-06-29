# Experimento Training Strategy — Problema #7

Predicción de baja de clientes Premium (clase **BAJA+2**) para la competencia
**Labo 1 Rosario 2026 — Analista Junior**.
Laboratorio de Implementación I · Maestría en Ciencia de Datos · Universidad Austral (Rosario, 2026).

Este repositorio contiene **todos los scripts** del experimento de *estrategia de
entrenamiento* (qué meses usar para entrenar el modelo) y las **instrucciones para
reproducirlos** partiendo del dataset de la competencia.

- **Grupo A:** Mariángeles Alomar · Judith Luna
- **Grupo B:** Juan I. Cacchione · Federico Spinelli

---

## ¿Qué se probó?

5 estrategias de entrenamiento, manteniendo **todo lo demás constante** (preprocesamiento,
feature engineering e hiperparámetros). Lo único que cambia entre experimentos es la celda
de *Training Strategy*.

| Experimento | Qué cambia respecto al baseline |
|---|---|
| **base (Exp0)** | Baseline: entrena con toda la historia disponible |
| **Exp1** | **Sin pandemia**: excluye los meses 202003–202006 |
| **Exp2** | **Ventana reciente**: entrena solo desde 202001 |
| **Exp4** | **Pesos temporales**: los meses recientes pesan más (peso lineal 0.2 → 1.0) |
| **Exp5** | **Pesos + pandemia penalizada**: pesos temporales y además peso 0.1 a los meses de pandemia |

---

## Dos familias de notebooks

Cada estrategia se corrió en dos notebooks (uno por grupo). **Solo difieren en qué mes
predicen**, lo que da dos validaciones independientes del mismo experimento.

| Familia | Notebooks | Predice (future) | Validación | Final train | Salida |
|---|---|---|---|---|---|
| **z910 (septiembre)** | `z910_WorkFlow_01_junior_*.ipynb` | 202109 | 202107 | …→202107 | sube a Kaggle (puntaje) |
| **z911 (julio)** | `z911_WorkFlow_01_junior_julio*.ipynb` | 202107 | 202105 | …→202105 | `ganancias.txt` (gan. suavizada) |

Archivos por experimento (en cada familia): `*_base`, `*_exp1`, `*_exp2`, `*_exp4`, `*_exp5`.

---

## Requisitos

- **Entorno:** máquina virtual de Google Cloud (modalidad Analista Jr) con runtime en **R**.
- **Dataset de la competencia:** `analistajr_competencia_2026.csv.gz`, ubicado en `/content/datasets/`.
- **Estructura de buckets:** `/content/buckets/b1/exp/` (cada corrida crea su carpeta `WF<experimento>`).
- **Librerías R** (se instalan solas si faltan): `data.table`, `lightgbm`, `R.utils`, `yaml`.

---

## Cómo reproducir un experimento

1. Abrir el notebook del experimento (ej. `z910_WorkFlow_01_junior_exp1.ipynb`).
2. Poner el runtime en **R** (*Runtime → Change runtime type → R*).
3. En la **celda de parámetros**, setear la semilla y el número de experimento:

   ```r
   PARAM$semilla_primigenia <- 600011   # una de las 5 semillas (ver abajo)
   PARAM$experimento        <- 9110     # número del experimento
   PARAM$dataset            <- "analistajr_competencia_2026.csv.gz"
   ```

4. Ejecutar **todas las celdas**, de arriba a abajo (~80–90 min por corrida).
5. Repetir con las **5 semillas**, cambiando `semilla_primigenia` y `experimento` cada vez.

### Semillas usadas
- **Grupo A (z910):** 999983, 524287, 999979, 700001, 817603
- **Grupo B (z911):** 600011, 314159, 898981, 750019, 450001

### Salidas de cada corrida (en `WF<experimento>/`)
- `prediccion.txt` — probabilidades por cliente (para ensembles).
- `ganancias.txt` — curva de ganancia (familia z911; columna `gan_suavizada` = ganancia a reportar).
- `modelo.txt`, `impo.txt` — modelo e importancia de variables.
- `PARAM.yml` — registro de los parámetros usados (incluye la semilla real).
- `kaggle/KA*.csv` — archivos para subir a Kaggle (familia z910).

---

## Qué hace el workflow (resumen)

1. **Catastrophe Analysis** — repara meses con variables rotas.
2. **Data Drifting** — ajuste de variables monetarias por inflación (deflación con IPC).
3. **Feature Engineering histórico** — lags 1 y 2 + deltas.
4. **Training Strategy** ← *lo único que cambia entre experimentos.*
5. **Hyperparameter Tuning** — grid search de LightGBM (optimiza AUC).
6. **Final training + Scoring**.
7. z910 → genera los CSV de Kaggle · z911 → genera `ganancias.txt`.

---

## La única celda que cambia entre experimentos

Todos los notebooks son idénticos salvo la celda de **Training Strategy**.
Por ejemplo, **Exp1 (sin pandemia)** queda así:

```r
PARAM$trainingstrategy$training <- c(
  201901, 201902, 201903, 201904, 201905, 201906,
  201907, 201908, 201909, 201910, 201911, 201912,
  202001, 202002,
  # SIN 202003, 202004, 202005, 202006  (meses de pandemia)
  202007, 202008, 202009, 202010, 202011, 202012,
  202101, 202102, 202103, 202104, 202105
)
```

(y `PARAM$trainingstrategy$final_train` igual, agregando los meses finales según la familia.)

Los experimentos con pesos (Exp4 y Exp5) además agregan, en la celda del `dtrain`/`dfinal_train`:

```r
meses_training <- PARAM$trainingstrategy$training
dataset[, peso := 0.2 + 0.8 * (foto_mes - min(meses_training)) /
                          (max(meses_training) - min(meses_training))]
# Solo Exp5: penalizar pandemia
dataset[foto_mes %in% c(202003, 202004, 202005, 202006), peso := 0.1]
# ... y weight = dataset[..., peso] dentro del lgb.Dataset
```

---

## Resultado y conclusión

La estrategia **Exp1 (sin pandemia)** fue la **mejor** y la **única que superó al baseline**,
confirmado por los dos grupos de forma independiente.
**Recomendación:** usar **Exp1** como configuración base de entrenamiento para la competencia.
