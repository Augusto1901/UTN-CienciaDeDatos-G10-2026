# 04 — Resultados y Conclusiones (métricas REALES)

> **Dos niveles de evidencia:**
> - **(REPO — guardado):** outputs presentes en el `.ipynb` → los **reportes de clasificación de la celda 14** de `03_modelado_final.ipynb` (Random Forest y Gradient Boosting). Son la fuente primaria.
> - **(REPO — reproducido):** outputs de las celdas 15‑19 que **no están guardados** (incluida toda la **Regresión Logística**, agregada en el último commit `0c7e746`); se reprodujeron con el mismo código y `random_state=42` (pandas 3.0.3 / sklearn 1.9.0 / imblearn 0.14.2). Coinciden con los guardados dentro de <1 punto.

---

## 1. Reporte de clasificación — Random Forest (test, 1152 planetas) — **(REPO guardado)**

| Clase | Precision | Recall | F1 | Soporte |
|---|---|---|---|---|
| Microlensing | 0,98 | 1,00 | 0,99 | 46 |
| Other(s) | 0,72 | 0,90 | 0,80 | 29 |
| Radial Velocity | 0,94 | 0,96 | 0,95 | 216 |
| Transit | 1,00 | 0,98 | 0,99 | 861 |
| **accuracy** | | | **0,98** | 1152 |
| macro avg | 0,91 | 0,96 | 0,93 | 1152 |
| weighted avg | 0,98 | 0,98 | 0,98 | 1152 |

## 2. Reporte de clasificación — Gradient Boosting — **(REPO guardado)**

| Clase | Precision | Recall | F1 | Soporte |
|---|---|---|---|---|
| Microlensing | 0,98 | 1,00 | 0,99 | 46 |
| Other(s) | 0,72 | 0,90 | 0,80 | 29 |
| Radial Velocity | 0,96 | 0,98 | 0,97 | 216 |
| Transit | 1,00 | 0,99 | 0,99 | 861 |
| **accuracy** | | | **0,98** | 1152 |
| macro avg | 0,92 | 0,97 | 0,94 | 1152 |
| weighted avg | 0,99 | 0,98 | 0,98 | 1152 |

**Lectura:** ambos modelos rinden casi igual (~0,98 accuracy). En los reportes guardados (2 decimales) **GB aparece levemente mejor** en macro F1 (0,94 vs 0,93), sobre todo en Radial Velocity (F1 0,97 vs 0,95). La clase **Other(s)** es la débil (F1 0,80) — esperable: es una mezcla heterogénea de métodos distintos con solo 29 casos de test. **(REPO)**

## 2b. Reporte de clasificación — Regresión Logística (baseline) — **(REPO reproducido)**

La celda 15 (agregada en el último commit) entrena una **Regresión Logística multinomial** sobre features escaladas con `StandardScaler` (fit solo en train). Su output **no está guardado**; valores reproducidos:

| Clase | Precision | Recall | F1 | Soporte |
|---|---|---|---|---|
| Microlensing | 1,00 | 1,00 | 1,00 | 46 |
| Other(s) | **0,35** | 0,86 | **0,50** | 29 |
| Radial Velocity | 0,92 | 0,91 | 0,92 | 216 |
| Transit | 1,00 | 0,95 | 0,97 | 861 |
| **accuracy** | | | **0,9436** | 1152 |
| macro avg | 0,82 | 0,93 | **0,8475** | 1152 |
| weighted avg | 0,97 | 0,94 | 0,95 | 1152 |

**Lectura:** el baseline lineal queda claramente por debajo de los árboles en F1 Macro (0,85 vs 0,93). Su gran debilidad es **Other(s)**: predice 71 planetas como "Others" pero solo 25 lo son (precision 0,35). Esto respalda la conclusión del propio notebook: hay **relaciones no lineales** entre las variables y el método que la logística no captura. **(REPO‑reproducido**; el argumento está escrito en la celda 19 **(REPO))**

## 3. Comparación de los 3 modelos y overfitting — **(REPO reproducido**, código en celda 17**)**

| Modelo | Accuracy Train | Accuracy Test | **F1 Macro Test** | F1 pond. Test | Dif. train‑test |
|---|---|---|---|---|---|
| Random Forest | 0,9983 | 0,9766 | **0,9317** | 0,9773 | 0,0217 |
| Gradient Boosting | 1,0000 | 0,9826 | **0,9313** | 0,9832 | 0,0174 |
| Regresión Logística | 0,9670 | 0,9436 | **0,8475** | 0,9522 | 0,0235 |

El criterio del notebook marca overfitting si la diferencia >0,15. Los tres están **muy por debajo** → **generalización aceptable**, no memorización. **(REPO** — criterio en `03_modelado_final.ipynb` celda 17; valores reproducidos**)**
> GB llega a 1,0000 en train pero 0,9826 en test: ajusta perfecto el train pero generaliza bien igual, gracias a `max_depth=5` y al test sin balancear.

### ⚠️ ¿Cuál es "el mejor modelo"? (matiz honesto)

El notebook elige por **`argmax(F1 Macro Test)`** (celda 17). En la **reproducción**, RF (0,9317) supera a GB (0,9313) por **0,0004** — un empate virtual que puede invertirse con otra versión de librería u otra corrida. En los **reportes guardados** (celda 14, 2 decimales), GB muestra macro F1 0,94 vs RF 0,93. **Respuesta defendible en el oral:** RF y GB son **estadísticamente equivalentes** (F1 Macro ≈ 0,93) y ambos superan con claridad al baseline lineal (0,85); cuál sale primero depende de la corrida. **(REPO‑reproducido / DUDA** — el output de la corrida del grupo no quedó guardado**)**

## 4. Matrices de confusión — **(REPO reproducido)**
Filas = clase real, columnas = predicha. Orden: [Microlensing, Other(s), Radial Velocity, Transit].

**Random Forest**
```
                 Micro  Other  RV   Transit
Microlensing  [   46     0     0      0 ]
Other(s)      [    0    26     2      1 ]
Radial Vel.   [    0     7   207      2 ]
Transit       [    0     4    11    846 ]
```
**Gradient Boosting**
```
                 Micro  Other  RV   Transit
Microlensing  [   46     0     0      0 ]
Other(s)      [    1    25     3      0 ]
Radial Vel.   [    0     4   211      1 ]
Transit       [    1     6     4    850 ]
```

**Regresión Logística (baseline)**
```
                 Micro  Other  RV   Transit
Microlensing  [   46     0     0      0 ]
Other(s)      [    0    25     4      0 ]
Radial Vel.   [    0    17   197      2 ]
Transit       [    0    29    13    819 ]
```

**Qué dicen los errores (REPO reproducido):**
- **Microlensing: 46/46 perfecto** en los tres modelos. Es la clase más fácil de aislar porque `sy_dist` la separa nítidamente (planetas lejanísimos). Confirma H3.
- La confusión principal es **Radial Velocity ↔ Other(s)** y algo de **Transit → RV**: planetas masivos detectados por RV o por métodos minoritarios comparten zona del espacio de features.
- **Transit** casi no se equivoca con los árboles (846/861 RF, 850/861 GB) — clase grande y bien separada.
- La **logística** comete los mismos tipos de error pero amplificados: manda 29 Transit y 17 RV a Other(s) — la frontera de esa clase mezclada no es separable linealmente.

## 5. Importancia de variables (Random Forest) — **(REPO reproducido)**

| # | Variable | Importancia |
|---|---|---|
| 1 | `sy_dist` (distancia) | 0,101 |
| 2 | `sy_kmag` (brillo banda K) | 0,097 |
| 3 | `sy_vmag` (brillo visual) | 0,097 |
| 4 | `pl_orbsmax` (semieje mayor) | 0,082 |
| 5 | `pl_bmasse_missing` | 0,078 |
| 6 | `pl_rade_missing` | 0,077 |
| 7 | `sy_gaiamag` (brillo Gaia) | 0,069 |
| 8 | `pl_orbper_missing` | 0,053 |
| 9 | `pl_orbeccen_missing` | 0,038 |
| 10 | `st_teff_missing` | 0,037 |

**Gradient Boosting** concentra aún más el peso: `sy_dist` 0,287 · `pl_bmasse_missing` 0,214 · `pl_orbsmax` 0,190 · `sy_kmag` 0,144 (estos 4 ≈ 84 %). **(REPO reproducido)**

**Interpretación (clave para defender el trabajo):**
- **`sy_dist` es el predictor #1** en ambos → consistente con H3 (Microlensing lejano). **(REPO/DOMINIO)**
- **Las magnitudes de brillo** pesan mucho porque el brillo está ligado a qué método se pudo usar (RV/Transit prefieren estrellas brillantes/cercanas). **(DOMINIO)**
- **Los indicadores `_missing` están en el top** → la **disponibilidad de cada medición** es una huella del método (Transit no mide masa; RV no mide radio). Esto valida la decisión del ETL de capturar la ausencia como señal, **pero** es en parte señal del proceso instrumental, no física pura (ver matiz en [03-decisiones-y-porques.md](03-decisiones-y-porques.md), decisión 14). **(REPO/DOMINIO)**

## 6. Conexión EDA ↔ modelo
El EDA **predijo** (con tests estadísticos) que radio, masa, período, distancia y brillo discriminan el método; el modelo lo **confirma** poniéndolos (o a sus `_missing`) en el top de importancia. La cadena hipótesis → validación estadística → feature importance está **cerrada y es coherente**. **(REPO** — `03_eda.ipynb` celda 27 lo anticipa como "próximo paso"; feature importance reproducida lo cumple**)**

## 7. Conclusiones finales del proyecto
1. **El método de descubrimiento es altamente predecible** a partir de propiedades físicas/observacionales: accuracy ≈ 0,98, F1 ponderado ≈ 0,98. **(REPO)**
2. **Los sesgos físicos de cada técnica son reales y medibles** (Kruskal‑Wallis p ≪ 0,05 en H1, H2, H3). **(REPO)**
3. **El ETL sin leakage y con SMOTEENN funcionó:** el modelo clasifica bien también las minorías (Microlensing F1 0,99; RV F1 0,95‑0,97), no solo la clase mayoritaria. **(REPO)**
4. **Sin overfitting** (diferencia train‑test ≤ 0,024 en los tres modelos). **(REPO reproducido)**
5. **La relación features→método es no lineal:** los árboles (F1 Macro ≈ 0,93) superan con claridad al baseline de Regresión Logística (0,85), tal como anticipa la celda 19 del notebook. **(REPO‑reproducido / REPO)**
6. **Limitación honesta:** parte del poder predictivo viene de los `_missing` (huella del proceso de medición), y `Other(s)` sigue siendo la clase floja en los tres modelos. **(REPO/DOMINIO)**

### Mejoras propuestas por el propio repo (celda 19, no ejecutada)
GridSearchCV / RandomizedSearchCV para seleccionar hiperparámetros, **validación cruzada estratificada** y **SHAP** para interpretar las predicciones. **(REPO** — `03_modelado_final.ipynb` celda 19**)**
> Nota: la versión anterior de esta celda mencionaba también stacking (RF+GB+SVM); el commit que agregó la Regresión Logística reescribió las conclusiones y esa mención ya no está. **(REPO** — `git log` `0c7e746` vs `995eafd`**)**
