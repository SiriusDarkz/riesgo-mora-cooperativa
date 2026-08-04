# Auditoría de fuga y clasificación de variables

**Proyecto:** Caso 1 · Riesgo de mora en préstamos nuevos — Cooperativa Progreso del Sur
**Etapa:** 3 · Auditar
**Estado:** clasificación completa; demostración empírica pendiente de ejecución (Etapa 5)

---

## 1. Qué contiene este documento

Aquí clasificamos cada variable del conjunto de datos según el papel que puede
cumplir en el proyecto, explicamos cuáles producirían fuga de información y
por qué, y dejamos registradas las decisiones que el grupo tomó sobre los
casos que no eran obvios. La auditoría se hizo antes de programar el generador
de datos, porque la clasificación de cada variable indica cómo debe generarse:
las de fuga se generan a partir del resultado, la intervención modifica la
probabilidad de mora, y la ambigua necesitaba una definición antes de poder
existir.

## 2. El criterio

Todas las variables se juzgan con la misma pregunta del enunciado:

> ¿Esta variable existiría, con el mismo valor y significado, al momento de
> decidir?

El momento de decidir quedó definido en la Etapa 1: cuando la solicitud llega
completa, antes de aprobarla. Varias clasificaciones dependen de esa elección
— por ejemplo, `approved` es "posterior" solo porque decidimos predecir antes
de la aprobación; con otro momento de predicción, su clasificación cambiaría.

## 3. Clasificación completa

Las categorías son las del enunciado: predictora potencialmente válida,
target, identificador, posterior al momento de predicción, intervención,
ambigua, solo para auditoría y variable que debe excluirse.

| # | Variable | Clasificación | Justificación |
|---|---|---|---|
| 1 | `application_date` | Identificador (eje temporal) | Existe al decidir, pero no describe al solicitante: sirve para ordenar las solicitudes en el tiempo, agruparlas por semana y hacer la partición temporal. No se usa como predictor directo. |
| 2 | `branch_id` | Predictora potencialmente válida | La sucursal se conoce desde que entra la solicitud y no cambia. También se usa para calcular el baseline histórico. Variable sensible por territorio (ver sección 10). |
| 3 | `client_id` | Identificador | Existe al decidir, pero solo identifica a la persona. No entra al modelo; sirve para que un mismo cliente no quede repartido entre desarrollo y test. |
| 4 | `age` | Predictora potencialmente válida | Viene en el formulario, así que está disponible al decidir. Variable sensible (ver sección 10). |
| 5 | `monthly_income` | Predictora potencialmente válida | Es el ingreso que la persona declara al solicitar; ese valor existe al decidir. El ingreso verificado por la revisión aparece después, así que no es este. |
| 6 | `requested_amount` | Predictora potencialmente válida | Es el monto que el cliente pide en la solicitud. El monto aprobado puede terminar siendo otro, pero ese llega después. |
| 7 | `loan_term_months` | Predictora potencialmente válida | Plazo que el cliente solicita, disponible al decidir. El plazo final puede cambiar tras la revisión. |
| 8 | `existing_debt` | Predictora potencialmente válida | Deuda que el cliente tiene al momento de solicitar, según buró o declaración. Existe al decidir. |
| 9 | `employment_type` | Predictora potencialmente válida | Se declara en el formulario, disponible al decidir. |
| 10 | `months_in_job` | Predictora potencialmente válida | Se declara en la solicitud, disponible al decidir. Viene vacía cuando el empleo es informal, porque no hay nómina que la acredite. |
| 11 | `prior_late_payments` | Predictora potencialmente válida | Cuenta atrasos de préstamos anteriores a esta solicitud. Es historial pasado, así que existe al decidir. |
| 12 | `payment_history_score` | Ambigua → válida bajo definición | Depende de cuándo se calcule. Si el sistema lo recalcula con el tiempo, el valor guardado hoy no es el que existía al decidir, y usarlo sería fuga. Lo definimos como el score calculado solo con historial previo y congelado en la fecha de la solicitud; con esa definición sí existe al decidir. Queda vacío para clientes nuevos. |
| 13 | `debt_to_income` | Predictora potencialmente válida | Se calcula con dos datos disponibles al decidir (deuda entre ingreso). Es además la base de la regla actual de la cooperativa. |
| 14 | `manual_review` | Intervención | No es un dato del solicitante: es la acción que la cooperativa toma sobre el expediente, y cambia el resultado porque la revisión reduce la probabilidad de mora. No entra al modelo; se analiza aparte. |
| 15 | `approved` | Posterior al momento de predicción | Como predecimos antes de aprobar, en ese momento esta variable todavía no existe. Solo sirve para saber qué solicitudes llegaron a tener etiqueta (las aprobadas y desembolsadas). |
| 16 | `days_past_due_6m` | Fuga como predictor · origen del target · solo auditoría | Es el resultado de los primeros 6 meses: no existe al decidir, y usarla para predecir sería fuga. Pero también tiene que estar en el dataset, porque de ella sale el target. Por eso el enunciado la lista dos veces: prohibida como predictor, obligatoria como origen de la etiqueta. |
| 17 | `default_30d` | Target | Es lo que queremos predecir. Se usa como etiqueta al entrenar y para evaluar; nunca como predictor. |
| 18 | `collection_calls_6m` | Fuga → excluir | Cobranza llama cuando ya hay atraso: el dato aparece después del resultado y por causa de él. Al decidir no existe. Se conserva solo para la demostración de fuga. |
| 19 | `restructured_after_default` | Fuga → excluir | Solo puede existir si el préstamo ya cayó en incumplimiento. Ocurre después del resultado. |
| 20 | `legal_collection_status` | Fuga → excluir | La cobranza legal empieza meses después de iniciada la mora. Al decidir no existe. |
| 21 | `p_mora_sin_intervencion` | Solo para auditoría (diseño propio) | Columna que agregamos nosotros: guarda la probabilidad de mora antes de aplicar el efecto de la revisión. En la vida real no existe; aquí sirve para medir cuánto cambia la intervención a las etiquetas. Nunca entra al modelo. |

**Resultado:** 11 variables candidatas a predictoras (10 válidas + la ambigua
ya definida), 2 identificadores, 1 intervención, 1 posterior (`approved`),
**las 4 variables de fuga del enunciado** (3 que se excluyen por completo y
`days_past_due_6m`, que además es el origen del target), 1 target y 1 columna
de auditoría propia.

## 4. Cómo funciona la fuga en estas variables

Las variables de fuga comparten un mismo patrón: aparecen después del
resultado y por causa de él. Cobranza llama porque hubo atraso; se
reestructura porque hubo incumplimiento; el caso llega a cobranza legal
porque la mora se agravó. En el generador esto se hace explícito: por
ejemplo, `collection_calls_6m` se genera con valores altos cuando
`default_30d` es 1 y cercanos a cero cuando es 0. Por construcción, estas
variables "aciertan" casi siempre — no porque aporten información, sino
porque repiten la respuesta.

Si se usaran como predictoras pasarían dos cosas. En desarrollo, el modelo se
vería casi perfecto, y sería un espejismo. Y en producción se derrumbaría:
cuando llega una solicitud nueva, esas columnas están vacías o en cero,
porque nada de eso ha ocurrido todavía.

Para auditar datos reales, donde nadie entrega la lista de fuga, este
ejercicio deja señales de alerta útiles: nombres con `final_`, `after_` o
`status`; ventanas de tiempo que terminan después del momento de decidir;
correlaciones sospechosamente altas con el target; y variables producidas por
procesos que solo se activan cuando el resultado ya ocurrió (cobranza,
laboratorio final, nómina).

## 5. La doble aparición de days_past_due_6m

El enunciado lista `days_past_due_6m` dos veces: entre las variables mínimas y
entre las de fuga. No hay contradicción, porque las dos listas responden
preguntas distintas. La lista de mínimas dice qué columnas deben existir en el
dataset, y esta debe existir: de ella se deriva el target. La lista de fuga
dice qué columnas no pueden usarse para predecir, y esta no puede: es el
resultado mismo. Estar en los datos y entrar al modelo son cosas distintas.

## 6. La decisión sobre payment_history_score

Era la variable que no se podía clasificar sin definirla primero, porque su
nombre no dice cuándo se calcula el score, y hay dos lecturas posibles:

- **Foto congelada:** el score tal como estaba el día de la solicitud,
  calculado solo con el historial anterior a esa fecha.
- **Valor vivo:** el score como está hoy en la base de datos, recalculado
  cada vez que llega comportamiento nuevo.

La segunda lectura es fuga disfrazada, y es un problema real de los proyectos
de crédito: las bases de datos guardan el estado actual, no fotos históricas.
Si se exporta la tabla hoy, la columna trae el score de hoy para una
solicitud vieja — un score que ya incorporó los atrasos posteriores,
incluidos los de este mismo préstamo.

**Decisión del grupo:** adoptamos la foto congelada. El score se define como
calculado solo con historial previo a la solicitud y congelado en esa fecha.
Así la variable existe con ese valor al momento de decidir, y además coincide
con lo que producción vería: cuando llegue una solicitud nueva, el sistema
consultará el score de ese día. En el generador se producirá de forma
coherente con `prior_late_payments` y quedará vacía para clientes sin
historial.

## 7. Fuera del modelo, pero con trabajo en el proyecto

Que una variable no entre al modelo no significa que sobre. Cada una de estas
tiene una tarea asignada:

| Variable | Su trabajo en el proyecto |
|---|---|
| `days_past_due_6m` | De ella se deriva el target; en la Etapa 7 sirve para analizar la severidad de los errores |
| `manual_review` | Es la intervención: con ella se analiza cuánto cambió las etiquetas la política de revisión |
| `client_id` | Llave para separar clientes entre desarrollo y test (Etapa 4) |
| `application_date` | Eje de la partición temporal y de la agrupación semanal |
| `collection_calls_6m`, `restructured_after_default`, `legal_collection_status` | Protagonistas de la demostración de fuga (sección 9) |
| `p_mora_sin_intervencion` | Mide cuánto distorsiona la intervención a las etiquetas (sección 8) |

## 8. La columna propia: p_mora_sin_intervencion

Es una columna que agregamos nosotros; no viene en el enunciado. El generador
calcula primero la probabilidad de mora que el perfil merece y después, si el
expediente fue revisado, la reduce en un 35%. Esta columna guarda el primer
número, antes de la reducción.

La agregamos porque responde una pregunta que con datos reales es imposible:
qué habría pasado con un caso revisado si no lo hubieran revisado. Como
nosotros fabricamos los datos, podemos guardar esa respuesta, y con ella
medir cuánto se alejan las etiquetas observadas del riesgo real de los
perfiles revisados — exactamente la advertencia del enunciado de que la
revisión puede modificar el resultado, pero con números.

Reglas para que sea legítima: nunca entra al modelo ni a los baselines, no
participa en entrenar, seleccionar ni evaluar, y queda declarada a la vista
en esta auditoría. Su único uso es el análisis del informe final. El
enunciado lo permite: la lista de variables se titula "mínimas" (es un piso,
no un techo) y la categoría "solo para auditoría" existe en el propio menú de
clasificación.

## 9. Demostración empírica (pendiente de ejecutar)

Para mostrar que la exclusión de las variables de fuga no es un formalismo,
en la Etapa 5 haremos esta comparación usando solo la partición de desarrollo
(el test permanece cerrado):

1. Una regresión logística con las 11 predictoras válidas más las variables
   de fuga. Esperamos un desempeño casi perfecto (AUC cercano a 0.99), con la
   importancia concentrada en las variables de fuga.
2. La misma regresión sin las variables de fuga. Esperamos un desempeño
   realista para datos de crédito (AUC entre 0.70 y 0.80).

| Modelo | AUC (validación) |
|---|---|
| Con variables de fuga | *pendiente de ejecución* |
| Sin variables de fuga (limpio) | *pendiente de ejecución* |

La brecha entre las dos filas mide cuánto ayuda "conocer la respuesta". El
primer modelo no es mejor: es tramposo, y en producción sería inservible.

## 10. Notas de equidad

`age` y `branch_id` existen al decidir, así que son válidas técnicamente,
pero son sensibles: la edad es una característica protegida en muchos marcos,
y la sucursal codifica territorio, y con él condiciones socioeconómicas. El
grupo decide mantenerlas como candidatas con dos compromisos: el análisis de
errores de la Etapa 7 se desagregará por sucursal para detectar
disparidades, y el informe ejecutivo discutirá las implicaciones de usarlas —
recordando que la acción que dispara el modelo es una verificación adicional,
no un rechazo.

## 11. Preguntas del enunciado que esta auditoría deja respondidas

- **P4 · ¿Qué variables estarán disponibles en producción?** Las 11
  candidatas a predictoras de la tabla, más los identificadores
  (disponibles, aunque no describen riesgo).
- **P5 · ¿Qué variables contienen información futura?** Las 4 de fuga del
  enunciado (`days_past_due_6m`, `collection_calls_6m`,
  `restructured_after_default`, `legal_collection_status`), `approved`
  (posterior al momento elegido) y el propio target `default_30d`.
- **P6 · ¿Qué intervención puede modificar el target?** La revisión manual
  (`manual_review`), que en la simulación reduce la probabilidad de mora del
  caso revisado en un 35% aproximadamente.
