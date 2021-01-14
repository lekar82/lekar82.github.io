<style> div { font-family:"Arial"; font-size: 18px; } </style>

# WORK IN PROGRESS....

# Detección de changepoints 1: método bayesiano

## Indice:

* [Objetivo](#Objetivo)
* [Librerías necesarias](#Librerías-necesarias)
* [Aplicación en datos simulados](#Aplicación-en-datos-simulados)

## Objetivo

En este post vamos a analizar como detectar Changepoint en nuestros datos mediante el método Bayesiano. 

Los métodos bayesianos para detectar cambios en la tendecia de nuestros datos pueden ser offline u online. La principal diferencia entre los dos métodos reside en la capacidad de poder procesar los datos según lleguen, es decir, en "tiempo real" (online) o si el proceso va a procesar todos los datos de una vez (offline). 

Uno u otro método se usará dependiendo del objetivo del análisis. Los métodos online suelen estar enfocados a la detección de roturas en la serie de datos  tan pronto como sea posible, intentando minimizar el número de falsos positivos. Por otro lado, los métodos offlines suelen tener como objetivo detectar todas los cambios de tendencia originados en una serie, para así definir los criterios de sensibilidad (detección de cambios de tendencia reales) y especefeceidad (evitando falsos positivos).

## Librerias

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from datetime import datetime

import io

import math

import cProfile
import bayesian_changepoint_detection.offline_changepoint_detection as offcd
import bayesian_changepoint_detection.online_changepoint_detection as oncd
from functools import partial

```

## Aplicación en datos simulados  

Vamos a empezar creando una serie aleatoria de datos de datos. En cada tramo o partición los datos serán generados por una distribución normal, en la que tanto el número de elementos de la muestra como los parámetros (media y varianza) se calcularán de una manera aleatoria.

*Definimos nuestra función para generar series:*

```python
def generate_normal_time_series(num, minl=50, maxl=1000):
    data = np.array([], dtype=np.float64)
    partition = np.random.randint(minl, maxl, num)
    print(partition)
    for p in partition:
        mean = np.random.randn()*10
        var = np.random.randn()*1
        if var < 0:
            var = var * -1
        tdata = np.random.normal(mean, var, p)
        data = np.concatenate((data, tdata))
    return data
```
Donde *num* es el número de particiones que queremos que tenga nuestra serie. y *minl* y *maxl* es el número múnimo y máximo de datos que tendremos en cada una de esas particiones.


Generamos una serie con 7 puntos de cambio, cada una de las cuales con entre 50 y 200 valores

```python
datos_ficticios = generate_normal_time_series(7, 50, 200)
```
Nuestra función ha creado las particiones de la siguiente manera:

    [ 70 182 113 162 161 161  88]
    
Graficamos nuestros datos para ver que pinta tienen:

```python
plt.figure(figsize=(60,20))
plt.plot(datos_ficticios,color = 'darkblue',)

plt.title('Serie datos ficticios', fontsize = 60)

plt.xlabel('indice', fontsize=50)
plt.ylabel('Nuestros valores', fontsize=50)
```

![png](/images/Chaingpoints_bayes/output_5_1.png)

Como podemos observar, aunque hemos definido nuetro paramentro num igual a 7, visualmente da la impresión de que tenemos 6 cambios.

A continuación, vamos a detectar los puntos de cambio usando el método **Bayesiano offline**:

```python
Q, P, Pcp = offcd.offline_changepoint_detection(datos_ficticios, 
                                                partial(offcd.const_prior, l=(len(datos_ficticios)+1)), 
                                                offcd.gaussian_obs_log_likelihood, truncate=-40)
```

Graficamos los resultados:

```python
fig, ax = plt.subplots(figsize=[18, 16])
ax = fig.add_subplot(2, 1, 1)
ax.plot(datos_ficticios[:])
ax = fig.add_subplot(2, 1, 2, sharex=ax)
ax.plot(np.exp(Pcp).sum(0))
```
    
![png](/images/Chaingpoints_bayes/output_7_1.png)

Obsevamos en la parte de arriba la serie, y en la parte de abajo los cambios detectados y su intensidad. 

Si, por ejemplo, definimos los cambios como aquellos con una intensidad mayor a 0.8, podemos obtener los puntos de cambio como sigue:

```python
test = pd.DataFrame(np.exp(Pcp).sum(0))
test.columns = ['valores']
breaks = test[test['valores'] >= 0.8].index
breaks
```
    Int64Index([77, 78, 231, 372, 373, 490, 629, 630], dtype='int64')

Aunque en este ejemplo no tiene sentido ya que disponemos ya de todos los datos, vamos a aplicar el método **Bayesiano online** para ver las diferencias:



```python
R, maxes = oncd.online_changepoint_detection(datos_ficticios, 
                                             partial(oncd.constant_hazard, 250), 
                                             oncd.StudentT(0.1, .01, 1, 0))
```


```python
fig, ax = plt.subplots(figsize=[18, 16])
ax = fig.add_subplot(3, 1, 1)
ax.plot(datos_ficticios)
ax = fig.add_subplot(3, 1, 2, sharex=ax)
sparsity = 5  # only plot every fifth data for faster display
ax.pcolor(np.array(range(0, len(R[:,0]), sparsity)), 
          np.array(range(0, len(R[:,0]), sparsity)), 
          -np.log(R[0:-1:sparsity, 0:-1:sparsity]), 
          cmap=cm.Greys, vmin=0, vmax=30)
ax = fig.add_subplot(3, 1, 3, sharex=ax)
Nw=10;
ax.plot(R[Nw,Nw:-1])
```

    
![png](/images/Chaingpoints_bayes/output_38_2.png)


