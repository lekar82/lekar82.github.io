<style> div { font-family:"Arial"; font-size: 18px; } </style>

# WORK IN PROGRESS....

# Detección de changepoints 1: método bayesiano

## Indice:

* [Objetivo](#Objetivo)
* [Librerías necesarias](#Librerías-necesarias)
* [Aplicación en datos simulados](#Aplicación-en-datos-simulados)
* [Aplicación en datos contagios Covid de la CCAA de Madrid](#Aplicación-en-datos-contagios-Covid-de-la-CCAA-de-Madrid)

## Objetivo

En este post vamos a analizar como detectar Changepoint en nuestros datos mediante el método Bayesiano. 

Los métodos bayesianos para detectar cambios en la tendecia de nuestros datos pueden ser offline u online. La principal diferencia entre los dos métodos reside en la capacidad de poder procesar los datos según lleguen, es decir, en "tiempo real" (online) o si el proceso va a procesar todos los datos de una vez (offline). 

Uno u otro método se usará dependiendo del objetivo del análisis. Los métodos online suelen estar enfocados a la detección de roturas en la serie de datos  tan pronto como sea posible, intentando minimizar el número de falsos positivos. Por otro lado, los métodos offlines suelen tener como objetivo detectar todas los cambios de tendencia originados en una serie, para así definir los criterios de sensibilidad (detección de cambios de tendencia reales) y especefeceidad (evitando falsos positivos).

## Librerias

```python
import pandas as pd
import numpy as np
import matplotlib.cm as cm
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

Como hemos comentado al comienzo del post, el método offline se suele usar para definir qué consideramos chaintpoint. Viendo la gráfica, podríamos definirlo como auquellos puntos cuyos puntos tienen una intensindad superior a 0.8:

```python
test = pd.DataFrame(np.exp(Pcp).sum(0))
test.columns = ['valores']
breaks = test[test['valores'] >= 0.8].index
breaks
```
    Int64Index([77, 78, 231, 372, 373, 490, 629, 630], dtype='int64')



Vamos a aplicar el método **Bayesiano online** para ver las diferencias:

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

## Aplicación en datos contagios Covid de la CCAA de Madrid

Hemos extraido los datos de la página de la Comunidad de Madrid, en el siguiente enlace:  
  
https://www.comunidad.madrid/sites/default/files/doc/sanidad/210111_cam_covid19.pdf

Una vez descargado el archivo, extraemos la información usando read_pdf y limpiamos los datos:

```python
def extr_date(x):
    datos = []
    datos_final = []

    df = read_pdf(x, pages = "all", multiple_tables = True)
    for num_col in range(7):
        df2 = df[num_col].dropna(axis=0, how='all')   
        try:
            # la primera celda puede ser dato, 'Fecha' o 'Notificación', según la hoja en la que estemos
            # queremos únicamente los datos por lo que empezaremos a guardar a partir de la primera celda que tenga datos.
            a = str(df2.columns[0])
            df2 = df2.loc[(df2[a] != 'Fecha') & (df2[a] != 'Notificación') ].dropna(axis=1, how='all') 
            df2 = df2.iloc[1:].dropna(axis=1, how='all')

            #Las hojas de datos tienen que tener 3, 6 u 9 columnas de datos:
            
            if((len(df2.columns) == 3) |(len(df2.columns) == 6) | (len(df2.columns) == 9 )): 
                # entre los datos se ha observado que se han exportado NaN, posiblemente debido a espacios en blanco
                # guardamos los datos que no son nan en una lista
                for j in range(len(df2)):
                    for i in df2.columns:
                        if str(df2[i].iloc[j]) != 'nan':
                            datos.append(df2[i].iloc[j])                                  

        except IndexError:
            next
   
   # eliminamos todo el texto  
    for item in datos:
        if item not in ['Fecha', 'Total', 'Notificación','Diario', 'Acumulado' ]:
            #print(item)
            datos_final.append(item)
            #len(datos_final)
    
    # por último, sabemos que nuestros tenemos 3 datos para cada row, hacemos un resaphe de la lista de manera que 
    # nos coloque nuestros datos en una tabla de n_días * 3 columnas
    #return(datos_final)
    datos_np = np.array(datos_final).reshape(-1, 3)
    
    datos_df = pd.DataFrame(datos_np,columns=['Fecha', 'dato_diario', 'acumulado'])
    datos_df['fecha_informe'] = file
    datos_df = datos_df[(datos_df['Fecha'] != 'Notificación') & (datos_df['Fecha'] != 'Fecha')] 
   ```     
    return(datos_df)  


```python
file = 'datos/Informe de situación 11 de enero 2021.pdf'

data_final_ultimo = extr_date(file)

data_final_ultimo["dato_diario"] = pd.to_numeric(data_final_ultimo["dato_diario"])
data_final_ultimo["acumulado"] = pd.to_numeric(data_final_ultimo["acumulado"])
data_final_ultimo["Fecha"] = data_final_ultimo["Fecha"].str.replace(' ', '')
data_final_ultimo["Fecha"] = pd.to_datetime(data_final_ultimo["Fecha"], format="%d/%m/%Y")

data_final_ultimo = data_final_ultimo.sort_values(by='Fecha')
data_covid = np.array(data_final_ultimo["dato_diario"])

```


```python
plt.figure(figsize=(60,20))
plt.plot(data_covid,color = 'darkblue',)

plt.title('Evolución contagios por Covid en la CCAA de Madrid', fontsize = 60)

plt.xlabel('indice', fontsize=50)
plt.ylabel('Nuestros valores', fontsize=50)
```




    Text(0, 0.5, 'Nuestros valores')


![png](/images/Chaingpoints_bayes/output_15_1.png)
    
```python
Q, P, Pcp = offcd.offline_changepoint_detection(data_covid, 
                                                partial(offcd.const_prior, l=(len(data_covid)+1)), 
                                                offcd.gaussian_obs_log_likelihood, truncate=-40)
```


```python
fig, ax = plt.subplots(figsize=[18, 12])
ax = fig.add_subplot(2, 1, 1)
ax.plot(data_covid[:])
ax = fig.add_subplot(2, 1, 2, sharex=ax)
ax.plot(np.exp(Pcp).sum(0))
```


![png](/images/Chaingpoints_bayes/output_18_1.png)


```python
R, maxes = oncd.online_changepoint_detection(data_covid, 
                                             partial(oncd.constant_hazard, 250), 
                                             oncd.StudentT(0.1, .01, 1, 0))
```


```python
fig, ax = plt.subplots(figsize=[18, 16])
ax = fig.add_subplot(3, 1, 1)
ax.plot(data_covid)
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



![png](/images/Chaingpoints_bayes/output_23_2.png)
