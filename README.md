# Predicción del Método de Descubrimiento de Exoplanetas - UTN Ciencia de Datos (Grupo 10)

## Descripción del Proyecto
Este proyecto tiene como objetivo desarrollar un modelo de Machine Learning capaz de predecir el **Método de Descubrimiento** (`discoverymethod`) de un exoplaneta utilizando exclusivamente sus características físicas, orbitales y estelares. 

A diferencia de los enfoques tradicionales, este modelo busca identificar los sesgos instrumentales y físicos de los diferentes métodos (Tránsito, Velocidad Radial, Microlente, etc.) mediante una **clasificación multiclase**, permitiendo entender qué tipo de planetas son más probables de ser detectados por cada tecnología actual.

El pipeline del proyecto ha sido estructurado y modularizado de forma robusta para evitar la fuga de información (*Data Leakage*), dividiendo el proceso de preparación de datos del modelado en dos etapas bien diferenciadas.

## Estructura del Repositorio
* **`Data/`**: Contiene el conjunto de datos astronómicos originales y procesados:
    * `NASA_Exoplanet_Archive_Data.csv`: Dataset original bruto extraído de la NASA.
    * `processed/`: Carpeta con los cuatro datasets limpios exportados tras el proceso de ETL:
        * `X_train_bal.csv` / `y_train_bal.csv`: Datos de características y objetivos de entrenamiento balanceados sintéticamente con SMOTEENN.
        * `X_test.csv` / `y_test.csv`: Datos de características y objetivos de prueba que conservan la distribución real del universo de exoplanetas.
* **`Notebooks/`**: Jupyter Notebooks principales ordenados secuencialmente:
    * `01_data_exploration.ipynb`: Primer contacto con el dataset bruto, tipos de datos y análisis preliminar de nulos.
    * `02_ETL.ipynb`: Pipeline de ETL corregido. Incluye limpieza de datos, mitigación de data leakage (eliminación de 69 columnas proxys o referencias), tratamiento de consistencia física, ingeniería de variables de ausencia (`_missing`), Winsorización e imputación KNN ajustadas estrictamente sobre el set de entrenamiento. Exporta los cuatro archivos CSV finales.
    * `03_modelado.ipynb`: Etapa de Machine Learning. Carga los datasets exportados y entrena tres modelos: Random Forest, Gradient Boosting y Regresión Logística multinomial (baseline). Realiza la comparación de métricas (priorizando F1-Macro), análisis de overfitting e interpretación física de variables predictivas.
    * `investigación y pruebas/`: Subcarpeta con notebooks experimentales o versiones de entregas anteriores.
* **`Docs/`**: Documentación oficial de la UTN, consignas del trabajo práctico e informes de avance.
    * `material-estudio/`**: Diapositivas, notas, cuestionarios de examen y guías de defensa oral.

## Dataset
Se utiliza el **NASA Exoplanet Archive Data**, centrándose en variables críticas:
* **Físicas:** Radio (`pl_rade`), Masa (`pl_bmasse`), Temperatura de equilibrio (`pl_eqt`).
* **Orbitales:** Período orbital (`pl_orbper`), Semieje mayor (`pl_orbsmax`), Excentricidad (`pl_orbeccen`).
* **Estelares:** Masa estelar (`st_mass`), Radio estelar (`st_rad`), Temperatura efectiva (`st_teff`), Metalicidad (`st_met`), Gravedad superficial (`st_logg`).

## Hoja de Ruta (Sprints)
1. **Sprint 0:** Configuración del entorno y selección del dataset. (Finalizado)
2. **Sprint 1:** Proceso ETL, limpieza de datos y tratamiento de nulos. (Finalizado)
3. **Sprint 2:** Análisis Exploratorio de Datos (EDA) y validación de hipótesis físicas. (Finalizado)
4. **Sprint 3:** Selección de algoritmos, balanceo de clases y entrenamiento de modelos. (Finalizado)
5. **Sprint 4:** Evaluación comparativa de métricas (F1-Score) e interpretación de resultados. (Finalizado)

## Integrantes del Grupo 10 (Curso 5K4)
* Roffe Apra, Bautista
* Belli, Marcos Ignacio
* Alfonso, Francisco
* Pereira Duarte, Manuel
* Yorio, Santino Baltazar
* Escudero, Jeremías
* Bossi, Augusto
* Lovera, Federico

## Cómo utilizar este repositorio
1. Clonar el repositorio.
2. Instalar dependencias: `pip install -r requirements.txt`.
3. Ejecutar los notebooks en el orden numérico establecido dentro de `Notebooks/`:
    * Ejecutar `02_ETL.ipynb` para limpiar, balancear y exportar los datos a `Data/processed/`.
    * Ejecutar `03_modelado.ipynb` para cargar los datos procesados y entrenar los modelos.

---
© 2026 - Universidad Tecnológica Nacional (UTN)
