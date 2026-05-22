# Predicción del Método de Descubrimiento de Exoplanetas - UTN Ciencia de Datos (Grupo 10)

## Descripción del Proyecto
Este proyecto tiene como objetivo desarrollar un modelo de Machine Learning capaz de predecir el **Método de Descubrimiento** (`discoverymethod`) de un exoplaneta utilizando exclusivamente sus características físicas, orbitales y estelares. 

A diferencia de los enfoques tradicionales, este modelo busca identificar los sesgos instrumentales y físicos de los diferentes métodos (Tránsito, Velocidad Radial, Microlente, etc.) mediante una **clasificación multiclase**, permitiendo entender qué tipo de planetas son más probables de ser detectados por cada tecnología actual.

## Estructura del Repositorio
* `Data/`: Contiene el dataset original de la NASA (`NASA_Exoplanet_Archive_Data.csv`) y los archivos procesados tras el ETL en la subcarpeta `processed/`.
* `Notebooks/`: Jupyter Notebooks ordenados secuencialmente:
    * `01_data_exploration.ipynb`: Primer contacto con el dataset y comprensión de variables.
    * `02_etl.ipynb` / `02_etl_parte2.ipynb`: Limpieza, tratamiento de nulos, ingeniería de variables y preparación del target.
    * `03_eda.ipynb`: Análisis exploratorio profundo y validación estadística de hipótesis.
    * `presentacion.ipynb`: Resumen ejecutivo de los hallazgos.
* `Docs/`: Documentación oficial, informes de avance y presentaciones del proyecto.

## Dataset
Se utiliza el **NASA Exoplanet Archive Data**, centrándose en variables críticas:
* **Físicas:** Radio (`pl_rade`), Masa (`pl_bmasse`), Temperatura de equilibrio (`pl_eqt`).
* **Orbitales:** Período orbital (`pl_orbper`), Semieje mayor (`pl_orbsmax`), Excentricidad.
* **Estelares:** Masa estelar (`st_mass`), Radio estelar (`st_rad`), Temperatura efectiva (`st_teff`), Metalicidad (`st_met`).

## Hoja de Ruta (Sprints)
1. **Sprint 0:** Configuración del entorno y selección del dataset. (Finalizado)
2. **Sprint 1:** Proceso ETL, limpieza de datos y tratamiento de nulos. (Finalizado)
3. **Sprint 2:** Análisis Exploratorio de Datos (EDA) y validación de hipótesis físicas. (Finalizado)
4. **Sprint 3:** Selección de algoritmos, balanceo de clases y entrenamiento de modelos. (En progreso)
5. **Sprint 4:** Evaluación de métricas (F1-Score) e interpretación de resultados.

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
3. Seguir el orden numérico de los notebooks en la carpeta `Notebooks/`.

---
© 2026 - Universidad Tecnológica Nacional (UTN)
