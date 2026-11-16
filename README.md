# Modelamiento de biorreactores fed-batch

Repositorio de trabajo para el curso de la **Unidad de Investigación DIQBM, Facultad de Ciencias Físicas y Matemáticas, Universidad de Chile** (Prof. J. Cristian Salgado; ayudantes Roberto Aliaga Medina y Darko Yurac).

Contiene dos notebooks complementarios:

| Notebook | Rol | Sistema |
|---|---|---|
| `SV1_2.ipynb` | Caso base: construir una función de modelamiento y optimización de un biorreactor | Células de hibridoma (producción de MAb), 1 alimentación |
| `tarea_fedbatch.ipynb` | Actividad: primer acercamiento al uso de redes neuronales para estimar un parámetro (μ) y base para un modelo híbrido | Células CHO, 2 alimentaciones (glucosa y glutamina) |

---

## 1. `SV1_2.ipynb`: caso base de modelamiento de un biorreactor

**Objetivo:** usar un caso de la literatura como referencia para construir una función reutilizable de modelamiento de un biorreactor fed-batch (cinéticas + balances de masa + simulación + optimización), que luego se adapta a otros sistemas.

**Sistema:** cultivo fed-batch de células de hibridoma con una sola corriente de alimentación, basado en Tremblay et al. (1992).

- **Estados (7):** `Xv`, `Glc`, `Gln`, `Lac`, `Amm`, `MAb`, `V`.
- **Cinética:** crecimiento tipo Monod de doble sustrato (glucosa y glutamina), muerte celular dependiente de lactato, amonio y glutamina, consumos específicos y producción de subproductos y anticuerpo.
- **Unidades de tiempo:** días.

**Contenido:**

1. **Celda 1:** parámetros, condiciones iniciales, función `kinetics`, EDOs `fedbatch_ode` y utilidad `simulate(F_func, ...)` para simular con cualquier perfil de alimentación. Incluye una verificación batch contra el valor del paper (~67 mg/L de MAb; el notebook entrega ~68.5 mg/L).
2. **Celda 2:** optimización global del perfil de alimentación con `scipy.optimize.differential_evolution`. El perfil se parametriza como 20 flujos constantes por tramo en 10 días, con `F ∈ [0, 0.5] L/d`, y el objetivo es maximizar el MAb total (`MAb · V`). Reconstruye la trayectoria y grafica el perfil óptimo.

**Resultado de referencia (salida guardada en el notebook):** ~152 mg/L de MAb y ~305 mg totales con volumen final de 2.0 L, frente a ~147 mg/L y ~294 mg reportados en el paper.

---

## 2. `tarea_fedbatch.ipynb`: actividad de redes neuronales para estimar μ

**Objetivo:** primer acercamiento básico al uso de redes neuronales para calcular un parámetro del modelo, en este caso la **tasa específica de crecimiento celular (μ)**. La meta de la actividad completa es reemplazar la expresión algebraica de μ del modelo mecanístico por la salida de la red y construir así un **modelo híbrido**, comparándolo con el modelo mecanístico y con los datos de referencia.

**Contexto del problema (resumen del enunciado):**

- Cultivo fed-batch de células CHO con dos alimentaciones independientes: glucosa (`F_glc`) y glutamina/nutrientes complementarios (`F_glu`).
- Modelo mecanístico de **8 estados**: `Xv`, `Xd`, `GLC`, `LAC`, `ALA`, `GLU`, `AMM`, `V`, con μ como función de glucosa y glutamina (suma de dos términos tipo Hill).
- Los datos son **sintéticos (in silico)**: 20 simulaciones de un controlador NMPC con ±5 % de variación en condiciones iniciales y ruido de medición. No son datos experimentales de laboratorio.

**Contenido del notebook:**

1. **Celda 0, red neuronal (`MLPRegressor` de scikit-learn):**
   - Carga `train_noisy.csv` y `test_noisy.csv` (índice `batch_id`), elimina filas con NaN y excluye de las entradas `mu_true`, `mud_true`, `F_glu`, `t_h` y `GLC`. La variable objetivo es `mu_true`.
   - Normaliza con `StandardScaler` (ajustado solo con train).
   - Configuración actual: una capa oculta de 10 neuronas, activación `tanh`, solver `lbfgs`, `max_iter=1000`, `tol=1e-6`.
   - Métricas sobre test: R² ≈ 0.72, MAPE ≈ 0.037, MSE ≈ 4.9e-7.
2. **Celdas 1 a 3, modelo mecanístico:** parámetros de la Tabla 1 del enunciado, condiciones iniciales, `kinetics`, `fedbatch_odes` y `simulate`, con ambas alimentaciones. Estas celdas son la base donde luego se reemplaza μ por la red neuronal.
3. **Celdas 4 a 8, optimización y trayectorias:** optimización del perfil de alimentación (`F_glc`, `F_glu`) con `differential_evolution` (20 intervalos en 168 h, `F ∈ [0, 5e-4] L/h` por corriente, objetivo: maximizar `Xv · V`), reconstrucción y gráfico de trayectorias, y guardado/carga de resultados en `optim_fedbatch.pkl`.

**Estado actual y pendientes respecto al enunciado:**

- La red de la celda 0 es una primera iteración exploratoria; el enunciado pide dos capas ocultas de 50 neuronas y entrenar solo con la fase batch.
- Falta la evaluación sobre validación y la discusión de extrapolación a fed-batch.
- Falta integrar la red al modelo (modelo híbrido) y compararlo con el mecanístico en los lotes de test.
- Falta la presentación final con referencias.

---

## Estructura sugerida del repositorio

```
.
├── README.md
├── SV1_2.ipynb              # caso base (hibridoma, 1 alimentación)
├── tarea_fedbatch.ipynb     # actividad: NN para μ + modelo CHO
└── data/                    # train_noisy.csv, test_noisy.csv, (validación)
```

## Requisitos

- Python 3.x
- `numpy`, `scipy`, `pandas`, `matplotlib`, `scikit-learn`

```bash
pip install numpy scipy pandas matplotlib scikit-learn
```

Los notebooks usan un kernel llamado `Bioreactors`; puedes cambiarlo por cualquier entorno que tenga las dependencias.

## Notas importantes

- **Datos no incluidos:** `tarea_fedbatch.ipynb` lee `train_noisy.csv` y `test_noisy.csv` desde una ruta local de Windows (`C:\Users\...\UI_Bioreactores\`). Hay que descargar los datos del curso y ajustar esas rutas (por ejemplo, a `data/train_noisy.csv`) antes de ejecutar la celda 0.
- **Unidades de tiempo distintas:** `SV1_2.ipynb` trabaja en días y `tarea_fedbatch.ipynb` en horas.
- **Optimización costosa:** `differential_evolution` con 300 iteraciones puede tardar varios minutos. `tarea_fedbatch.ipynb` guarda el resultado en `optim_fedbatch.pkl` para no repetirlo.

## Referencias

- Tremblay, M., Perrier, M., Chavarie, C., Archambault, J. (1992). *Optimization of fed-batch culture of hybridoma cells using dynamic programming: single and multi feed cases.* Base del modelo de `SV1_2.ipynb`.
- Enunciado del curso: *Desarrollo de un modelo híbrido para un cultivo fed-batch*, DIQBM, Universidad de Chile (Otoño 2026).
