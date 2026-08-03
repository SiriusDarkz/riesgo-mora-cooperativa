# Caso 1 · Riesgo de mora en préstamos nuevos

**Cooperativa Progreso del Sur**

Proyecto de la Semana 1 de la asignatura *Selección y Validación de Modelos*: del problema empresarial al protocolo reproducible.

| | |
|---|---|
| **Profesor** | Dr. Edian Franco |
| **Caso asignado** | Caso 1 · Cooperativa de ahorro y crédito |
| **Objetivo** | Priorizar hasta 120 solicitudes semanales para revisión manual adicional |

## Integrantes

| Nombre | Responsabilidad |
|---|---|
| Jose Eugenio Duran Vizcaino | *(por definir)* |
| Anthony Burgos | *(por definir)* |
| Isaac Sanchez | *(por definir)* |
| Maximo Martinez | *(por definir)* |
| Francisco Jose Mejia | *(por definir)* |

## Descripción del problema

La Cooperativa Progreso del Sur (sucursales en Santo Domingo, San Cristóbal, Baní y Azua) desea identificar cuáles solicitudes nuevas de préstamo podrían presentar una mora superior a 30 días durante sus primeros seis meses. El equipo de riesgo solo puede revisar en detalle **120 solicitudes por semana**, por lo que el procedimiento **prioriza** expedientes para verificación adicional; no rechaza solicitantes automáticamente.

Todo el proyecto utiliza **datos sintéticos** generados en Python con semilla fija.

## Estructura del repositorio

```
├── README.md                      # Este archivo
├── requirements.txt               # Librerías y versiones
├── data/
│   ├── solicitudes_sinteticas.csv # Datos generados (semilla fija)
│   └── diccionario_variables.md   # Descripción de cada variable
├── notebooks/
│   └── caso1_cooperativa.ipynb    # Notebook completo (Etapas 1–8)
└── reports/
    ├── auditoria_fuga.md          # Clasificación de variables
    ├── frozen_protocol.md         # Protocolo congelado antes de test
    └── informe_ejecutivo.pdf      # Informe (máx. 3 páginas)
```

## Cómo reproducir

**Opción A · Google Colab**

1. Abrir `notebooks/caso1_cooperativa.ipynb` en Google Colab.
2. Ejecutar todas las celdas en orden (Entorno de ejecución → Ejecutar todo).

**Opción B · Local**

```bash
pip install -r requirements.txt
jupyter notebook notebooks/caso1_cooperativa.ipynb
```

La semilla aleatoria está fijada (`seed = 42`), por lo que ejecutar el notebook completo reproduce exactamente los mismos datos, particiones y resultados.

## Entregables

| Entregable | Ubicación |
|---|---|
| Repositorio reproducible | Este repositorio |
| Notebook completo | `notebooks/caso1_cooperativa.ipynb` |
| Datos sintéticos y diccionario | `data/` |
| Auditoría de fuga | `reports/auditoria_fuga.md` |
| Protocolo congelado | `reports/frozen_protocol.md` |
| Informe ejecutivo | `reports/informe_ejecutivo.pdf` |
| Evidencia individual | Historial de commits de cada integrante |

## Reglas del protocolo

- La partición **no** es aleatoria 80/20: se utiliza una partición temporal con separación de clientes entre desarrollo y test (justificación en el notebook).
- El protocolo se congela en `reports/frozen_protocol.md` **antes** de abrir el conjunto de test.
- El test se abre **una sola vez** y no se utiliza para seleccionar ni modificar el modelo.

## Estado del proyecto

- [x] Etapa 1 · Formulación
- [ ] Etapa 2 · Generación de datos sintéticos
- [ ] Etapa 3 · Auditoría de variables
- [ ] Etapa 4 · Diseño de la evaluación
- [ ] Etapa 5 · Baselines y modelos
- [ ] Etapa 6 · Protocolo congelado
- [ ] Etapa 7 · Evaluación en test
- [ ] Etapa 8 · Informe y recomendación
