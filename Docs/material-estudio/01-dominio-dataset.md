# 01 — Dominio y Dataset

## 1. De qué trata (el dominio)

El proyecto trabaja con **exoplanetas**: planetas que orbitan estrellas distintas del Sol. **(DOMINIO)**

La variable a predecir es el **método de descubrimiento**: la técnica observacional con la que se detectó cada planeta. Esto es interesante porque **ningún método "ve" el planeta directamente la mayoría de las veces**; cada técnica infiere su existencia midiendo un efecto distinto, y por eso cada una **favorece** un tipo de planeta. **(DOMINIO)**

Los tres métodos masivos del dataset y su principio físico **(DOMINIO**, confirmado en `justificacion_etl.md` celda 3 y `03_eda.ipynb` celda 1 **(REPO))**:

| Método | Qué mide | Sesgo (qué planeta detecta mejor) |
|---|---|---|
| **Transit (Tránsito)** | La **caída de brillo** cuando el planeta pasa frente a su estrella. La señal ∝ (radio planeta / radio estrella)². | Planetas de **radio grande** y **período corto** (más tránsitos, más señal). |
| **Radial Velocity (Velocidad Radial)** | El **bamboleo** (corrimiento Doppler) de la estrella por el tirón gravitatorio del planeta. La amplitud ∝ masa del planeta. | Planetas **masivos** y relativamente cercanos a su estrella. |
| **Microlensing (Microlente gravitacional)** | La **amplificación de la luz** de una estrella de fondo cuando un sistema planetario pasa por delante y curva su luz (lente gravitacional). | Sistemas a **gran distancia** (kiloparsecs), no depende de ver luz del planeta ni de la estrella anfitriona. |

> La categoría **`Other(s)`** agrupa métodos minoritarios reales del archivo: Imaging (imagen directa), Transit Timing Variations, Eclipse Timing Variations, Pulsar Timing, Orbital Brightness Modulation, Astrometry, Pulsation Timing Variations y Disk Kinematics. **(REPO** — `presentacion.ipynb` celda 12: `Clases agrupadas como Other`**)**

## 2. Fuente y procedencia

- **Origen:** NASA Exoplanet Archive (la NASA es la fuente original). Gestionado por el **IPAC** (Infrared Processing and Analysis Center) de **Caltech**. **(REPO** — `Descripcion del Dataset.pdf` p.1**)**
- **Fecha de extracción:** **10 de noviembre de 2024**, acceso público. **(REPO** — `Descripcion del Dataset.pdf` p.1**)**
- **Limpieza que ya trae el archivo descargado** (hecha por quien lo publicó, antes de que lo recibiera el grupo): se quitó la columna índice; se excluyeron las 100 filas de metadatos del CSV original; se reformatearon las referencias (`*_refname`) a solo el hipervínculo; y **se eliminaron los planetas con `pl_controv_flag == 1`** (existencia cuestionada) — 30 entradas controvertidas/no confirmadas. **(REPO** — `Descripcion del Dataset.pdf` p.1**)**

## 3. Tamaño y estructura

| Aspecto | Valor | Fuente |
|---|---|---|
| Filas crudas | 36 418 | **(REPO** — `01_data_exploration.ipynb` celda 3**)** |
| Columnas crudas | 91 | **(REPO** — íd.**)** |
| Planetas únicos | 5 757 | **(REPO** — `02_etl_parte2.ipynb` celda 4**)** |

**Por qué hay más filas que planetas:** el archivo tiene **varias entradas por planeta**, una por cada publicación científica que midió ese objeto. La columna `default_flag = 1` marca la entrada "oficial"/preferida. Sin filtrar, el mismo planeta aparece repetido con valores algo distintos (pseudoduplicados). Tras `default_flag == 1` hay exactamente **1 fila por planeta** (5757 filas = 5757 `pl_name` únicos, 0 duplicados). **(REPO** — `presentacion.ipynb` celda 4: "Planetas duplicados luego del filtro: Series vacía"**)**

## 4. La variable objetivo (`discoverymethod`)

Distribución **cruda** (36 418 filas, 11 clases): **(REPO** — `presentacion.ipynb` celda 4**)**

```
Transit 32841 · Radial Velocity 2544 · Microlensing 689 · Imaging 145 ·
Transit Timing Variations 142 · Eclipse Timing Variations 22 ·
Orbital Brightness Modulation 16 · Pulsar Timing 13 · Astrometry 3 ·
Pulsation Timing Variations 2 · Disk Kinematics 1
```

Distribución tras `default_flag==1` y reagrupar (4 clases): **(REPO** — `03_eda.ipynb` celda 5**)**

| Clase | Cantidad | % |
|---|---|---|
| Transit | 4304 | 74,76 % |
| Radial Velocity | 1080 | 18,76 % |
| Microlensing | 230 | 4,00 % |
| Other(s) | 143 | 2,48 % |

→ **Dataset fuertemente desbalanceado.** Esto condiciona todo: obliga a estratificar el split, a usar métricas robustas (F1) y a balancear el train (SMOTEENN). **(REPO** — `03_eda.ipynb` celda 27**)**

## 5. Diccionario de variables relevantes (con unidades)

Tomado del diccionario oficial **(REPO** — `Descripcion del Dataset.pdf` pp.2‑5**)**. Solo las que sobreviven al ETL y/o se usan en análisis.

### Variables planetarias (`pl_*`)
| Columna | Significado | Unidad |
|---|---|---|
| `pl_orbper` | Período orbital | días |
| `pl_orbsmax` | Semieje mayor de la órbita | UA (unidades astronómicas) |
| `pl_rade` | **Radio del planeta** | radios terrestres (R⊕) |
| `pl_bmasse` | **Masa del planeta** (best mass) | masas terrestres (M⊕) |
| `pl_orbeccen` | Excentricidad orbital (0 = círculo, →1 = muy elíptica) | adimensional [0,1] |
| `pl_insol` | Flujo de insolación (radiación estelar recibida) | flujos terrestres |
| `pl_eqt` | Temperatura de equilibrio del planeta | Kelvin |
| `pl_radj`, `pl_bmassj` | Radio/masa en unidades de **Júpiter** (redundantes con R⊕/M⊕) | R_Júpiter / M_Júpiter |
| `pl_bmassprov` | Procedencia de la medición de masa (p. ej. `Msini`) | categórica |

### Variables estelares (`st_*`) — de la estrella anfitriona
| Columna | Significado | Unidad |
|---|---|---|
| `st_teff` | Temperatura efectiva estelar | Kelvin |
| `st_rad` | Radio estelar | radios solares (R☉) |
| `st_mass` | Masa estelar | masas solares (M☉) |
| `st_met` | Metalicidad estelar ([Fe/H]) | dex |
| `st_logg` | Gravedad superficial estelar | log(cm/s²) |
| `st_spectype` | Tipo espectral (O,B,A,F,G,K,M…) | categórica |

### Variables del sistema / observación (`sy_*`, coordenadas)
| Columna | Significado | Unidad |
|---|---|---|
| `sy_dist` | **Distancia al sistema** | parsecs (pc) |
| `sy_snum`, `sy_pnum` | N.º de estrellas / planetas del sistema | conteo |
| `sy_vmag` | Magnitud visual (brillo) | magnitud |
| `sy_kmag` | Magnitud en banda K (infrarrojo) | magnitud |
| `sy_gaiamag` | Magnitud de Gaia | magnitud |
| `ra`, `dec` | Ascensión recta / declinación (posición en el cielo) | grados |

### Columnas de proceso (se eliminan por leakage)
`disc_year` (año), `disc_facility` (observatorio), `soltype`, `default_flag`, `ttv_flag`, `*err1/*err2/*lim` (incertidumbres/límites), `*_refname` (referencias), `rastr/decstr` (coordenadas en texto). **(REPO** — `02_etl_parte2.ipynb` celda 6**)**

> **Nota de unidades (DOMINIO):** 1 magnitud astronómica es una escala **logarítmica e inversa**: cuanto **menor** la magnitud, **más brillante** la estrella. 1 parsec ≈ 3,26 años luz. R⊕ = radio terrestre; un Júpiter ≈ 11,2 R⊕ y ≈ 318 M⊕.

## 6. Glosario inicial del dominio
Ver glosario completo en [06-glosario.md](06-glosario.md). Términos imprescindibles: **exoplaneta, tránsito, velocidad radial, microlente, metalicidad, temperatura de equilibrio, semieje mayor, excentricidad, parsec, magnitud, Msini**.
