# Protocolo congelado

**Proyecto:** Caso 1 · Riesgo de mora en préstamos nuevos — Cooperativa Progreso del Sur
**Etapa:** 6 · Congelar
**Congelado el:** *6 de agosto del 2026.*

Este documento registra todas las decisiones del procedimiento **antes de
abrir el conjunto de test**. A partir del commit que lo sube al repositorio,
nada de lo aquí escrito se modifica. El test se abre una sola vez; cualquier
cambio posterior a esa apertura invalida la evaluación y exige una nueva
evaluación independiente con datos no utilizados.

## 1. Variables definitivas (entran al modelo)

Numéricas (9): `age`, `monthly_income`, `requested_amount`,
`loan_term_months`, `existing_debt`, `months_in_job`,
`prior_late_payments`, `payment_history_score`, `debt_to_income`.

Categóricas (2): `branch_id`, `employment_type`.

## 2. Variables excluidas del modelo

Las 10 columnas restantes del dataset no entran como predictores. El
razonamiento completo está en `reports/auditoria_fuga.md`.

| Variable | Rol |
|---|---|
| `client_id` | Identificador; llave de la separación de clientes |
| `application_date` | Eje temporal; agrupación semanal y partición |
| `manual_review` | Intervención |
| `approved` | Posterior; define qué filas tienen etiqueta |
| `days_past_due_6m` | Origen del target; fuga como predictor |
| `default_30d` | Target |
| `collection_calls_6m` | Fuga |
| `restructured_after_default` | Fuga |
| `legal_collection_status` | Fuga |
| `p_mora_sin_intervencion` | Solo auditoría (diseño propio) |

## 3. Reglas de limpieza

- Se entrena únicamente con solicitudes aprobadas (`approved == 1`), porque
  solo ellas tienen etiqueta observada.
- No se eliminan valores atípicos ni filas adicionales.
- La única remoción de filas es la limpieza de clientes de la partición
  (Etapa 4), ya ejecutada: 1,639 filas del desarrollo y 534 del
  entrenamiento.
- Los valores faltantes no se eliminan: se imputan (sección 4).

## 4. Imputación

Mediana por variable, calculada **solo con entrenamiento** y aplicada sin
recalcular a validación y test. Cada variable con faltantes genera además
una columna indicadora de "estaba vacío", porque el vacío informa (cliente
sin historial, empleo informal). Implementación:
`SimpleImputer(strategy="median", add_indicator=True)` dentro del Pipeline.

## 5. Codificación

- Categóricas: one-hot (`OneHotEncoder(handle_unknown="ignore")`), con
  categorías aprendidas solo de entrenamiento.
- Numéricas: estandarización (`StandardScaler`) ajustada solo con
  entrenamiento.

## 6. Algoritmo

Regresión logística (`LogisticRegression` de scikit-learn), dentro del
Pipeline con el preprocesamiento anterior. Se eligió en validación: superó
al gradient boosting (86.2% vs 83.0% de recall en capacidad) siendo además
interpretable, por lo que la complejidad adicional no se justifica.

## 7. Hiperparámetros y semillas

- `max_iter = 2000`; el resto en sus valores por defecto (penalización L2,
  `C = 1.0`, solver lbfgs).
- Semillas: `42` para la generación de datos y los modelos; `43` para el
  orden aleatorio del baseline trivial y los desempates.
- Las versiones exactas de las librerías quedan fijadas en
  `requirements.txt`.
- Los modelos **no se reentrenan** para el test: se aplican tal como
  quedaron ajustados con el bloque de entrenamiento.

## 8. Métrica

**Principal — recall en capacidad (recall@120):** proporción de las moras
reales del período que quedan dentro de las 120 solicitudes seleccionadas
cada semana. Decide la comparación entre métodos.

De apoyo (se reportan, no deciden): precisión en capacidad y AUC.

Justificación: según los supuestos de costo congelados (sección 13), un
falso negativo cuesta ~30 veces más que un falso positivo; lo que importa
es cuántas moras reciben revisión.

## 9. Umbral

No se usa umbral de probabilidad. El corte de decisión es por capacidad:
cada semana se seleccionan exactamente las 120 solicitudes con mayor
puntaje (empates resueltos por orden estable). Si en una semana llegan 120
solicitudes o menos, se seleccionan todas.

## 10. Baselines

1. **Trivial:** orden aleatorio con semilla 43 (equivale a "nadie tendrá
   mora": ningún criterio de prioridad).
2. **Histórico:** puntaje = tasa de mora de la sucursal, calculada solo con
   entrenamiento. Valores congelados: Azua 0.1319, Baní 0.0917,
   San Cristóbal 0.1085, Santo Domingo 0.1071.
3. **Operativo:** la regla vigente de la cooperativa:
   puntaje = 2.0 × mín(`debt_to_income`, 1.2) + 0.5 × mín(`prior_late_payments`, 3).

## 11. Regla de priorización

Cada semana se puntúan todas las solicitudes entrantes con el modelo
congelado, se ordenan de mayor a menor puntaje y las primeras 120 se envían
a revisión manual. El modelo no aprueba ni rechaza: solo ordena la cola de
revisión.

## 12. Capacidad operativa

120 revisiones manuales por semana.

## 13. Supuestos económicos (fijados en la Etapa 1, copiados sin cambio)

| Concepto | Supuesto |
|---|---|
| Monto promedio del préstamo | RD\$150,000 |
| Pérdida esperada si hay mora >30d | ≈20% del monto → RD\$30,000 |
| Costo de una revisión manual | RD\$1,000 |
| Efecto de la revisión sobre un caso riesgoso | reduce la probabilidad de mora ≈35% |

Valor esperado de cada mora adicional capturada: 0.35 × RD\$30,000 ≈
RD\$10,500.

## 14. Criterio de decisión

El procedimiento **no se recomienda** si, con las mismas 120 revisiones
semanales en el período de test, no captura más moras que el baseline
operativo; ni si la mejora, traducida a dinero con los supuestos de la
sección 13, resulta demasiado pequeña para compensar el costo de construir,
mantener y monitorear el modelo.

## 15. Resultado de validación que fundamenta estas decisiones

| Método | Recall@120 (validación) | AUC |
|---|---|---|
| Trivial | 67.8% | 0.503 |
| Histórico | 70.6% | 0.534 |
| Operativo | 76.1% | 0.584 |
| **Regresión logística (congelada)** | **86.2%** | **0.679** |
| Gradient boosting | 83.0% | 0.641 |

## 16. Procedimiento de apertura del test

1. Este archivo se sube al repositorio y su commit queda registrado
   **antes** de ejecutar cualquier celda que lea el bloque de test.
2. El test (semanas 79–104) se abre una sola vez.
3. Sobre el test se reporta: la comparación de los cinco métodos con la
   métrica congelada, el análisis de errores (por sucursal, tipo de empleo
   y severidad del atraso), la lectura económica con los supuestos de la
   sección 13, y la decisión según la sección 14.
4. Después de la apertura no se ajusta ninguna decisión de este documento.
   Si se detectara un error que obligue a cambiar algo, el resultado del
   test queda invalidado y se requeriría una nueva evaluación independiente.

**Equipo:** Jose Eugenio Duran Vizcaino · Anthony Burgos · Isaac Sanchez ·
Maximo Martinez · Francisco Jose Mejia
