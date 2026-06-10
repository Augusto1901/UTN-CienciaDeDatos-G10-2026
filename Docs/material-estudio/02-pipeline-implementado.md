# 02 — Pipeline Implementado (paso a paso, con referencias)

> **IMPORTANTE — hay DOS ETL en el repo.** No son lo mismo y conviene tenerlo clarísimo para no contradecirse en la exposición:
>
> | | **ETL "para EDA"** | **ETL "para modelado"** |
> |---|---|---|
> | Notebooks | `02_etl.ipynb`, `02_etl_parte2.ipynb`, `presentacion.ipynb` | `etl_de_cero_corregido.ipynb` (replicado en `03_modelado_final.ipynb` celdas 0‑13) |
> | Imputación | **Mediana global** | **KNN (k=5)** |
> | Outliers | Se **detectan, no se eliminan** | **Winsorización p1–p99** |
> | Balanceo | **No** (se deja para después) | **SMOTEENN** |
> | Split train/test | **No** (solo limpia) | **Sí, 80/20 estratificado**, antes de imputar/balancear |
> | Variables log | Crea columnas nuevas `*_log` (conserva ambas) | Aplica `log1p` **in‑place** sobre 5 columnas |
> | Etiqueta minoritaria | `Other` | `Others` |
> | Salida | CSV `Data/processed/NASA_Exoplanet_Clean_DiscoveryMethod.csv` (42 col) | Objetos en memoria `X_train_bal, y_train_bal, X_test, y_test` |
>
> El **EDA** (`03_eda.ipynb`) corre sobre el **CSV del ETL "para EDA"** (mediana). El **modelado** corre sobre el ETL "para modelado" (KNN+SMOTEENN). **(REPO** — `03_eda.ipynb` celda 3 lee el CSV; `03_modelado_final.ipynb` celda 0 carga el raw y re‑hace el ETL**)**

---

## A. ETL para modelado — `etl_de_cero_corregido.ipynb` = `03_modelado_final.ipynb` celdas 0‑13

Este es el pipeline que alimenta el modelo final. La justificación celda‑por‑celda está en `Notebooks/justificacion_etl.md` **(REPO)**.

| Paso | Celda | Qué hace | Resultado verificable |
|---|---|---|---|
| 1. Carga raw | 0 | Lee `NASA_Exoplanet_Archive_Data.csv`, define `TARGET="discoverymethod"` | 36 418 filas **(REPO)** |
| 2. Filtrado | 1 | `default_flag==1` + `dropna(target)` | **5757 filas, 5757 planetas únicos (REPO)** |
| 3. Reagrupar target | 2 | Mantiene `Transit/Radial Velocity/Microlensing`, resto → `Others` | Transit 4304, RV 1080, Microlensing 230, Others 143 **(REPO)** |
| 4. Anti‑leakage | 3 | Elimina IDs, metadatos, `disc_year`/`disc_facility`, `rastr`/`decstr`, `default_flag`/`ttv_flag`/`pl_bmassprov`/`st_metratio`, `pl_radj`/`pl_bmassj`, y todo `*err1/*err2/*lim` | **69 columnas eliminadas → shape (5757, 22) (REPO)** |
| 5. Análisis de nulos | 4 | % de nulos por columna, líneas de referencia 50 % y 75 % | `pl_insol` 87,3 %, `st_spectype` 79,9 %, `pl_eqt` 74,8 %… **(REPO)** |
| 6. `_missing` | 5 | Crea 13 indicadores binarios `{col}_missing` antes de imputar | 13 columnas nuevas **(REPO)** |
| 7. Drop >75 % nulos | 6 | Elimina columnas con >75 % nulos (protege target y `_missing`) | Elimina **`pl_insol`, `st_spectype`** → shape (5757, 33) **(REPO)** |
| 8. Consistencia física | 7 | Negativos→NaN en columnas positivas; `pl_orbeccen` recortada a [0,1] | 0 negativos encontrados **(REPO)** |
| 9. `log1p` | 8 | Sobre `pl_orbper, pl_orbsmax, pl_rade, pl_bmasse, sy_dist` (in‑place). Guarda `df_before_processing` para graficar | Transformación determinista **(REPO)** |
| **10. SPLIT** | 9 | **`train_test_split(test_size=0.2, random_state=42, stratify=y)`** | **Train 4605 / Test 1152 (REPO)** |
| 11. Winsorización | 10 | Recorta a p1–p99; **límites aprendidos solo en train**, aplicados a ambos | 19 columnas, 0 filas perdidas **(REPO)** |
| 12. Imputación KNN | 11 | `KNNImputer(n_neighbors=5)`: `fit_transform` en train, `transform` en test. Excluye `_missing`. Categóricas→`'Unknown'` | Sin nulos, sin leakage **(REPO)** |
| 13. SMOTEENN | 12 | `LabelEncoder` + `SMOTEENN(random_state=42).fit_resample` **solo en train** | **Train balanceado 11 591 filas (REPO)** |
| 14. Visualización | 13 | Histogramas ANTES (rojo) vs DESPUÉS (verde) + tablas de frecuencia | Control de calidad **(REPO)** |

**Distribución del train antes del balanceo** (celda 9): Transit 3443, RV 864, Microlensing 184, Others 114. **(REPO)**

**Distribución del train DESPUÉS de SMOTEENN** (celda 12): Microlensing 3388, Others 3067, Radial Velocity 2798, **Transit 2338**. **(REPO)**
→ Detalle clave: tras SMOTEENN **Transit queda como la clase más chica**. ENN borra muchas muestras (reales y sintéticas) de las zonas densas, y Transit, al ser la mayoritaria y compacta, es la que más pierde. **(REPO** + interpretación**)**

**El test NUNCA se balancea** — mantiene la distribución real (Transit 861, RV 216, Microlensing 46, Others 29). **(REPO** — celda 12**)**

### El orden importa (regla anti‑leakage)
Todo lo **determinista o justificado sobre el dataset completo** va **antes** del split (limpieza de columnas, validación física, `log1p`). Todo lo que **aprende parámetros de los datos** (winsorización, KNN, SMOTEENN) va **después** y se fitea **solo en train**. Si se invirtiera, el test "contaminaría" el entrenamiento y las métricas saldrían infladas. **(REPO** — `justificacion_etl.md` celda 10**)**

---

## B. ETL para EDA — `presentacion.ipynb` / `02_etl_parte2.ipynb`

Produce el CSV limpio que usa el EDA. Mismos pasos 1‑4 que arriba, pero a partir del paso de nulos cambia:

| Paso | Celda (`presentacion.ipynb`) | Qué hace | Resultado |
|---|---|---|---|
| Imputación numérica | 10 | **Mediana global** por columna (no usa el target → sin leakage) | Tabla de medianas: `pl_rade`→2,40, `pl_bmasse`→187,52, `sy_dist`→399,03, `pl_eqt`→903, etc. **(REPO)** |
| Reagrupar minoritarias | 12 | `MIN_CLASS_SIZE=100`: clases <100 → `Other` | 4 clases (Other=143) **(REPO)** |
| Outliers | 12 | Los **detecta con IQR pero NO los elimina** (en astronomía muchos extremos son reales) | Tabla top‑20 outliers **(REPO)** |
| `*_log` | 12 | Crea 9 columnas `*_log` con `log1p` (conserva originales) | `pl_bmasse_log`, `pl_rade_log`, … **(REPO)** |
| Exportación | 14 | `df.to_csv(.../NASA_Exoplanet_Clean_DiscoveryMethod.csv)` | **Shape final (5757, 42), 0 nulos, 0 duplicados (REPO)** |

---

## C. EDA — `03_eda.ipynb` (sobre el CSV mediana, 42 columnas)

Objetivo: validar estadísticamente las hipótesis de sesgo físico. **(REPO** — celda 0**)**

| Análisis | Celda | Técnica | Resultado guardado |
|---|---|---|---|
| Distribución target | 5 | conteo | Transit 74,76 %… **(REPO)** |
| Medianas por clase | 7 | `groupby().median()` | ⚠️ incluye valores imputados (ver nota abajo) **(REPO)** |
| Correlación general | 9 | **Spearman** (relaciones monótonas, robusto a escala) | heatmap **(REPO)** |
| Correlación con target | 11 | Spearman vs target codificado | `pl_orbsmax` −0,552, `sy_kmag` 0,539, `sy_gaiamag` 0,533, `sy_vmag` 0,520, `pl_orbper` −0,426… **(REPO)** |
| **H1 radio** | 13 | Boxplot + **Kruskal‑Wallis** | **H=38,66, p=4,03e‑09** → se rechaza H0 **(REPO)** |
| **H2 masa/período** | 16 | Boxplots + Kruskal‑Wallis | masa **H=278,93, p=3,6e‑60**; período **H=1358,33, p=3,2e‑294 (REPO)** |
| **H3 distancia** | 20 | Boxplot + Kruskal‑Wallis | **H=2357,95, p≈0 (REPO)** |
| **H4 masa‑radio** | 23 | Scatter + **Spearman** | **rho=0,822, p≈1,5e‑320 (REPO)** |

> **Trampa de la celda 7 (leer con cuidado).** La tabla de "medianas por clase" mezcla valores reales con los imputados por mediana global, por eso varias filas se ven casi iguales entre clases (p. ej. `pl_rade`≈2,40 en todas porque 2,40 fue el valor imputado). Las comparaciones **reales** entre métodos están en las celdas 13/16/20, donde el código **filtra los imputados** con `df[col+"_missing"]==0`. Esas son las que hay que citar. **(REPO** — `03_eda.ipynb` celda 6 lo advierte explícitamente**)**

### Por qué Kruskal‑Wallis y no ANOVA
**(DOMINIO/REPO)** Las variables astronómicas son fuertemente asimétricas y de muchas escalas; no cumplen normalidad ni homocedasticidad. Kruskal‑Wallis es **no paramétrico** (compara medianas/rangos, no asume normalidad), por eso es la elección correcta. La hipótesis nula H0 es "todas las clases tienen la misma distribución"; al ser p ≪ 0,05, se rechaza → la variable **sí** discrimina entre métodos. **(REPO** — `03_eda.ipynb` celda 1 y 13**)**

---

## D. Modelado — `03_modelado_final.ipynb` celdas 14‑19

> **Última modificación del repo (commit `0c7e746`, 09‑jun‑2026, "Agregue regresion Logistica para comparar"):** se agregó una **Regresión Logística multinomial como baseline lineal** (celda 15) y la comparación pasó de 2 a **3 modelos**, con **F1 Macro como métrica principal**. **(REPO** — `git log`; celdas 15, 17 y 19**)**

| Paso | Celda | Detalle |
|---|---|---|
| Definición y entrenamiento RF/GB | 14 | **RandomForest**(`n_estimators=150, max_depth=12, min_samples_split=5, class_weight='balanced', random_state=42`) y **GradientBoosting**(`n_estimators=150, learning_rate=0.1, max_depth=5, random_state=42`); `.fit(X_train_bal, y_train_bal)` (11 591 filas) y `classification_report` sobre el test real (1152 filas) **(REPO — outputs guardados)** |
| **Baseline: Regresión Logística** | 15 | `LogisticRegression(C=1.0, max_iter=3000, solver='lbfgs', class_weight='balanced', random_state=42)` sobre features **escaladas con `StandardScaler`** (fit solo en train) **(REPO — código; output NO guardado)** |
| Matriz de confusión RF + feature importance | 16 | `confusion_matrix` (RF) + `feature_importances_` top‑15 **(REPO — código; output NO guardado)** |
| Comparación de 3 modelos + overfitting | 17 | Accuracy train/test, **F1 Macro**, F1 ponderado, precision/recall ponderados; elige el "mejor modelo" por `argmax(F1 Macro Test)` **(REPO — código; output NO guardado)** |
| Feature importance RF vs GB + 3 matrices de confusión + reportes (4 decimales) | 18 | **(REPO — código; output NO guardado)** |
| Conclusiones | 19 | Texto generado con f‑strings: mejor modelo por F1 Macro, diferencias train‑test de los 3, rol del baseline lineal **(REPO — código; output NO guardado)** |

> **Las celdas 15‑19 no tienen output guardado en el `.ipynb`** (el commit que agregó la Regresión Logística se guardó sin ejecutar esas celdas). Sus números (comparación, feature importance, matrices) fueron **reproducidos** ejecutando el mismo código determinista; ver [04-resultados-y-conclusiones.md](04-resultados-y-conclusiones.md). Los reportes de la celda 14 (RF y GB) **sí** están guardados y son la fuente primaria. **(REPO)**

### Por qué un baseline lineal (celda 19, texto del repo)
"Si los árboles superan a la Regresión Logística, esto sugiere que existen **relaciones no lineales** importantes entre las variables y el método de descubrimiento." **(REPO** — celda 19**)** La reproducción lo confirma: F1 Macro ≈ 0,93 (árboles) vs **0,85** (logística). **(REPO‑reproducido)**

## E. Tecnologías

`pandas`, `numpy`, `matplotlib`, `seaborn`, `scipy.stats`, `scikit-learn` (`KNNImputer`, `train_test_split`, `LabelEncoder`, `StandardScaler`, `RandomForestClassifier`, `GradientBoostingClassifier`, `LogisticRegression`, métricas) e `imbalanced-learn` (`SMOTEENN`). **(REPO** — imports de cada notebook + `requirements.txt`**)**
> ⚠️ `imbalanced-learn` (imblearn) **no figura en `requirements.txt`** aunque es necesario para correr el ETL/modelado. Es un hueco a corregir (ver [05-preguntas-y-respuestas.md](05-preguntas-y-respuestas.md) y lista final). **(REPO** — `requirements.txt`**)**
