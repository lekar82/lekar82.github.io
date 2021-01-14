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

import ruptures as rpt
from tabula.io import read_pdf
from PyPDF2 import PdfFileWriter, PdfFileReader
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

Como podemos observar, 

