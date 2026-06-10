# 05 — Preguntas y Respuestas tipo examen

> 31 preguntas. Las respuestas citan la evidencia. Las preguntas "por qué" se responden con la **causa real**, no con una descripción.

---

### Bloque A — Dominio y física de la detección

**P1. ¿Qué predice el modelo y con qué insumos?**
El **método de descubrimiento** (`discoverymethod`) de un exoplaneta, usando **solo** sus propiedades físicas, orbitales y estelares — nunca metadatos del proceso. Es clasificación **multiclase** (4 clases). **(REPO** — `README.md`; `02_etl_parte2.ipynb` celda 0**)**

**P2. ¿Por qué para planetas de MASA grande el método suele ser Velocidad Radial (y no Tránsito)?** *(la pregunta "estrella")*
Porque **Velocidad Radial mide el bamboleo gravitatorio** que el planeta induce en su estrella (corrimiento Doppler), y **la amplitud de ese bamboleo crece con la masa del planeta**: un planeta más masivo tira más fuerte → señal más grande → detección más fácil. Tránsito, en cambio, **no depende de la masa sino del radio** (mide la luz bloqueada ∝ (Rp/R⋆)²). Por eso los planetas masivos pueblan la clase RV. **Evidencia en el repo:** mediana de masa por método (planetas con masa medida) = Radial Velocity **346,4 M⊕** vs Transit **88,9 M⊕**, y Kruskal‑Wallis **H=278,93, p=3,6e‑60**. **(REPO** — `03_eda.ipynb` celda 16; causa **(DOMINIO))**

**P3. ¿Por qué para planetas de RADIO grande el método suele ser Tránsito?**
Porque Tránsito detecta por la **caída de brillo** cuando el planeta cruza frente a la estrella, y esa caída ∝ (radio planeta / radio estrella)². **Un planeta grande tapa más luz** → señal más fuerte y más fácil de confirmar. **Evidencia:** mediana de `pl_rade` medido = Transit 2,39 vs (Other 13,0, que son sobre todo gigantes por imagen directa); Kruskal **H=38,66, p=4,03e‑09**. **(REPO** — `03_eda.ipynb` celda 13; causa **(DOMINIO))**

**P4. ¿Por qué Microlensing detecta sistemas tan lejanos?**
Porque **no necesita ver la luz del planeta ni de la estrella anfitriona**: aprovecha que el sistema, al pasar frente a una estrella de fondo, **curva su luz como una lente gravitacional** y la amplifica. Eso funciona a **kiloparsecs**. **Evidencia:** mediana `sy_dist` = Microlensing **6015 pc** vs Radial Velocity 44,4 pc; Kruskal **H=2357,95, p≈0**. **(REPO** — `03_eda.ipynb` celda 20; causa **(DOMINIO))**

**P5. ¿Por qué el brillo de la estrella (magnitudes) ayuda a predecir el método?**
Porque RV y Tránsito necesitan **buena relación señal‑ruido**, que se logra con estrellas **brillantes y cercanas**; Microlensing no. Así, el brillo correlaciona con qué método se pudo aplicar. Se ve en que `sy_kmag/vmag/gaiamag` están entre las features más importantes del modelo. **(REPO‑reproducido** — feature importance**; (DOMINIO)** la causa**)**

**P6. ¿Qué es la metalicidad estelar y por qué se la mira?**
`st_met` = [Fe/H], la abundancia de elementos pesados de la estrella. **(DOMINIO)** Históricamente correlaciona con la presencia de planetas gigantes. En este dataset, sin embargo, **resultó poco discriminante** del método (Spearman con target ≈ −0,028, casi nula). **(REPO** — `03_eda.ipynb` celdas 11 y 25**)**

---

### Bloque B — Datos y ETL

**P7. ¿Cuántas filas/columnas tiene el dataset y por qué se reduce a 5757 filas?**
Crudo: **36 418 × 91**. Tras `default_flag==1` (entrada canónica por planeta) + quitar target nulo: **5757 planetas únicos**. Había varias filas por planeta (una por publicación). **(REPO** — `01_data_exploration.ipynb` celda 3; `presentacion.ipynb` celda 4**)**

**P8. ¿Qué es el data leakage y qué columnas se eliminaron por eso?**
Leakage = darle al modelo información que delata la respuesta y no estaría al predecir un planeta nuevo. Se eliminaron `disc_year`/`disc_facility` (observatorio↔método), `pl_bmassprov`/`st_metratio` (técnica de medición), `ttv_flag` (consecuencia del tránsito), IDs, metadatos, `*err/*lim` (el error revela el instrumento) y unidades redundantes (`pl_radj`, `pl_bmassj`). **69 columnas → quedan 22.** **(REPO** — `02_etl_parte2.ipynb` celda 6; `justificacion_etl.md` celda 4**)**

**P9. ¿Por qué `disc_facility` sería el peor leakage de todos?**
Porque el observatorio está **casi determinado** por el método: Kepler hacía Transit, ciertos espectrógrafos solo RV. Si el modelo aprende "telescopio = Kepler → Transit", no aprende física, **memoriza el atajo**. Sería como darle la respuesta. **(REPO/DOMINIO** — `presentacion.ipynb` celda 5**)**

**P10. ¿Qué son las columnas `_missing` y por qué se crean ANTES de imputar?**
Son indicadores binarios (1 = el valor original estaba ausente). En astronomía, **la ausencia no es aleatoria**: que falte la masa suele significar que el método no la mide. Se crean antes de imputar para no perder esa señal cuando los NaN se rellenan. **(REPO** — `etl_de_cero_corregido.ipynb` celda 5; `justificacion_etl.md` celda 6**)**

**P11. ¿Por qué se eliminan columnas con >75 % de nulos? ¿Cuáles?**
Porque con <25 % de datos reales, imputar **inventa** la mayoría de la columna; el feature representaría al imputador, no a la realidad. Se eliminan `pl_insol` (87,3 %) y `st_spectype` (79,9 %). `pl_eqt` (74,8 %) **se salva** por estar bajo el umbral. **(REPO** — `etl_de_cero_corregido.ipynb` celda 6**)**

**P12. ¿Por qué KNN para imputar y no la media?**
Porque un planeta con un dato faltante se parece a otros con estrella y órbita similares; KNN promedia los **5 vecinos más parecidos** (estructura local), mientras la media global trataría igual a un Júpiter y a una Tierra. Se usa `KNNImputer(n_neighbors=5)`, fiteado **solo en train**. **(REPO** — `etl_de_cero_corregido.ipynb` celda 11; `justificacion_etl.md` celda 12**)**
> Ojo: el ETL "para EDA" usa **mediana global**, no KNN. Son dos pipelines distintos (ver [02](02-pipeline-implementado.md)).

**P13. ¿Por qué `log1p` y no `log` a secas?**
Porque `log1p(x)=log(1+x)` está definido en x=0 (hay planetas con semieje o excentricidad ≈0), mientras `log(0)` diverge. Se aplica a 5 variables que abarcan varios órdenes de magnitud (período, semieje, radio, masa, distancia) para comprimir la cola larga. **(REPO** — `etl_de_cero_corregido.ipynb` celda 8; `justificacion_etl.md` celda 9**)**

**P14. ¿Por qué la transformación log va ANTES del split pero la imputación DESPUÉS?**
Porque `log1p` es **determinista** (no aprende nada del dataset) → no hay leakage aplicarla a todo. La imputación KNN, la winsorización y SMOTEENN **aprenden parámetros de los datos**; si se fitearan con el dataset completo, el test contaminaría el train. Por eso van después del split y se fitean solo en train. **(REPO** — `justificacion_etl.md` celdas 9 y 10**)**

**P15. ¿Por qué el split es estratificado?**
Por el desbalance (Transit 75 %): un split al azar podría dejar el test casi sin Microlensing. `stratify=y` mantiene la misma proporción de clases en train y test. Train 4605 / Test 1152. **(REPO** — `etl_de_cero_corregido.ipynb` celda 9**)**

---

### Bloque C — Balanceo y outliers

**P16. ¿Qué es SMOTEENN y por qué se eligió?**
SMOTE genera muestras sintéticas **interpolando** entre vecinos reales de las clases minoritarias (mejor que duplicar filas). ENN (Edited Nearest Neighbours) **limpia después**: borra muestras —reales o sintéticas— mal clasificadas por sus 3 vecinos. Resultado: clases balanceadas y fronteras más limpias. **(REPO** — `etl_de_cero_corregido.ipynb` celda 12; `justificacion_etl.md` celda 13**)**

**P17. ¿Por qué NO se balancea el test?**
Porque el test debe reflejar la **distribución real** del problema (Transit domina). Balancearlo daría métricas de un mundo artificial y el modelo rendiría peor de lo esperado en producción. **(REPO** — `justificacion_etl.md` celda 13**)**

**P18. Tras SMOTEENN, ¿por qué Transit queda como la clase MÁS chica del train balanceado?**
Porque ENN elimina muestras en zonas densas y bien pobladas; Transit es la mayoritaria y compacta, así que es la que más recortes recibe. Queda Transit 2338 vs Microlensing 3388. **(REPO** — `etl_de_cero_corregido.ipynb` celda 12**)**

**P19. ¿Por qué se winsoriza en vez de borrar outliers? Y en astronomía, ¿los outliers son errores?**
Se winsoriza (recorta a p1‑p99) para **no perder filas**: el valor extremo se reemplaza por el del percentil, evitando que SMOTE amplifique zonas aisladas. En astronomía **muchos extremos son reales** (gigantes, órbitas raras), por eso el ETL "para EDA" los **detecta pero no los elimina**. **(REPO** — `etl_de_cero_corregido.ipynb` celda 10; `presentacion.ipynb` celda 12**)**

**P20. Sobre los "picos" en los histogramas (p. ej. `pl_rade`): ¿son un error del ETL?**
No. Ya existían en el dato crudo (gráfico rojo "ANTES") y **sobrevivieron a winsorización y a ENN**, que justo borran concentraciones espurias. El pico en radio grande refleja el **sesgo real de Transit** hacia planetas grandes. Borrarlo destruiría la señal más discriminante. **(REPO** — `justificacion_etl.md`, nota final**)**

---

### Bloque D — Modelado y métricas

**P21. ¿Qué algoritmos se usaron y con qué hiperparámetros?**
**Random Forest** (`n_estimators=150, max_depth=12, min_samples_split=5, class_weight='balanced', random_state=42`), **Gradient Boosting** (`n_estimators=150, learning_rate=0.1, max_depth=5, random_state=42`) y, como **baseline lineal**, una **Regresión Logística multinomial** (`C=1.0, solver='lbfgs', max_iter=3000, class_weight='balanced', random_state=42`) sobre features escaladas. **(REPO** — `03_modelado_final.ipynb` celdas 14 y 15**)**

**P22. ¿Por qué se agregó una Regresión Logística si los árboles son mejores?**
Justamente para **probar** que son mejores: la logística es el **piso lineal de referencia**. El notebook lo dice: "si los árboles superan a la Regresión Logística, esto sugiere que existen **relaciones no lineales** importantes entre las variables y el método". Y lo superan: F1 Macro ≈ 0,93 (árboles) vs **0,85** (logística). Además los árboles no asumen ninguna distribución de los features (no les afecta el skew) y dan feature importance interpretable. **(REPO** — celdas 15 y 19; **(REPO‑reproducido)** las métricas**)**

**P23. ¿Por qué se escala con `StandardScaler` SOLO para la Regresión Logística y no para los árboles?**
Porque la logística optimiza una combinación **lineal** de los features → es sensible a la escala (una variable en miles aplasta a una en décimas y el solver converge mal). Los árboles deciden con **umbrales** ("¿x > 2,3?"), invariantes a la escala. El scaler se fitea **solo en train** para no filtrar información del test. **(REPO** — comentarios de la celda 15**; (DOMINIO)** la explicación**)**

**P24. ¿Qué resultados dio el modelo?**
Los árboles ≈ **0,98 de accuracy** y **F1 Macro ≈ 0,93** en test; la logística 0,94 de accuracy y F1 Macro 0,85. Mejor clase: Microlensing y Transit (F1 ≈ 0,99); peor: Other(s) (F1 0,80 en árboles, 0,50 en logística; solo 29 casos). **(REPO** — celda 14 guardada; logística **(REPO‑reproducido)**, su output no quedó guardado**)**

**P25. ¿Por qué la métrica principal es F1 Macro y no accuracy o F1 ponderado?**
Porque con 75 % de Transit, un modelo que prediga siempre "Transit" tendría ~75 % de accuracy y sería inútil; el F1 ponderado también queda dominado por Transit. El **F1 Macro promedia las 4 clases con el mismo peso**, así que exige rendir también en Microlensing y Others. El notebook lo explicita: "Se prioriza F1 macro porque el dataset original está desbalanceado". **(REPO** — `03_modelado_final.ipynb` celdas 17 y 19; `03_eda.ipynb` celda 27 ya lo recomendaba**)**

**P25b. ¿Hay overfitting?**
No relevante. Diferencia accuracy train‑test: RF **0,0217**, GB **0,0174**, logística **0,0235** (criterio del notebook: overfitting si >0,15). **(REPO‑reproducido** — `03_modelado_final.ipynb` celda 17**)**

**P25c. ¿Cuál es el mejor modelo?**
**RF y GB están virtualmente empatados** (F1 Macro 0,9317 vs 0,9313 en la reproducción; en los reportes guardados a 2 decimales GB muestra 0,94 vs 0,93). La respuesta defendible: ambos árboles son equivalentes y muy superiores al baseline (0,85); cuál sale primero depende de la corrida/versión. El notebook elige por `argmax(F1 Macro Test)`. **(REPO‑reproducido / DUDA** — el output de la corrida del grupo no quedó guardado**)**

**P26. ¿Cuáles son las variables más importantes y qué te dice eso?**
`sy_dist`, las magnitudes de brillo (`sy_kmag/vmag/gaiamag`), `pl_orbsmax` y los indicadores `pl_bmasse_missing`/`pl_rade_missing`. Dice que el modelo recupera los sesgos físicos (distancia→Microlensing, brillo→RV/Transit) **y** que la disponibilidad de cada medición es una huella del método. **(REPO‑reproducido** — celdas 16‑18; matiz en [03](03-decisiones-y-porques.md) decisión 14**)**

**P27. Pregunta capciosa: ¿el modelo aprende física o aprende qué dato falta?**
**Ambas cosas.** `sy_dist` y `pl_orbsmax` son físicas; pero `pl_bmasse_missing`/`pl_rade_missing` están en el top, y eso es señal del **proceso de medición** (Transit no mide masa, RV no mide radio). El grupo lo justifica como "ausencia informativa" del dominio; un evaluador estricto podría verlo como **leakage suave**. Respuesta honesta: es una decisión consciente y documentada; quitando los `_missing` se podría medir la física pura. **(REPO/DOMINIO/DUDA)**

**P28. Si tuvieras que mejorar el trabajo, ¿qué harías?**
Lo que sugiere el propio notebook (celda 19): GridSearchCV/RandomizedSearchCV para hiperparámetros, **validación cruzada estratificada** y **SHAP** para interpretabilidad. Yo agregaría: re‑evaluar **sin** los `_missing` para aislar la física, reforzar la clase `Other(s)`, re‑ejecutar y guardar los outputs de las celdas 15‑19, y unificar los dos ETL en uno solo. **(REPO** — celda 19; el resto **(DOMINIO))**
