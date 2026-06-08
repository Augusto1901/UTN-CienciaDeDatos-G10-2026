# 00 — Resumen Ejecutivo (1 página)

> **Cómo leer este material.** Cada afirmación está etiquetada:
> - **(REPO)** = verificable en un archivo/celda del repositorio.
> - **(DOMINIO)** = contexto astrofísico/estadístico que agregamos; no sale del repo.
> - **(DUDA)** = no se pudo verificar; queda como pregunta abierta para el grupo/profesora.
>
> Las **métricas y feature importances de las celdas 15–18 de `03_modelado_final.ipynb` NO están guardadas en el notebook**. Para no inventar, se reprodujo el pipeline determinista del repo (`random_state=42`) con scikit‑learn 1.9.0 / imbalanced‑learn 0.14.2. Esos números se etiquetan **(REPO‑reproducido)** y coinciden con los reportes de clasificación que sí están guardados (diferencias <1 punto, atribuibles a la versión de librería). Ver detalle en [04-resultados-y-conclusiones.md](04-resultados-y-conclusiones.md).

---

## Qué hace el proyecto

Construye un **clasificador multiclase** que predice **el método con el que se descubrió un exoplaneta** (`discoverymethod`) usando **solo sus propiedades físicas, orbitales y estelares** — nunca metadatos del proceso de descubrimiento. **(REPO** — `README.md` líneas 3‑6; `02_etl_parte2.ipynb` celda 0**)**

La idea de fondo: cada técnica de detección tiene **sesgos físicos conocidos** (detecta más fácil cierto tipo de planeta). Si esos sesgos existen, las variables del dataset deberían distribuirse distinto según el método, y un modelo debería poder recuperar esa relación. **(REPO** — hipótesis en `03_eda.ipynb` celda 1**)**

## Dataset

- **Fuente:** NASA Exoplanet Archive, gestionado por IPAC/Caltech; descargado el **10‑nov‑2024**. **(REPO** — `Descripcion del Dataset.pdf`**)**
- **Tamaño crudo:** **36 418 filas × 91 columnas**. **(REPO** — `01_data_exploration.ipynb` celda 3**)**
- **Unidad de análisis tras limpiar:** **5 757 planetas únicos** (una fila por planeta). **(REPO** — `02_etl_parte2.ipynb` celda 4**)**
- **Target (4 clases tras reagrupar):** `Transit` 4304 · `Radial Velocity` 1080 · `Microlensing` 230 · `Other/Others` 143. Fuerte desbalance (Transit ≈ 75 %). **(REPO** — `03_eda.ipynb` celda 5; `etl_de_cero_corregido.ipynb` celda 2**)**

## Pipeline (de punta a punta)

1. **Filtrado:** `default_flag == 1` (una entrada canónica por planeta) + descartar target nulo → 5757 filas. **(REPO)**
2. **Reagrupación del target:** se conservan los 3 métodos masivos y el resto va a `Other(s)`. **(REPO)**
3. **Anti‑leakage:** se eliminan IDs, metadatos, `disc_year`/`disc_facility`, errores/límites, unidades redundantes → quedan 22 columnas. **(REPO)**
4. **Nulos:** indicadores `_missing` (la ausencia es señal) + se eliminan columnas con >75 % de nulos (`pl_insol`, `st_spectype`). **(REPO)**
5. **Consistencia física** (negativos→NaN, excentricidad en [0,1]) y **log1p** sobre 5 variables muy sesgadas. **(REPO)**
6. **Existen DOS pipelines de preparación** (ver [02-pipeline-implementado.md](02-pipeline-implementado.md)):
   - **ETL para EDA** (`02_etl_parte2.ipynb` / `presentacion.ipynb`): imputa con **mediana global**, crea columnas `_log`, exporta `Data/processed/NASA_Exoplanet_Clean_DiscoveryMethod.csv` (42 columnas). **(REPO)**
   - **ETL para modelado** (`etl_de_cero_corregido.ipynb`, replicado dentro de `03_modelado_final.ipynb`): **split 80/20 estratificado**, **winsorización p1–p99**, **imputación KNN (k=5)** y **balanceo SMOTEENN** — todo fiteado solo en train. **(REPO)**
7. **Modelado:** **Random Forest** y **Gradient Boosting**, entrenados sobre el train balanceado (11 591 filas) y evaluados sobre el test real (1152 filas). **(REPO** — `03_modelado_final.ipynb` celda 14**)**

## Resultados

- **Accuracy en test ≈ 0,98** para ambos modelos; **F1 ponderado ≈ 0,98**. **(REPO** — celda 14**)**
- La clase fácil es **Transit** (F1 0,99); la más difícil es **Other(s)** (F1 ≈ 0,80, solo 29 casos). **(REPO** — celda 14**)**
- Sin overfitting relevante: diferencia train‑test ≈ 0,02 en ambos. **(REPO‑reproducido)**
- Variables más predictivas (Random Forest): **`sy_dist`, magnitudes de brillo (`sy_kmag/vmag/gaiamag`), `pl_orbsmax` y los indicadores `pl_bmasse_missing`/`pl_rade_missing`**. **(REPO‑reproducido)** — matiz importante discutido en [03-decisiones-y-porques.md](03-decisiones-y-porques.md).

## Conclusión en una frase

El método de descubrimiento de un exoplaneta **se puede predecir con alta exactitud a partir de sus propiedades físicas** porque cada técnica de detección impone un sesgo físico medible (radio, masa, distancia, brillo) — y el modelo recupera exactamente esos sesgos, validados estadísticamente en el EDA con tests de Kruskal‑Wallis (p ≪ 0,05 en las 3 hipótesis). **(REPO** — `03_eda.ipynb` celdas 13, 16, 20**)**
