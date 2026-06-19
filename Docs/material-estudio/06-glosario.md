# 06 — Glosario

> Definiciones simples. **(DOMINIO)** = conocimiento astrofísico/estadístico general; **(REPO)** = el término aparece y se usa en el repo.

## Términos del dominio (astronomía)

- **Exoplaneta** — Planeta que orbita una estrella distinta del Sol. **(DOMINIO)**
- **Estrella anfitriona (host)** — La estrella alrededor de la cual orbita el exoplaneta; sus propiedades son las variables `st_*`. **(DOMINIO/REPO)**
- **Tránsito (Transit)** — Método: detecta el planeta por la **caída de brillo** cuando pasa frente a su estrella. Señal ∝ (radio planeta / radio estrella)². Favorece planetas **grandes** y de período corto. **(DOMINIO)**
- **Velocidad Radial (Radial Velocity)** — Método: detecta el **bamboleo Doppler** de la estrella por el tirón gravitatorio del planeta. Amplitud ∝ masa del planeta. Favorece planetas **masivos**. **(DOMINIO)**
- **Microlente gravitacional (Microlensing)** — Método: detecta por la **amplificación de la luz** de una estrella de fondo cuando el sistema actúa como lente gravitacional. Funciona a **gran distancia** (kpc). **(DOMINIO)**
- **Imaging (imagen directa)** — Fotografiar directamente el planeta separándolo de la luz de su estrella; funciona con gigantes jóvenes y brillantes lejos de su estrella. Va en `Other(s)`. **(DOMINIO/REPO)**
- **Período orbital (`pl_orbper`)** — Tiempo que tarda el planeta en dar una vuelta a su estrella, en **días**. **(REPO)**
- **Semieje mayor (`pl_orbsmax`)** — "Radio promedio" de la órbita, en **UA** (1 UA = distancia Tierra‑Sol). **(REPO)**
- **Excentricidad (`pl_orbeccen`)** — Cuán elíptica es la órbita: 0 = círculo perfecto; →1 = muy alargada; 1 = parabólica (escapa). Adimensional, en [0,1]. **(REPO/DOMINIO)**
- **Radio planetario (`pl_rade`)** — Tamaño del planeta en **radios terrestres (R⊕)**. **(REPO)**
- **Masa planetaria (`pl_bmasse`)** — Masa en **masas terrestres (M⊕)**. "b" = *best mass*. **(REPO)**
- **Msini (`pl_bmassprov`)** — Cuando la masa se mide por velocidad radial, solo se conoce **M·sin(i)** (masa × seno de la inclinación), una cota inferior. Que la procedencia sea `Msini` delata el método RV → por eso se elimina (leakage). **(DOMINIO/REPO)**
- **Temperatura de equilibrio (`pl_eqt`)** — Temperatura teórica del planeta por la radiación que recibe, en **Kelvin**. **(REPO)**
- **Insolación (`pl_insol`)** — Radiación estelar que recibe el planeta, en flujos terrestres. Eliminada por 87 % de nulos. **(REPO)**
- **Temperatura efectiva (`st_teff`)** — Temperatura de la superficie de la estrella, en **Kelvin**. **(REPO)**
- **Metalicidad (`st_met`, [Fe/H])** — Abundancia de elementos pesados de la estrella respecto al Sol. [Fe/H]=0 → como el Sol; >0 → más metálica. **(DOMINIO/REPO)**
- **Gravedad superficial (`st_logg`)** — log de la gravedad en la superficie estelar, en log(cm/s²). **(REPO)**
- **Tipo espectral (`st_spectype`)** — Clasificación O‑B‑A‑F‑G‑K‑M (de más caliente a más fría). Eliminado por 80 % de nulos. **(DOMINIO/REPO)**
- **Distancia al sistema (`sy_dist`)** — Distancia Tierra‑sistema, en **parsecs** (1 pc ≈ 3,26 años luz). **(REPO/DOMINIO)**
- **Magnitud (`sy_vmag`, `sy_kmag`, `sy_gaiamag`)** — Brillo aparente de la estrella. Escala **inversa y logarítmica**: menor magnitud = más brillante. V = visual, K = infrarrojo, Gaia = la del satélite Gaia. **(REPO/DOMINIO)**
- **Ascensión recta / Declinación (`ra`/`dec`)** — Coordenadas de la estrella en el cielo, en grados (como longitud/latitud celestes). **(REPO/DOMINIO)**
- **`default_flag`** — Marca (1) la entrada **canónica/preferida** de cada planeta entre sus múltiples publicaciones. **(REPO)**
- **`disc_year` / `disc_facility`** — Año y observatorio de descubrimiento. **Eliminados (leakage):** atados al método. **(REPO)**

## Términos técnicos (ciencia de datos)

- **ETL** — Extract, Transform, Load: extraer, limpiar/transformar y dejar el dataset listo. **(REPO)**
- **EDA** — Exploratory Data Analysis: explorar y entender los datos con estadística y gráficos. **(REPO)**
- **Target / variable objetivo** — Lo que se predice: aquí `discoverymethod`. **(REPO)**
- **Clasificación multiclase** — Predecir una etiqueta entre **más de dos** clases (aquí 4). **(REPO)**
- **Desbalance de clases** — Cuando una clase domina (Transit 75 %); obliga a estratificar, balancear y usar F1. **(REPO/DOMINIO)**
- **Data leakage (fuga de información)** — Que el modelo use info que delata la respuesta y no estaría disponible al predecir casos nuevos → métricas infladas y falsas. **(REPO/DOMINIO)**
- **Indicador `_missing`** — Columna binaria que marca si un valor estaba ausente; convierte el patrón de nulos en feature. **(REPO)**
- **Imputación** — Rellenar valores faltantes. Aquí: **KNN** (modelado) o **mediana global** (EDA). **(REPO)**
- **KNN Imputer** — Rellena un nulo con el promedio de los **k vecinos** más parecidos (k=5). **(REPO)**
- **`log1p`** — Transformación log(1+x); comprime variables muy sesgadas y admite x=0. **(REPO)**
- **Winsorización** — Recortar valores extremos a un percentil (p1/p99) **sin borrar filas**. **(REPO)**
- **Outlier** — Valor atípico, muy alejado del resto. En astronomía muchos son **reales**, no errores. **(REPO/DOMINIO)**
- **Split train/test** — Partir los datos en entrenamiento (80 %) y evaluación (20 %). **(REPO)**
- **Estratificación (`stratify`)** — Que train y test mantengan la misma proporción de clases. **(REPO)**
- **SMOTE** — Genera ejemplos sintéticos de la clase minoritaria interpolando entre vecinos reales. **(DOMINIO/REPO)**
- **ENN (Edited Nearest Neighbours)** — Limpia muestras mal clasificadas por sus vecinos; en SMOTEENN va después de SMOTE. **(DOMINIO/REPO)**
- **SMOTEENN** — Combinación SMOTE (genera) + ENN (limpia). Solo en train. **(REPO)**
- **Random Forest** — Conjunto de muchos árboles de decisión en **paralelo** (bagging); promedia sus votos. **(REPO/DOMINIO)**
- **Gradient Boosting** — Árboles **secuenciales**, cada uno corrige el error del anterior. **(REPO/DOMINIO)**
- **Regresión Logística (multinomial)** — Modelo **lineal** de clasificación: estima la probabilidad de cada clase con una combinación lineal de los features. Aquí se usa como **baseline**. **(REPO/DOMINIO)**
- **Baseline** — Modelo simple de referencia; si los modelos complejos no lo superan, la complejidad no se justifica. Aquí el baseline lineal pierde por mucho (F1 Macro 0,85 vs 0,93) → hay relaciones no lineales. **(REPO/DOMINIO)**
- **StandardScaler / escalado** — Resta la media y divide por el desvío de cada feature (z‑score). Necesario para modelos lineales (sensibles a la escala); irrelevante para árboles (deciden por umbrales). Se fitea solo en train. **(REPO/DOMINIO)**
- **`class_weight='balanced'`** — Penaliza más equivocarse en clases chicas; compensa desbalance residual. **(REPO)**
- **Feature importance** — Cuánto aporta cada variable a las predicciones del árbol/bosque. **(REPO)**
- **Matriz de confusión** — Tabla real vs predicho; la diagonal son los aciertos. **(REPO)**
- **Precision** — De lo que predije como clase X, cuánto era realmente X. **(REPO/DOMINIO)**
- **Recall** — De los X reales, cuántos detecté. **(REPO/DOMINIO)**
- **F1‑score** — Media armónica de precision y recall (equilibra ambas). **macro** = promedia clases por igual; **weighted** = pondera por soporte. **(REPO/DOMINIO)**
- **Accuracy** — Proporción total de aciertos. Engañosa con desbalance. **(REPO/DOMINIO)**
- **Overfitting** — Memorizar el train y generalizar mal; se detecta si train ≫ test. Aquí diferencia ≈ 0,02 → no hay. **(REPO/DOMINIO)**
- **`random_state=42`** — Semilla que fija la aleatoriedad para que los resultados sean **reproducibles**. **(REPO)**
- **Kruskal‑Wallis** — Test **no paramétrico** que compara si ≥3 grupos tienen la misma distribución; no asume normalidad. p<0,05 → difieren. **(REPO/DOMINIO)**
- **Correlación de Spearman** — Mide relación **monótona** (no necesariamente lineal) entre dos variables; robusta a escalas y outliers. **(REPO/DOMINIO)**
- **p‑value** — Probabilidad de ver estos datos si la hipótesis nula (no hay diferencia) fuera cierta; chico (<0,05) → se rechaza la nula. **(DOMINIO)**
