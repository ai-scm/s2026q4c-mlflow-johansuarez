# Práctica de MLflow - Johan Suarez

## Desarrollo de la práctica

### 1. Model Serving sin Model Registry

Se clonó el repositorio del tutorial y se construyeron las imágenes de Docker. Se entrenó un modelo Random Forest de Scikit-learn con el dataset Breast Cancer, separando los datos en entrenamiento (80 %) y prueba (20 %) y utilizando `StandardScaler` para estandarizar las características.

```powershell
git clone -c core.autocrlf=false https://github.com/mtpatter/mlflow-tutorial.git
cd mlflow-tutorial
docker compose -f docker-compose-no-registry.yml build
docker compose -f docker-compose-no-registry.yml up
```

El modelo se almacenó localmente en `clf-model` y se desplegó como una API REST mediante MLflow. Se enviaron los datos de `test.csv` al endpoint `/invocations`, obteniendo las probabilidades de clasificación en formato JSON.

**Resultados:** el entrenamiento generó `clf-model` y `test.csv`. El servicio `trainmodel` finalizó con código 0 y `servemodel` quedó disponible en el puerto 1234.

```powershell
curl.exe -i http://localhost:1234/ping
curl.exe http://localhost:1234/invocations -H "Content-Type: text/csv" --data-binary "@test.csv"
```

El servidor respondió correctamente y devolvió 114 predicciones. Ejemplo de las primeras cinco:

```json
{"predictions": [0.5074, 0.0, 0.9837, 0.986, 0.9913]}
```

Cada valor representa la probabilidad estimada de pertenecer a la clase benigna para un registro del conjunto de prueba.

### 2. Model Serving con Model Registry

Mediante MLflow Tracking se registraron los entrenamientos y sus artefactos, mientras que Model Registry permitió registrar dos versiones de `clf-model` y asignar el alias `Staging` a la segunda.

```powershell
docker compose -f docker-compose-no-registry.yml down
docker compose -f docker-compose.yml up --build
```

Se accedió a la interfaz de MLflow en `http://localhost:8000`. Posteriormente, se aumentó el número de árboles de 100 a 200 en `clf-train-registry.py` y se ejecutó un segundo entrenamiento:

```powershell
docker compose -f docker-compose.yml run --rm --no-deps trainmodel
```

**Resultados:** se registraron las ejecuciones `painted-penguin-248` y `luminous-kite-179`, correspondientes a las versiones 1 y 2. MLflow confirmó la creación de la segunda versión y la asignación del alias:

```text
Created version '2' of model 'clf-model'.
Model status: READY
Aliases for model 'clf-model' version 2: ['Staging']
```

La comparación de las métricas obtenidas fue:

| Métrica | Versión 1 (100 árboles) | Versión 2 (200 árboles) |
| --- | ---: | ---: |
| Accuracy de entrenamiento | 99,78 % | 99,56 % |
| Accuracy de prueba | 94,74 % | 96,49 % |

La segunda versión obtuvo una mejora de 1,75 puntos porcentuales en los datos de prueba, sin que esto garantice un mejor rendimiento sobre cualquier conjunto de datos.

Se volvió a consumir el modelo mediante el mismo endpoint `/invocations`, obteniendo nuevas probabilidades:

| Registro | Versión 1 | Versión 2 |
| --- | ---: | ---: |
| 1 | 0,5074 | 0,4943 |
| 3 | 0,9837 | 0,9869 |
| 63 | 0,6422 | 0,6906 |
| 86 | 0,4790 | 0,5376 |

Esto demuestra que se puede actualizar el modelo sin modificar la petición HTTP del cliente. El servidor debe cargar la nueva versión para aplicar el cambio de alias.

### 3. Gestión de dependencias

Se presentaron problemas con los saltos de línea de los scripts y una incompatibilidad entre MLflow 2.16.2 y SQLAlchemy. Se solucionaron conservando el formato LF al clonar el repositorio y agregando la siguiente dependencia a `requirements.txt`:

```text
SQLAlchemy==2.0.35
```

Después se reconstruyeron las imágenes de Docker. Esto evidencia la importancia de controlar las dependencias y garantizar la reproducibilidad de los entornos en MLOps.

## Relación con el Data Lifecycle

| Etapa | Relación con la práctica |
| --- | --- |
| Ingesta de datos | Se cargó el dataset Breast Cancer desde Scikit-learn como fuente de información. |
| Montaje de datos | Se organizaron los datos y se separaron en conjuntos de entrenamiento y prueba. |
| Limpieza de datos | Se estandarizaron las características mediante `StandardScaler`, como parte de la preparación de los datos. No se realizó una limpieza explícita de duplicados o valores faltantes. |
| Análisis y visualización de datos | Se entrenó el modelo, se generaron predicciones y se compararon las métricas de ambas versiones mediante la interfaz de MLflow. |
| Archivado de datos | Se conservaron los modelos y artefactos de los entrenamientos en MLflow, facilitando su consulta y reutilización. No se implementó una política de archivado de datos a largo plazo. |

## Conclusión

MLflow complementa el Data Lifecycle al facilitar el seguimiento, la trazabilidad, el versionado y el despliegue de modelos construidos a partir de los datos. Junto con Docker, permite reproducir los entrenamientos y actualizar los modelos sin modificar la interfaz utilizada por las aplicaciones que consumen las predicciones.
