# Material de Estudio — TP Integral Ciencia de Datos (G10, UTN, 2026)

Material para preparar la **defensa oral** del proyecto: *Predicción del Método de Descubrimiento de Exoplanetas* (NASA Exoplanet Archive). Pensado para que **cualquiera del grupo** pueda exponer y responder preguntas de la profesora.

## Orden de lectura sugerido
1. [00 — Resumen ejecutivo](00-resumen-ejecutivo.md) — el proyecto en 1 página.
2. [01 — Dominio y dataset](01-dominio-dataset.md) — qué es, fuente, variables, unidades.
3. [02 — Pipeline implementado](02-pipeline-implementado.md) — paso a paso, con celdas. **Ojo: hay 2 ETL.**
4. [03 — Decisiones y porqués](03-decisiones-y-porques.md) — cada decisión técnica justificada (físico + estadístico).
5. [04 — Resultados y conclusiones](04-resultados-y-conclusiones.md) — métricas reales y su significado.
6. [05 — Preguntas y respuestas](05-preguntas-y-respuestas.md) — 28 preguntas tipo examen, incluidas las "por qué".
7. [06 — Glosario](06-glosario.md) — términos del dominio y técnicos.
8. [07 — Verificación, supuestos y huecos](07-verificacion-supuestos-y-huecos.md) — qué confirmar con el grupo.
9. [08 — Speech de defensa](08-speech-defensa.md) — guion de 7 minutos en 3 bloques + tarjeta de números + pivotes para preguntas.

## Convención de etiquetas
- **(REPO)** — hecho verificable en un archivo/celda del repo.
- **(REPO‑reproducido)** — output no guardado en el `.ipynb`, reproducido ejecutando el código determinista (`random_state=42`).
- **(DOMINIO)** — contexto astrofísico/estadístico aportado, no sale del repo.
- **(DUDA)** — no verificable; pregunta abierta.

## Las 4 cosas que SÍ o SÍ hay que saber
1. **Qué se predice y con qué:** el método de descubrimiento, usando solo física del planeta/estrella (nunca metadatos del proceso) → evitar *data leakage*.
2. **Los 3 sesgos físicos:** Tránsito→radio grande · Velocidad Radial→masa grande · Microlente→distancia grande. Validados con Kruskal‑Wallis (p ≪ 0,05).
3. **El pipeline de modelado:** filtro → anti‑leakage → `_missing` → log1p → **split estratificado** → winsorización → **KNN** → **SMOTEENN** (todo fiteado solo en train) → **Random Forest / Gradient Boosting + baseline Regresión Logística**.
4. **Resultado:** accuracy ≈ 0,98 y **F1 Macro ≈ 0,93** (RF y GB empatados), baseline lineal 0,85 → la relación es no lineal; sin overfitting; features top = `sy_dist`, brillo y los `_missing`.
