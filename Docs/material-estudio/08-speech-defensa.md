# 08 — Speech de defensa oral (máx. 7 minutos)

> **Cómo usarlo.** El speech está dividido en **3 bloques de ~2–2,5 minutos** (≈910 palabras totales ≈ 6,5–7 min a ritmo normal). La profesora elige 3 al azar → **cada integrante tiene que poder decir cualquiera de los 3 bloques**. Los cortes están pensados para que cada bloque cierre una idea completa.
>
> Todos los números del speech están verificados contra el repo (ver [04-resultados-y-conclusiones.md](04-resultados-y-conclusiones.md) y [07-verificacion-supuestos-y-huecos.md](07-verificacion-supuestos-y-huecos.md)).
>
> ⚠️ **Antes de la defensa:** re-ejecutar y guardar `03_modelado_final.ipynb` completo. Las celdas 15–19 (Regresión Logística, comparación, conclusiones) hoy están **sin output**: si la profesora abre el notebook, no va a ver esos resultados.

---

## BLOQUE 1 — Problema, dataset e hipótesis físicas (~2 min)

Buenos días. Nuestro proyecto trabaja con **exoplanetas**: planetas que orbitan estrellas distintas del Sol. Usamos el **NASA Exoplanet Archive**, el catálogo público que mantiene el IPAC de Caltech para la NASA, descargado en noviembre de 2024: **36.418 filas y 91 columnas**.

La pregunta que nos hicimos fue: **¿se puede predecir con qué método fue descubierto un exoplaneta mirando solamente sus propiedades físicas, orbitales y estelares?** Es decir, sin usar ningún metadato del proceso: ni el año, ni el observatorio, ni las referencias de publicación.

¿Por qué tiene sentido esa pregunta? Porque ningún método "ve" al planeta directamente: cada técnica infiere su existencia midiendo un efecto distinto, y eso le impone un **sesgo físico**:

- **Tránsito** mide la caída de brillo cuando el planeta pasa frente a su estrella. La señal crece con el cuadrado del radio relativo → favorece **planetas grandes**.
- **Velocidad Radial** mide el bamboleo Doppler de la estrella por el tirón gravitatorio del planeta → la amplitud crece con la masa → favorece **planetas masivos**.
- **Microlente gravitacional** no necesita ver luz del sistema: detecta la amplificación de la luz de una estrella de fondo → funciona a kiloparsecs → favorece **sistemas lejanos**.

Formulamos esas tres hipótesis y las validamos en el EDA con **Kruskal-Wallis**, un test no paramétrico, porque los datos astronómicos no cumplen normalidad. **Las tres se validan con p muchísimo menor a 0,05.** Los números más gráficos: la mediana de masa en Velocidad Radial es **346 masas terrestres contra 89 en Tránsito**; y la mediana de distancia en Microlente es **6.015 parsecs contra 44 en Velocidad Radial**. El sesgo existe y es medible.

*(pase al siguiente: "Con las hipótesis validadas, lo que sigue es preparar los datos sin hacer trampa…")*

---

## BLOQUE 2 — ETL y decisiones anti-leakage (~2,5 min)

El archivo trae **varias filas por planeta**, una por cada publicación científica. Filtramos `default_flag = 1`, la entrada canónica, y quedamos con **5.757 planetas únicos**. El target tenía 11 métodos, pero 8 de ellos suman apenas 143 planetas; los agrupamos en "Others" y quedó un problema de **4 clases**: Tránsito 75 %, Velocidad Radial 19 %, Microlente 4 % y Otros 2 %. **Fuerte desbalance**, y eso condiciona todo lo que sigue.

La decisión central del ETL fue **mitigar el data leakage**. Eliminamos **69 columnas**: el año y el observatorio de descubrimiento —porque Kepler hacía casi solo Tránsito: darle el telescopio al modelo es darle la respuesta—, la procedencia de la masa tipo *Msini* que delata Velocidad Radial, las columnas de error instrumental, los identificadores y las unidades redundantes.

Segunda decisión: **la ausencia como señal**. En astronomía, que falte un dato no es aleatorio: Tránsito rara vez mide la masa, Velocidad Radial rara vez mide el radio. Antes de imputar creamos **13 indicadores binarios `_missing`** que conservan ese patrón.

Tercera: **el orden del pipeline**. Todo lo determinista —limpieza, consistencia física, logaritmo `log1p` sobre las 5 variables que abarcan órdenes de magnitud— va **antes** del split. Todo lo que **aprende parámetros de los datos** va **después** y se ajusta **solo con train**: winsorización a percentiles 1–99, imputación KNN con 5 vecinos y el balanceo SMOTEENN. El split es **80/20 estratificado**: 4.605 planetas para entrenar y 1.152 para evaluar.

Sobre **SMOTEENN**: SMOTE genera muestras sintéticas interpolando entre vecinos reales de las clases minoritarias, y ENN limpia después las muestras mal clasificadas por sus vecinos. El train queda balanceado en **11.591 filas**. El **test no se toca nunca**: conserva la distribución real, porque las métricas tienen que reflejar el mundo como es.

*(pase al siguiente: "Con los datos listos, entrenamos y comparamos tres modelos…")*

---

## BLOQUE 3 — Modelado, resultados y conclusiones (~2,5 min)

Entrenamos **tres modelos**: Random Forest, Gradient Boosting y una **Regresión Logística multinomial como baseline lineal**, con escalado StandardScaler ajustado solo en train. ¿Por qué un baseline? Porque si los árboles le ganan, queda demostrado que hay **relaciones no lineales** entre las variables y el método.

La métrica principal es **F1 Macro**, no accuracy: con 75 % de Tránsito, un modelo que prediga siempre "Tránsito" tiene 75 % de accuracy y es inútil. F1 Macro promedia las cuatro clases con el mismo peso.

Resultados sobre el test real: los dos modelos de árboles llegan a **98 % de accuracy y F1 Macro 0,93, prácticamente empatados**. El baseline queda en **0,85** — eso confirma la no linealidad. Y **no hay overfitting**: la diferencia train-test es de unos 2 puntos, muy por debajo del umbral de 15 que fijamos.

Por clase: **Microlente se clasifica perfecto, 46 de 46** — la distancia la separa sola. Tránsito, casi perfecto. La clase débil es "Otros", con F1 0,80: es una mezcla heterogénea de 8 métodos con solo 29 casos de test.

¿Qué aprendió el modelo? Las variables más importantes son **la distancia al sistema, las magnitudes de brillo, el semieje mayor y los indicadores `_missing` de masa y radio**. Es decir: el modelo recuperó exactamente los sesgos físicos que habíamos validado en el EDA —distancia para Microlente, brillo para los métodos que necesitan estrellas cercanas— y además usa la disponibilidad de cada medición como huella del método.

**Conclusión:** el método de descubrimiento de un exoplaneta **se puede predecir con alta exactitud a partir de su física**, porque cada técnica de detección impone un sesgo real y medible, y el modelo lo recupera. Como mejoras proponemos búsqueda de hiperparámetros, validación cruzada estratificada y SHAP para interpretabilidad. Muchas gracias.

---

## Tarjeta de números (memorizar — son los únicos 12 que hay que saber)

| Dato | Valor |
|---|---|
| Dataset crudo | 36.418 × 91 |
| Planetas únicos (`default_flag=1`) | 5.757 |
| Clases del target | Transit 4.304 · RV 1.080 · Micro 230 · Others 143 |
| Columnas eliminadas por leakage | 69 → quedan 22 |
| Indicadores `_missing` | 13 |
| Split | 80/20 estratificado: 4.605 / 1.152 |
| Train tras SMOTEENN | 11.591 |
| Accuracy test (árboles) | ≈ 0,98 |
| F1 Macro | RF 0,93 ≈ GB 0,93 ≫ Logística 0,85 |
| Overfitting (dif. train-test) | ≤ 0,024 (umbral 0,15) |
| Medianas clave | masa: RV 346 vs Transit 89 M⊕ · distancia: Micro 6.015 vs RV 44 pc |
| Tests EDA | Kruskal-Wallis p ≪ 0,05 en H1/H2/H3 · Spearman masa-radio 0,82 |

## Si la profesora interrumpe con preguntas (pivotes rápidos)

- **"¿Por qué un planeta masivo se descubre por Velocidad Radial?"** → Porque RV mide el bamboleo Doppler de la estrella y su amplitud **crece con la masa**; Tránsito en cambio depende del **radio**, no de la masa. Evidencia: mediana 346 vs 89 M⊕, Kruskal p=3,6e-60. *(P2 del archivo 05)*
- **"¿Qué es data leakage acá?"** → Darle al modelo info que delata la respuesta: el observatorio determina casi el método (Kepler→Transit). Por eso volaron 69 columnas. *(P8–P9)*
- **"¿Por qué no balancearon el test?"** → Porque el test debe reflejar la distribución real; balancearlo daría métricas de un mundo artificial. *(P17)*
- **"¿Cuál es el mejor modelo?"** → RF y GB están virtualmente empatados en F1 Macro (≈0,93); lo importante es que ambos superan por mucho al baseline lineal (0,85). *(P25c)*
- **"¿El modelo aprende física o aprende qué dato falta?"** → Ambas. Los `_missing` son una decisión consciente ("ausencia informativa"); si se quisiera física pura, se quitan y se re-evalúa. *(P27 — respuesta honesta)*

> Todo lo demás está en [05-preguntas-y-respuestas.md](05-preguntas-y-respuestas.md) (31 preguntas con respuesta).
