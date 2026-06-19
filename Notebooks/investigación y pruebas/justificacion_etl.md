# Justificación de Decisiones ETL
### Proyecto: Predicción del Método de Descubrimiento de Exoplanetas (NASA)

---

## Celda 1 — Carga del Dataset e Imports

**Qué se hizo:** Se carga el CSV raw de la NASA sin ninguna modificación, y se definen las librerías necesarias para todo el pipeline. Se configura pandas para mostrar todas las columnas, hasta 100 filas, y con 4 decimales.

**Por qué:** Trabajar siempre sobre el dato crudo como punto de partida es una buena práctica fundamental. Nunca se debe modificar el archivo original; cualquier transformación se aplica en memoria sobre copias. El `TARGET = "discoverymethod"` se define como constante global para que cualquier celda pueda referenciarlo sin hardcodear el string.

**Efecto en el ETL:** Establece la base reproducible del pipeline. Si el CSV cambia (nueva versión del archivo de la NASA), solo se actualiza `RAW_PATH` y el resto del código no se toca.

---

## Celda 2 — Filtrado Inicial: Default Flag y Target

**Qué se hizo:** Se filtra el dataset para quedarse solo con las filas donde `default_flag == 1`, y se eliminan las filas donde el target (`discoverymethod`) es nulo.

**Por qué — `default_flag == 1`:** El archivo de la NASA contiene múltiples entradas por planeta, correspondientes a distintas publicaciones científicas que reportaron mediciones del mismo objeto. La columna `default_flag` indica cuál es la entrada "oficial" o más actualizada para cada planeta. Trabajar con todas las entradas significaría tener el mismo planeta repetido con valores ligeramente distintos, lo cual introduciría pseudoduplicados que sesgarían cualquier modelo.

**Por qué — `dropna(subset=[TARGET])`:** Un registro sin etiqueta no puede usarse para entrenar ni evaluar un clasificador supervisado. Mantenerlo generaría errores en las librerías de ML o, peor, introduciría ruido silencioso.

**Efecto en el ETL:** Se pasa de un dataset con múltiples entradas por planeta a uno con exactamente un registro por planeta, con target conocido. Esta es la unidad de análisis correcta para el problema.

---

## Celda 3 — Agrupación de Categorías del Target

**Qué se hizo:** Los métodos de descubrimiento poco frecuentes (imaging directo, astrometría, pulsaciones, etc.) se unifican bajo la etiqueta `'Others'`. Solo se mantienen como categorías propias: `Transit`, `Radial Velocity` y `Microlensing`.

**Por qué:** El dataset original tiene más de 10 métodos de descubrimiento, pero la mayoría tienen menos de 50 planetas. Entrenar un clasificador multiclase con clases de 10 o 20 muestras es inviable: el modelo no puede aprender patrones estadísticamente válidos de tan pocos ejemplos, y las métricas de evaluación para esas clases serían completamente no confiables.

**Por qué estos tres:** Son los tres métodos con representación masiva y con características físicas bien diferenciadas entre sí. Transit detecta planetas por bloqueo de luz (favorece planetas grandes y órbitas cortas). Radial Velocity detecta por oscilación gravitacional de la estrella (favorece planetas masivos). Microlensing detecta por deformación gravitacional de la luz de fondo (funciona a grandes distancias). Estas diferencias físicas hacen el problema de clasificación genuinamente abordable con features astrofísicos.

**Efecto en el ETL:** Se convierte un problema de clasificación con clases raras e inmanejables en uno de 4 clases balanceables con suficientes ejemplos para aprendizaje estadístico robusto.

---

## Celda 4 — Mitigación de Data Leakage

**Qué se hizo:** Se eliminan columnas organizadas en 6 grupos según el tipo de leakage que representan.

**Por qué cada grupo:**

- **`id_cols` (`pl_name`, `hostname`):** Son identificadores de texto únicos. No aportan información generalizable y cualquier encoding de ellos haría que el modelo memorice planetas específicos en lugar de aprender patrones físicos.

- **`metadata_cols` (referencias, fechas de publicación, tipo de solución):** Son metadatos del proceso científico de publicación, no propiedades físicas del planeta. El modelo no va a predecir planetas nuevos usando la fecha en que se publicó el paper.

- **`discovery_proc_cols` (`disc_year`, `disc_facility`):** Leakage directo. El año y el observatorio de descubrimiento están causalmente ligados al método: Kepler solo hacía Transit, ciertos observatorios solo hacen Radial Velocity. Incluirlos es equivalente a darle la respuesta al modelo.

- **`coord_text_cols` (`rastr`, `decstr`):** Son las coordenadas en formato de texto (string). Las coordenadas numéricas (`ra`, `dec`) ya están disponibles como columnas separadas. Los strings no agregan información y son formato incorrecto para ML.

- **`technical_cols` (`default_flag`, `ttv_flag`, `pl_bmassprov`, `st_metratio`):** `default_flag` ya fue usada para filtrar y su valor es constante (siempre 1). `ttv_flag` indica variaciones de tránsito — es una consecuencia del método Transit, no una causa. `pl_bmassprov` y `st_metratio` indican qué técnica se usó para calcular la masa/metalicidad, lo cual revela directamente el método de detección.

- **`redundant_unit_cols` (`pl_radj`, `pl_bmassj`):** Son el radio y masa del planeta expresados en unidades de Júpiter. Ya existen `pl_rade` y `pl_bmasse` con los mismos valores en unidades de Tierra. Tener ambas versiones es redundancia perfecta que solo infla la dimensionalidad sin aportar información nueva.

- **`error_lim_cols` (columnas que terminan en `err1`, `err2`, `lim`):** Son los errores de medición e indicadores de límites superiores/inferiores de cada medición. El error instrumental de un telescopio específico está correlacionado con el método que usa ese telescopio. Incluirlos filtraría información del método de forma indirecta.

**Efecto en el ETL:** Se elimina toda fuente de información que "traicione" la variable objetivo. Esto garantiza que el modelo aprenda las relaciones físicas reales entre propiedades planetarias y método de detección, no correlaciones espurias de proceso.

---

## Celda 5 — Análisis de Valores Nulos

**Qué se hizo:** Se calcula el porcentaje de nulos por columna y se visualiza con un gráfico de barras con líneas de referencia al 50% y 75%.

**Por qué:** Antes de tomar cualquier decisión de imputación o eliminación, es necesario conocer la magnitud y distribución del problema de datos faltantes. Una columna con 90% de nulos no puede imputarse de forma confiable. Una columna con 5% puede imputarse sin riesgos. El umbral del 75% (línea roja) es el criterio de decisión que se usará en la siguiente celda.

**Efecto en el ETL:** Este paso es diagnóstico puro. Informa las decisiones de las celdas 6 y 7. Sin este análisis, las decisiones de imputación se tomarían a ciegas.

---

## Celda 6 — Indicadores de Ausencia Informativa (`_missing`)

**Qué se hizo:** Para 13 columnas físicas clave, se crea una columna binaria `{col}_missing` que vale 1 si el valor original era nulo, 0 si estaba presente.

**Por qué:** En datos astronómicos, el hecho de que falte una medición no es aleatorio. Si falta `pl_rade` (radio del planeta), puede ser porque el método de detección no permite medir radios directamente — por ejemplo, Radial Velocity detecta la masa pero no el tamaño. El patrón de ausencia es en sí mismo una señal predictiva del método. Si se imputa el nulo sin registrar que estuvo ausente, se pierde esa señal.

**Por qué estas 13 columnas:** Son las variables físicas centrales del sistema planetario y estelar. Las columnas derivadas o de menor relevancia no requieren este tratamiento porque su ausencia no tiene interpretación física directa.

**Efecto en el ETL:** Convierte información implícita (la estructura de los nulos) en features explícitos que el modelo puede usar. Cada columna `_missing` es un feature binario adicional sin costo de imputación.

---

## Celda 7 — Eliminación de Columnas con Nulos Excesivos (>75%)

**Qué se hizo:** Se eliminan todas las columnas con más del 75% de valores nulos, excluyendo el target y las columnas `_missing` ya creadas.

**Por qué el umbral del 75%:** Con menos del 25% de datos reales, cualquier método de imputación (KNN, mediana, media) está básicamente inventando más del 75% de los valores de esa columna. El feature resultante no representaría la realidad sino el artefacto del imputador. Más que ayudar al modelo, lo confunde con señal fabricada.

**Por qué proteger las columnas `_missing`:** Estas columnas fueron creadas antes de la eliminación precisamente para conservar la señal de ausencia de columnas que sí van a ser eliminadas. Si se eliminasen también, se perdería esa información capturada.

**Efecto en el ETL:** Reduce la dimensionalidad eliminando features que solo añadirían ruido. El modelo entrena con features que tienen respaldo real de datos, no con columnas mayoritariamente imputadas.

---

## Celda 8 — Consistencia Física

**Qué se hizo:** Se convierten a `NaN` los valores negativos en columnas que físicamente solo pueden ser positivas. Adicionalmente, se valida que la excentricidad orbital (`pl_orbeccen`) esté en el rango `[0, 1]`.

**Por qué:** Los datasets científicos a veces contienen errores de codificación donde valores faltantes o inválidos fueron ingresados como `-1`, `-999`, o similares. Un radio de planeta de `-5` radios terrestres es físicamente imposible. Si estos valores se dejan, el logaritmo de la celda siguiente fallaría (`log(-5)` = NaN, pero de forma silenciosa con `log1p`), y peor aún, el KNN usaría esos valores negativos absurdos para calcular distancias, degradando la calidad de la imputación.

**Por qué `pl_orbeccen` tiene un tratamiento especial:** La excentricidad orbital es adimensional y acotada: 0 es órbita circular perfecta, 1 es parabolica (el objeto escapa). Valores fuera de `[0, 1]` son físicamente incoherentes y probablemente errores de entrada de datos.

**Efecto en el ETL:** Garantiza que los valores numéricos que pasan al pipeline son físicamente posibles. Esto protege la integridad de las transformaciones logarítmicas y la calidad de las distancias calculadas por KNN.

---

## Celda 9 — Transformación Logarítmica (`log1p`)

**Qué se hizo:** Se aplica `np.log1p(x)` a cinco columnas con distribuciones fuertemente sesgadas: `pl_orbper`, `pl_orbsmax`, `pl_rade`, `pl_bmasse`, `sy_dist`.

**Por qué estas columnas:** Son variables con rangos de varios órdenes de magnitud. El periodo orbital va de horas a miles de años. La masa planetaria va de fracciones de masa terrestre a miles de masas de Júpiter. Esta escala hace que un planeta con periodo de 1 año y otro con periodo de 1000 años estén "a 999 años de distancia" en escala lineal, cuando en realidad la diferencia relativa es lo que importa físicamente.

**Por qué `log1p` y no `log`:** `log1p(x) = log(1 + x)` está definido para `x = 0`, a diferencia de `log(x)` que diverge. Como puede haber planetas con excentricidad o semi-eje menor prácticamente nulo, `log1p` es la elección segura.

**Por qué antes del split:** La transformación logarítmica es determinista — no aprende ningún parámetro del dataset (a diferencia de una normalización que aprende media y desvío). No genera leakage aplicarla antes del split.

**Efecto en el ETL:** Comprime distribuciones de cola larga en distribuciones más simétricas. Esto beneficia directamente al KNN (distancias más representativas), a algoritmos sensibles a escala, y hace que SMOTEENN genere sintéticos en regiones más coherentes del espacio de features.

---

## Celda 10 — Train/Test Split

**Qué se hizo:** Se divide el dataset en 80% train y 20% test usando `stratify=y`.

**Por qué aquí en el pipeline:** Este es el punto de mayor importancia estructural del ETL. Todo lo que vino antes (limpieza de columnas, validación física, log transform) es determinista o basado en el dataset completo de forma justificada. Todo lo que viene después (imputación KNN, winsorización, balanceo) **aprende parámetros de los datos**. Si esos pasos se aplicaran antes del split, el test set "contaminaría" el entrenamiento, y las métricas de evaluación serían irrealmente optimistas.

**Por qué `stratify=y`:** El dataset está desbalanceado (Transit domina fuertemente). Sin estratificación, podría ocurrir por azar que el test set tenga proporcionalmente muchos más planetas de Transit o casi ningún planeta de Microlensing, haciendo la evaluación poco representativa. Con `stratify=y`, se garantiza que la proporción de cada clase sea idéntica en train y test.

**Por qué 80/20:** Es la partición estándar para datasets de este tamaño. Con 20% en test hay suficientes muestras de cada clase para una evaluación estadísticamente significativa, y con 80% en train hay suficientes datos para que SMOTEENN trabaje correctamente.

**Efecto en el ETL:** Establece la barrera de información entre entrenamiento y evaluación. Todo lo posterior se aplica solo a train o se aplica a test usando parámetros aprendidos exclusivamente de train.

---

## Celda 11 — Winsorización (Recorte de Percentiles)

**Qué se hizo:** Se calculan los percentiles 1 y 99 de cada columna numérica **usando solo el train set**, y se recortan los valores que caen fuera de ese rango en ambos conjuntos. Las columnas `_missing` se excluyen porque son binarias.

**Por qué:** Sin winsorización, algunos planetas con valores extremos (el 1% más lejano, el 1% más masivo) forman clusters aislados en el espacio de features. Cuando SMOTEENN genera sintéticos para las clases minoritarias, si algún planeta real de esa clase tiene un valor extremo, genera sintéticos cerca de ese extremo, creando una concentración artificial (el "pico" visible en los histogramas anteriores).

**Por qué percentil 1-99 y no eliminar las filas:** La winsorización recorta sin perder filas. El valor extremo no desaparece; se reemplaza por el valor del percentil 99. Esto preserva la información de que "ese planeta era el más extremo del rango valid" sin que su valor exacto distorsione el espacio de features.

**Por qué fitear solo en train:** Los percentiles se calculan exclusivamente sobre train. Si se calcularan sobre el dataset completo, los valores de test influirían en los límites de recorte, lo cual sería otra forma de leakage. El test se transforma usando los mismos límites del train, como si fuera un dato nuevo llegando a producción.

**Efecto en el ETL:** Reduce la influencia de outliers extremos sin perder filas, y acondiciona el espacio de features para que SMOTEENN genere sintéticos en zonas más representativas de la distribución real.

---

## Celda 12 — Imputación KNN

**Qué se hizo:** Se usa `KNNImputer(n_neighbors=5)` con `.fit_transform()` sobre el train y solo `.transform()` sobre el test. Las columnas `_missing` se excluyen del imputer. Las columnas categóricas se imputan con el string `'Unknown'`.

**Por qué KNN y no media/mediana:** Un planeta con radio desconocido probablemente tiene un radio similar al de otros planetas con características estelares y orbitales parecidas. KNN identifica los 5 vecinos más similares en el espacio de features y promedia sus valores. La media global, en cambio, trataría igual a un planeta de tipo Júpiter y a uno de tipo Tierra, lo cual es incorrecto físicamente. KNN respeta la estructura local de los datos.

**Por qué `n_neighbors=5`:** Es el valor por defecto y funciona bien para datasets de tamaño moderado. Valores muy bajos (1-2) hacen la imputación ruidosa; valores muy altos (>20) la hacen demasiado suavizada. 5 es un balance estándar.

**Por qué excluir las columnas `_missing`:** Estas columnas son indicadores binarios (0 o 1) creados a partir de los propios nulos que se están imputando. Incluirlas en el cálculo de distancias KNN sesgaría el resultado: el imputer priorizaría vecinos que también tienen ese valor ausente, creando un círculo vicioso. Son features para el modelo, no para el imputador.

**Por qué `'Unknown'` para categóricas:** Las columnas categóricas restantes tienen nulos escasos. Imputarlas con la moda introduciría sesgo hacia la categoría más frecuente. `'Unknown'` es una categoría honesta que le dice al modelo "no tenemos esta información", que es exactamente la situación.

**Por qué fit solo en train:** Si el imputer se fiteara sobre el dataset completo, los valores de test influirían en cómo se imputan los valores de train (al ser posibles vecinos). Esto haría que el modelo "vea" datos de test durante el entrenamiento.

**Efecto en el ETL:** Completa el dataset sin nulos usando información contextual de cada planeta, preservando la estructura física de los datos mejor que cualquier imputación global.

---

## Celda 13 — Balanceo con SMOTEENN

**Qué se hizo:** Se aplica SMOTEENN exclusivamente sobre el train set. Primero se encodean las columnas categóricas y el target con `LabelEncoder` (requerimiento técnico de la librería). Luego SMOTEENN genera muestras sintéticas y limpia el resultado.

**Por qué SMOTEENN y no oversampling por duplicación (`sample(replace=True)`):** El oversampling simple copia filas existentes exactamente. Si 200 planetas de Microlensing se deben convertir en 3000, se repiten las mismas 200 filas 15 veces. Esto crea picos artificiales en los histogramas en exactamente los valores donde se concentraban esos 200 planetas originales, y hace que el modelo memorice esas filas en lugar de aprender patrones generales.

**Por qué SMOTEENN y no SMOTE solo:** SMOTE interpola entre vecinos reales para crear sintéticos, lo cual es mejor que duplicar. Sin embargo, si los vecinos reales están concentrados en una zona específica del espacio de features (como `pl_rade` ~9 radios terrestres para Microlensing), los sintéticos también se concentran ahí, replicando el pico. ENN (Edited Nearest Neighbours) resuelve esto: después de generar los sintéticos, elimina cualquier muestra —real o sintética— que sus 3 vecinos más cercanos clasificarían de forma incorrecta. Los sintéticos en zonas de alta densidad de otra clase son exactamente los que ENN elimina.

**Por qué solo en train:** El test set debe representar la distribución real del problema (datos de exoplanetas como existen en la naturaleza, con todas las clases desbalanceadas). Si se balanceara el test, las métricas reflejarían un mundo artificial donde hay tantos planetas de Microlensing como de Transit, y el modelo en producción rendiría peor de lo esperado.

**Por qué `LabelEncoder` para categóricas:** SMOTEENN trabaja en espacios numéricos para calcular distancias. Los strings no tienen distancia definida. El encoding es temporal y técnico, no una decisión de representación del modelo final.

**Efecto en el ETL:** Produce un train set donde el modelo puede aprender con igual énfasis en cada clase, sin picos artificiales que distorsionen las distribuciones aprendidas, y sin leakage del test set.

---

## Celda 14 — Visualización Comparativa y Tablas de Frecuencia

**Qué se hizo:** Para cada feature clave se genera un gráfico lado a lado: distribución original (rojo) vs. distribución del train balanceado final (verde). Se agrega una tabla de frecuencias por bins.

**Por qué:** La visualización comparativa es el control de calidad final del ETL. Permite verificar que las transformaciones aplicadas lograron el efecto buscado sin introducir artefactos nuevos. Una distribución razonable después del procesamiento debe ser más suave, más comprimida en escala, y sin picos que no existían en el original.

**Qué significan los picos que persisten:** Si un pico existe en el gráfico verde **y también existía (en forma más extrema) en el rojo**, es una concentración real del fenómeno físico. Por ejemplo, la alta concentración de planetas con radio ~9 R⊕ refleja el sesgo de detección de Transit hacia planetas grandes. Eso no es un error del ETL sino una característica del dominio.

**Efecto en el ETL:** Sirve de documentación visual del proceso y como herramienta de debugging. Si aparece un pico que no existía en el original, señala un problema en algún paso anterior.

---

## Resumen del Pipeline y Orden de Decisiones

```
RAW CSV
  │
  ├─ [1] Carga sin modificar
  ├─ [2] Filtro: un registro por planeta, con target conocido
  ├─ [3] Reagrupación del target: 4 clases balanceables
  ├─ [4] Eliminación de leakage: IDs, metadatos, proxy del target, redundancias
  ├─ [5] Diagnóstico de nulos
  ├─ [6] Captura de ausencia informativa → columnas _missing
  ├─ [7] Eliminación de columnas >75% nulos
  ├─ [8] Corrección física: negativos y rangos imposibles → NaN
  ├─ [9] Log1p: comprimir órdenes de magnitud (sin parámetros = OK antes del split)
  │
  ├─ [10] ═══════════ TRAIN / TEST SPLIT ═══════════
  │         Todo lo siguiente usa SOLO datos de train para fitear
  │
  ├─ [11] Winsorización p1–p99: recortar extremos sin perder filas
  ├─ [12] KNN Imputer: completar nulos con contexto local
  ├─ [13] SMOTEENN: balanceo sintético + limpieza de zonas densas
  │
  └─ [14] Visualización comparativa: control de calidad final

SALIDA:
  X_train_bal / y_train_bal  →  para entrenar modelos
  X_test      / y_test       →  para evaluar modelos (distribución real)
```

---

## Nota Final — Los Picos en los Gráficos Están Bien

### Qué se ve en los gráficos

En la visualización comparativa (celda 14) se observan concentraciones pronunciadas en dos variables:

- **`pl_rade` ~2.3 en escala log1p** — equivale a aproximadamente 9 radios terrestres, es decir, planetas del tamaño de Júpiter.
- **`pl_orbper` ~6 en escala log1p** — equivale a aproximadamente 400 días de período orbital, cerca de la zona habitable estelar.

Estas concentraciones se ven como picos altos en el histograma verde (distribución final del train).

---

### Por qué son picos físicos reales y no artefactos del ETL

El argumento central es simple: **estos picos ya existían en el dato crudo, antes de cualquier procesamiento**. El gráfico rojo (ANTES) muestra exactamente las mismas concentraciones, en forma más extrema. El ETL no los creó.

La razón física es concreta. El método Transit detecta planetas midiendo el porcentaje de luz estelar que se bloquea cuando el planeta pasa frente a la estrella. Un planeta del tamaño de Júpiter bloquea mucho más luz que uno del tamaño de la Tierra, lo que hace que sea detectado con mayor facilidad, mayor frecuencia y mayor confianza estadística. El resultado es una concentración real y esperable de planetas grandes en el dataset. Si ese pico no existiera después del procesamiento, sería sospechoso, porque significaría que el ETL destruyó información real del dominio.

La prueba de que el pico no es artificial es que sobrevivió a dos filtros diseñados específicamente para eliminar concentraciones espurias: la **Winsorización** (que achata los valores más extremos de cada columna) y el componente **ENN de SMOTEENN** (que elimina muestras sintéticas ubicadas en zonas de alta densidad). Si el pico hubiera sido un artefacto del oversampling, ENN lo habría borrado. El hecho de que persista confirma que tiene respaldo en datos reales.

---

### Por qué los modelos de clasificación no se ven perjudicados por estos picos

Los algoritmos que típicamente se usan para este tipo de problema (Random Forest, XGBoost, SVM, redes neuronales) **no asumen ninguna distribución específica en los features**. No son modelos paramétricos como la regresión lineal, que sí se ve afectada por skewness o distribuciones no normales. Un árbol de decisión simplemente evalúa umbrales del tipo "¿`pl_rade` > 2.3?", y opera correctamente independientemente de si hay un pico en ese valor o no.

Lo que sí afecta la calidad de un clasificador es el **desbalance en el target**, es decir, que haya muchísimos más planetas de Transit que de Microlensing. Ese problema fue resuelto específicamente por SMOTEENN. La forma de la distribución de los features, en cambio, no es un problema para estos modelos.

---

### Por qué eliminar esos valores degradaría el modelo

El pico en `pl_rade` es probablemente uno de los **features más discriminantes** de todo el dataset. La razón es directa: los planetas descubiertos por Transit tienden a ser grandes (radio alto), mientras que los descubiertos por Radial Velocity tienden a ser masivos pero no necesariamente grandes, y los de Microlensing tienen distribuciones de tamaño distintas. Esa diferencia en la distribución de radios es exactamente la señal que el modelo necesita para separar las clases.

Si se aplicara undersampling agresivo, transformaciones adicionales, o cualquier técnica que aplane artificialmente ese pico, se estaría borrando la concentración de planetas grandes detectados por Transit. El modelo perdería precisamente la información que le permite distinguir Transit de las otras clases, lo que se traduciría en peores métricas de precisión y recall para esa clase en el test set.

En síntesis: **un pico que existe en el dato original, sobrevive a los filtros de limpieza, y corresponde a un fenómeno físico explicable, no debe ser eliminado**. Eliminarlo no es limpiar el dato, es distorsionarlo.
