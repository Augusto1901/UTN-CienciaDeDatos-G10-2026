# 03 — Decisiones Técnicas y su Justificación (el "por qué")

> Formato: **Decisión → Por qué → Dónde está en el repo**. Las causas físicas van marcadas **(DOMINIO)**; lo verificable, **(REPO)**.

---

## 1. Predecir el método de descubrimiento a partir de variables físicas
**Por qué tiene sentido el problema:** cada técnica de detección tiene un **sesgo de selección físico**: detecta más fácil cierto tipo de planeta. Si el sesgo existe, las features físicas contienen la "huella" del método y el problema es aprendible. **(DOMINIO)** El EDA lo confirma con tests estadísticos (ver decisión 13). **(REPO** — `03_eda.ipynb`**)**

## 2. Filtrar `default_flag == 1`
**Por qué:** el archivo tiene varias filas por planeta (una por publicación). `default_flag` marca la entrada canónica. Sin filtrar, el mismo planeta aparece repetido con valores algo distintos → **pseudoduplicados** que sesgan el modelo y rompen la unidad de análisis. Tras el filtro: 5757 filas = 5757 planetas, 0 duplicados. **(REPO** — `presentacion.ipynb` celda 4; `justificacion_etl.md` celda 2**)**

## 3. Reagrupar el target en 4 clases
**Por qué:** el target crudo tiene 11 métodos, pero 8 de ellos suman 143 planetas (algunos con 1‑3 ejemplos). **No se puede entrenar ni evaluar un clasificador con clases de 1‑20 muestras**: el modelo no aprende patrones válidos y las métricas de esas clases no son confiables. Se conservan los 3 métodos masivos (con física bien diferenciada) y el resto va a `Other(s)`. **(REPO** — `justificacion_etl.md` celda 3; `presentacion.ipynb` celda 12**)**

## 4. Eliminar columnas por *data leakage* (decisión central del ETL)
**Qué es leakage:** que el modelo "vea" información que delata la respuesta y que no estaría disponible al predecir un planeta nuevo. **(DOMINIO)** Se eliminan 6 grupos **(REPO** — `02_etl_parte2.ipynb` celda 6; `justificacion_etl.md` celda 4**)**:

- **`disc_year`, `disc_facility` (leakage directo):** el observatorio y el año están **causalmente atados** al método. Kepler hacía casi solo Transit; ciertos observatorios solo Radial Velocity. Darle el telescopio al modelo es **darle la respuesta**. **(REPO/DOMINIO)**
- **`pl_bmassprov`, `st_metratio` (proxy del método):** indican **con qué técnica** se midió la masa/metalicidad → revela el método. `Msini`, por ejemplo, es típico de velocidad radial. **(REPO/DOMINIO)**
- **`ttv_flag`:** las variaciones de tiempo de tránsito son una **consecuencia** del método Transit, no una causa física del planeta. **(REPO)**
- **IDs (`pl_name`, `hostname`):** identificadores únicos; el modelo memorizaría planetas en vez de aprender física. **(REPO)**
- **Metadatos/referencias/fechas:** son del proceso de publicación, no del planeta. **(REPO)**
- **`*err1/*err2/*lim`:** el **error de medición** de un telescopio está correlacionado con el método que usa ese telescopio → leakage indirecto. **(REPO/DOMINIO)**
- **`pl_radj`, `pl_bmassj`:** radio/masa en unidades de Júpiter, **idénticos** a `pl_rade`/`pl_bmasse` salvo escala → redundancia que solo infla dimensionalidad. **(REPO)**

## 5. Indicadores `_missing` (la ausencia como señal)
**Por qué:** en datos astronómicos, **que falte una medición no es aleatorio**. Si falta `pl_rade`, suele ser porque el método no permite medir el radio (Radial Velocity mide masa, no tamaño). El **patrón de ausencia es en sí mismo predictivo del método**. Se captura en columnas binarias antes de imputar para no perder esa señal. **(REPO** — `justificacion_etl.md` celda 6**)**
> Este punto reaparece en los resultados: `pl_bmasse_missing` y `pl_rade_missing` están **entre los predictores más importantes del modelo** (ver [04](04-resultados-y-conclusiones.md) y matiz en decisión 14). **(REPO‑reproducido)**

## 6. Eliminar columnas con >75 % de nulos
**Por qué el umbral 75 %:** con <25 % de datos reales, cualquier imputación **inventa** más del 75 % de la columna; el feature representaría al imputador, no a la realidad. Se eliminan `pl_insol` (87,3 %) y `st_spectype` (79,9 %). **(REPO** — `etl_de_cero_corregido.ipynb` celda 6**)**
> Detalle fino: **`pl_eqt` (74,8 %) NO se elimina** porque queda apenas por debajo del umbral. Es un caso límite defendible en ambos sentidos. **(REPO)**

## 7. Consistencia física (negativos→NaN, excentricidad en [0,1])
**Por qué:** los datasets científicos a veces codifican faltantes como `-1`/`-999`. Un radio de −5 R⊕ es imposible; dejarlo rompería el `log1p` y envenenaría las distancias del KNN. La excentricidad es adimensional y acotada: 0 = órbita circular, →1 = muy elíptica (1 = parabólica, el objeto escapa); fuera de [0,1] es incoherente. **(REPO** — `justificacion_etl.md` celda 8**)** En estos datos, el chequeo encontró **0 negativos** (el dataset ya venía limpio en ese aspecto), pero el control queda como salvaguarda. **(REPO** — celda 7 output**)**

## 8. Transformación `log1p` sobre variables sesgadas
**Por qué:** período, semieje, radio, masa y distancia abarcan **varios órdenes de magnitud** (período de horas a miles de años; masa de fracciones de M⊕ a miles de M⊕). En escala lineal, un planeta de 1 año y otro de 1000 años parecen "a 999 de distancia" cuando lo que importa es la diferencia **relativa**. `log1p(x)=log(1+x)` comprime la cola larga **y** está definida en x=0 (a diferencia de `log`, que diverge). **(REPO** — `justificacion_etl.md` celda 9**)**
**Por qué antes del split:** es **determinista** (no aprende parámetros del dataset), así que no genera leakage aplicarla sobre todo el dataset. **(REPO)**

## 9. Split 80/20 **estratificado** y en el lugar correcto del pipeline
**Por qué estratificado:** con el desbalance (Transit 75 %), un split al azar podría dejar el test con casi ningún Microlensing. `stratify=y` garantiza la misma proporción de clases en train y test. **(REPO** — `etl_de_cero_corregido.ipynb` celda 9**)**
**Por qué 80/20:** estándar para este tamaño; 20 % deja muestras suficientes por clase para evaluar, 80 % deja datos suficientes para que SMOTEENN trabaje. **(REPO/DOMINIO)**

## 10. Winsorización p1–p99 (recortar, no borrar)
**Por qué:** recorta el 1 % más extremo de cada cola **sin perder filas** (el valor extremo se reemplaza por el del percentil 99). Esto evita que SMOTE luego genere sintéticos en zonas extremas y aisladas creando "picos" artificiales. **Se fitea solo en train** (si se calcularan los percentiles con todo el dataset, el test influiría en los límites → leakage). **(REPO** — `justificacion_etl.md` celda 11**)**

## 11. Imputación **KNN (k=5)** en vez de media/mediana
**Por qué KNN:** un planeta con radio desconocido probablemente se parece a otros planetas con estrella y órbita similares. KNN promedia los 5 vecinos más parecidos en el espacio de features → respeta la **estructura local**. La media global trataría igual a un Júpiter y a una Tierra. **(REPO** — `justificacion_etl.md` celda 12**)**
**Por qué excluir `_missing` del imputer:** son binarias y sin NaN; meterlas en el cálculo de distancias sesgaría hacia vecinos que también tienen el dato ausente (círculo vicioso). **(REPO)**
**Por qué k=5:** valor estándar; k muy chico (1‑2) hace la imputación ruidosa, k muy grande (>20) la sobre‑suaviza. **(REPO/DOMINIO)**
> Diferencia con el ETL "para EDA", que usa **mediana global**. Las dos son válidas; KNN es más fino pero más costoso. Es la diferencia principal entre ambos pipelines. **(REPO)**

## 12. Balanceo **SMOTEENN** (solo en train)
**Por qué balancear:** con Transit dominando, un modelo "perezoso" que prediga siempre Transit acertaría ~75 % pero sería inútil para las minorías. **(DOMINIO)**
**Por qué SMOTEENN y no duplicar (`sample(replace=True)`):** duplicar copia filas exactas → picos artificiales y memorización. **SMOTE** interpola entre vecinos reales (mejor). **ENN** (Edited Nearest Neighbours) **limpia después**: borra cualquier muestra —real o sintética— que sus 3 vecinos clasificarían mal, eliminando los sintéticos que cayeron en zonas de otra clase. **(REPO** — `justificacion_etl.md` celda 13**)**
**Por qué solo en train:** el test debe reflejar la **distribución real** (con el desbalance natural). Balancear el test daría métricas de un mundo irreal. **(REPO)**
> Efecto medible: el train pasa de 4605 a **11 591 filas**, y tras ENN **Transit queda como la clase más pequeña** (2338) porque es la más densa y la que más recorta ENN. **(REPO** — celda 12**)**

## 13. Validación estadística de los sesgos físicos (el corazón del "por qué")
Las tres hipótesis del EDA validan, con datos reales (no imputados), los sesgos de cada método **(REPO** — `03_eda.ipynb`**)**:

- **H1 — Tránsito favorece radio grande.** Mediana de `pl_rade` (R⊕) entre los planetas con radio **medido**: Other 13,0 · Radial Velocity 2,773 · Transit 2,39. Kruskal‑Wallis **H=38,66, p=4,03e‑09**. **(REPO** — celda 13**)**
  - *Causa (DOMINIO):* Transit detecta por la caída de brillo ∝ (Rp/R⋆)². Un planeta grande tapa más luz → señal más fuerte, detección más fácil y más frecuente.
- **H2 — Velocidad Radial favorece masa grande.** Mediana de `pl_bmasse` (M⊕) entre planetas con masa medida: **Other 2542,6 · Radial Velocity 346,4 · Microlensing 180,6 · Transit 88,9**. Kruskal masa **H=278,93, p=3,6e‑60**; período **H=1358,33, p=3,2e‑294**. **(REPO** — celda 16**)**
  - *Causa (DOMINIO):* RV mide el bamboleo Doppler de la estrella; su amplitud crece con la masa del planeta → favorece planetas masivos.
- **H3 — Microlente favorece gran distancia.** Mediana de `sy_dist` (pc): **Microlensing 6015 · Transit 521,8 · Other 138,5 · Radial Velocity 44,4**. Kruskal **H=2357,95, p≈0**. **(REPO** — celda 20**)**
  - *Causa (DOMINIO):* el microlente no necesita ver luz del planeta ni de la estrella anfitriona; aprovecha la curvatura gravitatoria de la luz de una estrella de fondo, por eso alcanza sistemas a kiloparsecs.
- **H4 — Relación masa‑radio.** Spearman `pl_rade`‑`pl_bmasse`: **rho=0,822, p≈1,5e‑320** (relación monótona fuerte, ley empírica conocida). **(REPO** — celda 23**)**

## 14. ⚠️ Matiz honesto: qué aprendió realmente el modelo
Al reproducir las feature importances (no guardadas), los **predictores top no son `pl_rade`/`pl_bmasse` directos**, sino **`sy_dist`, las magnitudes de brillo (`sy_kmag/vmag/gaiamag`), `pl_orbsmax`** y los **indicadores `_missing`** (`pl_bmasse_missing`, `pl_rade_missing`, `pl_orbper_missing`). **(REPO‑reproducido)**

**Por qué (DOMINIO + REPO):**
1. `pl_rade`/`pl_bmasse` tienen muchos valores **imputados** (53 % de masa era nula), lo que diluye su poder directo — pero **su ausencia** sí discrimina: Transit rara vez mide masa (→`pl_bmasse_missing`=1) y RV rara vez mide radio (→`pl_rade_missing`=1).
2. `sy_dist` separa nítidamente Microlensing (lejano) del resto (H3).
3. Las magnitudes/brillo correlacionan con el método porque RV y Transit prefieren **estrellas brillantes y cercanas**.

**Punto defendible (y posible crítica de la profesora):** parte de la exactitud viene de **qué medición está disponible** (los `_missing`), que es un rasgo del **proceso instrumental**, no una propiedad física pura del planeta. El grupo lo justifica como "ausencia informativa" legítima del dominio (celda 6 de la justificación), pero un evaluador estricto podría llamarlo **leakage suave**. La respuesta honesta: es una decisión consciente y documentada; si se quisiera medir solo la física pura, se podrían quitar los `_missing` y volver a evaluar. **(REPO/DOMINIO/DUDA)**

## 15. Elección de algoritmos: Random Forest + Gradient Boosting
**Por qué modelos de árboles:** **no asumen ninguna distribución** en los features (no les molesta el skew ni los picos), manejan no‑linealidades e interacciones, y dan **feature importance** interpretable — clave para "leer" qué física aprendió el modelo. **(REPO** — comentarios celda 14; `justificacion_etl.md` nota final**)**
- **Random Forest:** muchos árboles en paralelo (bagging), robusto a varianza. `max_depth=12` y `min_samples_split=5` limitan el overfitting; `class_weight='balanced'` compensa desbalances residuales tras SMOTEENN. **(REPO)**
- **Gradient Boosting:** árboles secuenciales que corrigen el error del anterior. `max_depth=5` (menor que RF) porque boosting tiende más al overfitting; `learning_rate=0.1` controla cuánto corrige cada árbol. **(REPO)**
**Por qué dos modelos:** para **comparar** y elegir el mejor por F1, y contrastar sus feature importances. **(REPO** — celda 16‑17**)**
