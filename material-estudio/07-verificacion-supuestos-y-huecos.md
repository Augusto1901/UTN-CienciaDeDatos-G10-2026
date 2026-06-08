# 07 — Verificación Final: Supuestos y Huecos

> Releído cada afirmación **(REPO)** contra su archivo/celda. Lo que no se pudo probar quedó como **(DUDA)** o se reprodujo y etiquetó como **(REPO‑reproducido)**. Acá se listan los supuestos y los huecos a confirmar con el grupo o la profesora.

---

## (a) Supuestos que tuvimos que hacer

1. **Reproducción del modelado.** Las celdas 15‑18 de `03_modelado_final.ipynb` (feature importance, overfitting, matrices de confusión, conclusiones) **no tienen output guardado**. Se reprodujeron ejecutando el mismo código con `random_state=42`, pero con **scikit‑learn 1.9.0 / numpy 2.4.6 / imblearn 0.14.2** (no exactamente las versiones de `requirements.txt`: sklearn 1.8.0, numpy 2.4.4). Los reportes de clasificación reproducidos coinciden con los **guardados** (celda 14) dentro de **<1 punto**, lo que valida la reproducción; aun así, los valores de feature importance podrían variar levemente con las versiones exactas. **Supuesto:** esas pequeñas diferencias no cambian el ranking ni las conclusiones.

2. **"Mejor modelo".** El notebook elige por F1 ponderado. Con los números reproducidos, **Gradient Boosting** sale levemente mejor (F1 0,9832 vs 0,9773). El texto de conclusiones de la celda 18 se arma dinámicamente, así que el "mejor modelo" mostrado depende de la corrida. **Supuesto:** GB es el mejor, pero la diferencia es marginal y RF es prácticamente equivalente.

3. **Causas físicas (DOMINIO).** Las explicaciones de por qué cada método sesga hacia masa/radio/distancia son conocimiento astrofísico estándar, **coherente** con los tests del EDA, pero la **redacción causal** la aportamos nosotros; el repo prueba la **correlación/diferencia estadística**, no la causa física en sí.

4. **Interpretación de los `_missing` como "leakage suave".** Es una lectura crítica nuestra **(DOMINIO/DUDA)**; el repo lo defiende como "ausencia informativa" legítima. No está saldado en el repo cuál de las dos lecturas adoptar.

5. **Unificación conceptual de los dos ETL.** Asumimos que el ETL "para EDA" (mediana) y el "para modelado" (KNN+SMOTEENN) son **etapas/iteraciones distintas** del proyecto que conviven, no un error. El repo no documenta explícitamente por qué hay dos.

## (b) Huecos del repo a aclarar con el grupo / profesora

1. **`imbalanced-learn` falta en `requirements.txt`.** El ETL/modelado lo importan (`from imblearn.combine import SMOTEENN`) pero no está listado. Quien clone el repo y siga el README **no podrá ejecutar** sin instalarlo aparte. → Agregar `imbalanced-learn` a `requirements.txt`. **(REPO** — `requirements.txt` vs imports**)**

2. **Outputs de las celdas 15‑18 no guardados.** Conviene re‑ejecutar y guardar el notebook para que las feature importances, matrices y conclusiones queden versionadas (hoy hay que reproducirlas). **(REPO)**

3. **Inconsistencia de un output en `etl_de_cero_corregido.ipynb` celda 3:** imprime "Columnas eliminadas: **19**" pero el shape resultante es (5757, **22**), que implica 69 columnas eliminadas (91−69=22). El conteo "19" es un **output desactualizado** (corresponde solo a las columnas listadas explícitamente, sin las 50 de error/límite). En `03_modelado_final.ipynb` el mismo paso imprime correctamente **69**. → Re‑ejecutar para sincronizar. **(REPO** — comparar celda 3 de ambos notebooks**)**

4. **Dos ETL en paralelo.** No está documentado en el README por qué coexisten (mediana vs KNN). ¿El de mediana quedó como histórico de la 2ª entrega y el de KNN es el definitivo? Confirmar con el grupo. **(DUDA)**

5. **`README.md` desactualizado respecto al modelado.** Describe Sprint 3 como "En progreso" y nombra `03_eda.ipynb` y `presentacion.ipynb`, pero no menciona `03_modelado_final.ipynb` ni `etl_de_cero_corregido.ipynb` (los archivos de la última entrega). → Actualizar el README. **(REPO** — `README.md` líneas 27 y 10‑14**)**

6. **`Docs/proceso_ETL` está vacío (0 bytes)** y hay documentos `.docx`/`.pptx` que no se pudieron inspeccionar en texto plano (`2026-CD-5K4-GPO_10.docx`, `ETL-Procedimiento.docx`, `Presentación - Proyecto de Ciencia de Datos.pptx`). Pueden contener material adicional de la exposición que conviene revisar manualmente. **(DUDA)** — no analizados en este material.

7. **Microlensing y `pl_rade`.** En el EDA (celda 13) la tabla de medianas de radio **medido** muestra solo Other, Radial Velocity y Transit; Microlensing no aparece porque casi no tiene radios medidos (su masa/radio suelen ser inciertos). Es coherente con el dominio, pero conviene tenerlo presente para no afirmar de más sobre el radio de Microlensing. **(REPO** — `03_eda.ipynb` celda 13**)**

8. **Curso/grupo (A vs B).** El README dice "Curso 5K4"; el TP fija fechas distintas para Grupo A (Tercera Entrega 03/06) y Grupo B (10/06). Hoy (08/06/2026) el grupo está entre la 3ª y 4ª entrega. Confirmar a qué grupo (A/B) pertenece el G10 para alinear la fecha de la defensa oral. **(REPO** — `Docs/Trabajo Práctico 2026.pdf`**)**

---

### Checklist de coherencia (auto‑verificación)
- [x] 36 418 filas crudas — `01_data_exploration.ipynb` celda 3 ✔
- [x] 5757 planetas únicos — `presentacion.ipynb` celda 4 ✔
- [x] 69 columnas eliminadas → shape (5757, 22) — reproducido + `03_modelado_final.ipynb` celda 3 ✔
- [x] Drop >75 %: `pl_insol`, `st_spectype` (pl_eqt se salva) — `etl_de_cero_corregido.ipynb` celda 6 ✔
- [x] Train 4605 / Test 1152; SMOTEENN → 11 591 — celdas 9 y 12 ✔
- [x] Accuracy ≈ 0,98 ambos modelos — celda 14 (guardado) ✔
- [x] Kruskal H1/H2/H3 con p ≪ 0,05; Spearman masa‑radio rho=0,822 — `03_eda.ipynb` celdas 13/16/20/23 ✔
