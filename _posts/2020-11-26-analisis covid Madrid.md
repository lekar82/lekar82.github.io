<style> div { font-family:"Arial"; font-size: 18px; } </style>

# Evolución del Covid-19 en la CCAA de Madrid

## Indice:

* [Objetivo](#Objetivo)
* [Librerías necesarias](#Librerías-necesarias)
* [Datos de contagios](#Datos-de-contagios)


## Objetivo

Se pretende analizar la evolución del Covid-en la Comunidad Autónoma de Madrid. Tendremos en cuenta:  
* Número de contagios reportados.  
* Número de pruebas diagnosticas reportadas.  

Las fuentes que vamos a usar para realizar el estudio son:

* [Página oficial de la comunidad de Madrid](https://www.comunidad.madrid/servicios/salud/2019-nuevo-coronavirus#situacion-epidemiologica-actual): donde tenemos datos diarios del número de contagios.
* [Página oficial del Ministerio de Salud del Gobierno de España](https://www.mscbs.gob.es/profesionales/saludPublica/ccayes/alertasActual/nCov/): semanalmente publican en sus notas de presa los datos de pruebas realizadas.

## Librerías necesarias

Importamos librerías necesarias

```python
import pandas as pd
import numpy as np
import math
import matplotlib.pyplot as plt

from bs4 import BeautifulSoup
import urllib.request
import pandas as pd
import re

#!pip install tabula
#!pip install pyPdf2
#!pip install StringIO
import tabula
from PyPDF2 import PdfFileWriter, PdfFileReader
import io
```
## Datos de contagios

Como hemos comentado, la Comunidad de Madrid en su página oficial publica los datos diaríos de contagios oficiales. Los informes están en formato pdf, por lo que haremos uso de la librería *Tabula* para extraer la información.

A la fecha de realización de este análisis, el último pdf disponible es el correspondiente al 23 de Noviembre de 2020. Mediante el uso de la librería urllib accederemos a la página donde está contenido dicho pdf, y luego con las librerías io y pyPdf2 guardaremos el pdf en nuestro ordenador.

```python
url = "https://www.comunidad.madrid/sites/default/files/doc/sanidad/201124_cam_covid19.pdf"
writer = PdfFileWriter()

remoteFile =  urllib.request.urlopen((url)).read()
memoryFile = io.BytesIO(remoteFile)
pdfFile = PdfFileReader(memoryFile)
for pageNum in range(pdfFile.getNumPages()):
    currentPage = pdfFile.getPage(pageNum)
    writer.addPage(currentPage)

name_path = "datos/Informe de situación 23 de septiembre 2020.pdf"
links_contagios.append(name_path)
        
outputStream = open(name_path,"wb")
writer.write(outputStream)
outputStream.close()
```
A continuación, haciendo uso de la librería Tabula, extraeremos la información contenida en las tablas del pdf. Hay que tener paciencia a la hora de leer los datos de pdfs, requiere su tiempo ir limpiando la tabla hasta conseguir lo que quieres.

```python
def extr_date(x):
    datos = []
    df = tabula.read_pdf(x, pages = "all", multiple_tables = True)
    for num_col in range(9):
        df2 = df[num_col].dropna(axis=1, how='all')   
        try:
            # la primera celda de nuestro dataframe es una fecha.
            # la primera celda puede ser dato, 'Fecha' o 'Notificación', según la hoja en la que estemos
            # queremos únicamente los datos por lo que empezaremos a guardar a partir de la primera celda que tenga datos.
            a = str(df2.columns[0])
            df2 = df2.loc[(df2[a] != 'Fecha') & (df2[a] != 'Notificación')].dropna(axis=1, how='all') 
            #print(num_col, len(df2.columns))
            #los df con 9 o más columnas son las que contienen nuestros datos:
            if(len(df2.columns) >= 9): 
                columnas = list(df2)
                # entre los datos se ha observado que se han exportado NaN, posiblemente debido a espacios en blanco
                # guardamos los datos que no son nan en una lista
                for j in range(len(df2)):
                    for i in columnas:
                        if str(df2[i].iloc[j]) != 'nan':
                            datos.append(df2[i].iloc[j])
            
        except IndexError:
            next
        
    # por último, sabemos que nuestros tenemos 3 datos para cada row, hacemos un resaphe de la lista de manera que 
    # nos coloque nuestros datos en una tabla de n_días * 3 columnas
    datos_np = np.array(datos).reshape(-1, 3)
    datos_df = pd.DataFrame(datos_np,columns=['Fecha', 'dato_diario', 'acumulado'])
    datos_df['fecha_informe'] = file
    datos_df = datos_df[(datos_df['Fecha'] != 'Notificación') & (datos_df['Fecha'] != 'Fecha')] 
        
   return(datos_df)
   ```
Extraemos los datos 

```python
file = 'datos/Informe de situación 24 de noviembre 2020.pdf'

data_final_ultimo = extr_date(file)
```
Analizamos las variables de nuestro dataframe.
* Fecha: 
* dato_diario: número de contagios del día en cuestión. Hay que resaltar que la CCAA de Madrid considera que el día de contagio es el día en el que se realizó la prueba, por ejemplo, si una persona se hizo la prueba el 3 de Abril y el resultado se lo dieron el 8 de Abril, el caso positivo se sumará a los datos del 3 de Abril, no el día 8. Esto hay que tenerlo en cuenta a la hora de hacer los análisis porque los datos de contagios de los últimos días no son definitivos, en posteriores informes podría cambiar.
* acumulado: Contagios confirmados acumulados hasta la fecha
* fecha_informe: fecha en la que la CCAA generó el informe. 

```python
data_final_ultimo.info()
```

    <class 'pandas.core.frame.DataFrame'>
    Int64Index: 273 entries, 0 to 272
    Data columns (total 4 columns):
     #   Column         Non-Null Count  Dtype 
    ---  ------         --------------  ----- 
     0   Fecha          273 non-null    object
     1   dato_diario    273 non-null    object
     2   acumulado      273 non-null    object
     3   fecha_informe  273 non-null    object
    dtypes: object(4)
    memory usage: 10.7+ KB
    
Como podemos observar, todas las variables son de tipo string ahora mismo. Cambiamos los tipos:

```python
data_final_ultimo["dato_diario"] = pd.to_numeric(data_final_ultimo["dato_diario"])
data_final_ultimo["acumulado"] = pd.to_numeric(data_final_ultimo["acumulado"])
data_final_ultimo["Fecha"] = data_final_ultimo["Fecha"].str.replace(' ', '')
data_final_ultimo["Fecha"] = pd.to_datetime(data_final_ultimo["Fecha"], format="%d/%m/%Y")
```

```python
data_final_ultimo.info()
```

    <class 'pandas.core.frame.DataFrame'>
    Int64Index: 273 entries, 0 to 272
    Data columns (total 4 columns):
     #   Column         Non-Null Count  Dtype         
    ---  ------         --------------  -----         
     0   Fecha          273 non-null    datetime64[ns]
     1   dato_diario    273 non-null    float64       
     2   acumulado      273 non-null    float64       
     3   fecha_informe  273 non-null    object        
    dtypes: datetime64[ns](1), float64(2), object(1)
    memory usage: 10.7+ KB
    
```python
data_final_ultimo.head()
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Fecha</th>
      <th>dato_diario</th>
      <th>acumulado</th>
      <th>fecha_informe</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2020-02-25</td>
      <td>2.0</td>
      <td>2.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2020-04-12</td>
      <td>408.0</td>
      <td>56564.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2020-05-29</td>
      <td>522.0</td>
      <td>84376.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2020-02-26</td>
      <td>5.0</td>
      <td>7.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2020-04-13</td>
      <td>1209.0</td>
      <td>57773.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
    </tr>
  </tbody>
</table>
</div>

```python
datos_df = data_final_ultimo.sort_values(by='Fecha')

plt.figure(figsize=(60,20))
plt.bar(data_final_ultimo['Fecha'], data_final_ultimo['dato_diario'],color = 'darkblue',)

plt.title('Evolución número contagios Covid-19 en la CCAA de Madrid', fontsize = 60)

plt.xticks(rotation= 45)
plt.xticks(fontsize=30)
plt.yticks(fontsize=30)

plt.xlabel('Fecha', fontsize=50)
plt.ylabel('Numero de contagios', fontsize=50)
```

![png](/images/covid_madrid/output_18_1.png)

Vamos a añadir al gráfici una linea suavizada que nos permita ver mejor la tendencia de nuestros datos. Para ello usamos la función rolling. Hay que acordarse de ordenar previamente los datos para que el resultado sea el deseado

```python
data_final_ultimo = data_final_ultimo.sort_values(by='Fecha')
data_final_ultimo['MovingAgerage'] = data_final_ultimo['dato_diario'].rolling(min_periods=7, window=7).mean()
```
Y ahora repetimos el gráfico:

```python
datos_df = data_final_ultimo.sort_values(by='Fecha')
plt.style.use(['classic'])

plt.figure(figsize=(60,20))

plt.bar(data_final_ultimo['Fecha'], data_final_ultimo['dato_diario'], 
        color = 'darkblue',
       label='casos diarios')

plt.plot(data_final_ultimo['Fecha'], data_final_ultimo['MovingAgerage'], 
         color='red', 
         linewidth=8, 
         label='media movil 7 días')

plt.title('Evolución número contagios Covid-19 en la CCAA de Madrid', fontsize = 60)

plt.xticks(rotation= 45 )
plt.xticks(fontsize=30)
plt.yticks(fontsize=30)

plt.xlabel('Fecha', fontsize=50)
plt.ylabel('Numero de contagios', fontsize=50)

plt.legend(fontsize=40)
```

![png](/images/covid_madrid/output_21_1.png)

## Pruebas diagnosticas realizadas.

La información de las pruebas diagnósticas la podemos encontraer el la página del Ministerio de Sanidad del Gobierno de España. En este caso, el último informe no diene los informes anteriores, por lo que tendremos que consultar todos y cada uno de los informes. Tampoco en este caso disponemos de datos diarios, los datos que nos ofrece el ministerio son agregaciones semanales.

Los pdfs no son fáciles de localizar en la página web, afortunadamente todos los enlaces a los pdfs tienen una misma estructura, por lo que hemos podido realizar un script que nos permita descargar todos. Desafortunadamente, no todos los pdfs tienen la misma estructura por lo que no ha sido posible crear una función para extraer toda la información, de hecho, en algunos de los pdf los datos están en formato imagen en vez de tabla. Por ello, y dado que no son muchos pdfs, se ha creado a mano un excel con la información.

Primero creamos una lista de los posibles links donde podemos encontrar pdf

```python
#creamos una serie de pandas que contenga todas las posibles fechas del año 2020:
date_rng = pd.date_range(start='2020/01/01', end='2020/12/31', freq='D')

#y les cambiamos el formato para que coincidan con la fecha que aparece en los links:
import string
cambio_date_rng = lambda fecha: fecha.strftime('%d_%m_%Y')
date_rng_str = cambio_date_rng(date_rng)

#creamos la lista con todos los posibles links:
lista_links_pruebas = []
for i in range(len(date_rng_str)):   
    link = 'https://www.mscbs.gob.es/profesionales/saludPublica/ccayes/alertasActual/nCov/documentos/COVID-19_pruebas_diagnosticas_' + str(date_rng_str[i]) + '.pdf'
    lista_links_pruebas.append([link, date_rng_str[i]])
```
Una vez tenemos una lista con todos los posibles links hacemos un código que descargue el pdf en aquellos links que efectivamente tienen pdf:
```python
links_contagios = []

for i in range(len(lista_links_pruebas)):
    url = lista_links_pruebas[i][0]
    writer = PdfFileWriter()
    try:
        remoteFile =  urllib.request.urlopen((url)).read()
        #memoryFile = io.StringIO(remoteFile)
        memoryFile = io.BytesIO(remoteFile)
        pdfFile = PdfFileReader(memoryFile)
        for pageNum in range(pdfFile.getNumPages()):
            currentPage = pdfFile.getPage(pageNum)
            #currentPage.mergePage(watermark.getPage(0))
            writer.addPage(currentPage)

        name_path = "datos/pruebas_diganosticas" +  str(lista_links_pruebas[i][1]) + ".pdf"
        links_contagios.append(name_path)
        
        outputStream = open(name_path,"wb")
        writer.write(outputStream)
        outputStream.close()
    except  urllib.request.HTTPError:
        next
```
 Manualmente creamos el excel con la información que necesitamos y a continuación importamos los datos y echamos un ojo a ver que pinta tienen nuestros datos

```python
import pandas as pd

df_diagnosticas = pd.read_excel('datos/PRUEBAS.xls')
df_diagnosticas = df_diagnosticas.fillna(0) # los na suponemos que son semanas sin pruebas 
df_diagnosticas.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 29 entries, 0 to 28
    Data columns (total 7 columns):
     #   Column            Non-Null Count  Dtype         
    ---  ------            --------------  -----         
     0   desde             29 non-null     datetime64[ns]
     1   hasta             29 non-null     datetime64[ns]
     2   PCR               29 non-null     int64         
     3   TEST_RAPIDO_AC    29 non-null     int64         
     4   PRAg              29 non-null     int64         
     5   OTRAS_PRUEBAS_AC  29 non-null     int64         
     6   TOTAL             29 non-null     int64         
    dtypes: datetime64[ns](2), int64(5)
    memory usage: 1.7 KB
    
Y finalmente realizamos un gráfico para ver visualmente la evolución de los datos:

```python
import matplotlib.pyplot as plt

plt.style.use(['classic'])

width = 3      # the width of the bars: can also be len(x) sequence
fig, ax = plt.subplots(1, figsize=(24, 10))

ax.bar(df_diagnosticas["hasta"], df_diagnosticas['PCR'], width, label='PCR', color = '#1D2F6F')
ax.bar(df_diagnosticas["hasta"], df_diagnosticas['TEST_RAPIDO_AC'], width, bottom=df_diagnosticas['PCR'],label='Test rápido AC', color = '#8390FA')
ax.bar(df_diagnosticas["hasta"], df_diagnosticas['PRAg'], width, bottom=df_diagnosticas['PCR'], label='PRAg', color = '#6EAF46')
ax.bar(df_diagnosticas["hasta"], df_diagnosticas['OTRAS_PRUEBAS_AC'], width, bottom=df_diagnosticas['PCR'],label='Oras pruebas AC', color = '#FAC748')

#añadimos los totales:
for xpos, ypos, yval in zip(df_diagnosticas["hasta"], df_diagnosticas['TOTAL'], df_diagnosticas['TOTAL']):
    plt.text(xpos, ypos, "%.0f"%yval, horizontalalignment='center',  verticalalignment='bottom',
            bbox=dict(facecolor='gray', alpha=0.5))

ax.set_ylabel('ruebas')
ax.set_title('Pruebas semanales por tipo de prueba', fontsize=24)
ax.legend(loc='upper left')

ax.set_ylim(0, 250000)

plt.show()
```


![png](/images/covid_madrid/output_30_0.png)


## Análisis conjunto del número de contagios y las pruebas realizadas

Antes de unir los dos dataframe tenemos que tener en cuenta que tenemos datos de contagios diarios vs pruebas semanales. Por ello, lo primero que haremos será agrupar los datos de contagios:

```python
data_final_ultimo = data_final_ultimo.sort_values(by='Fecha')

data_final_ultimo['acumulado semanal'] = data_final_ultimo['dato_diario'].rolling(min_periods=7, window=7).sum()
```
Como podemos ver, en la fecha = '2020-03-02' hemos acumulado la incidencia observada en los 7 días. Ahora ya es comparable con el dato de pruebas realizadas entre el 2020-02-05 (desde) y el 2020-03-02 (hasta)

```python
data_final_ultimo.head(7)
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Fecha</th>
      <th>dato_diario</th>
      <th>acumulado</th>
      <th>fecha_informe</th>
      <th>MovingAgerage</th>
      <th>acumulado semanal</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2020-02-25</td>
      <td>2.0</td>
      <td>2.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2020-02-26</td>
      <td>5.0</td>
      <td>7.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>6</th>
      <td>2020-02-27</td>
      <td>4.0</td>
      <td>11.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>9</th>
      <td>2020-02-28</td>
      <td>13.0</td>
      <td>24.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>12</th>
      <td>2020-02-29</td>
      <td>11.0</td>
      <td>35.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>15</th>
      <td>2020-03-01</td>
      <td>26.0</td>
      <td>61.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>18</th>
      <td>2020-03-02</td>
      <td>47.0</td>
      <td>108.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>15.428571</td>
      <td>108.0</td>
    </tr>
  </tbody>
</table>
</div>

Ahora, unimos las dos tablas por la fecha hasta, quedandonos únicamente con las fechas que encontramos en la tabla de pruebas diagnósticas

```python
DF_Pruebas_casos = data_final_ultimo.merge(df_diagnosticas, how='right', right_on = ['hasta'], left_on = ['Fecha'])
DF_Pruebas_casos.sort_values(by='Fecha').head(10)
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Fecha</th>
      <th>dato_diario</th>
      <th>acumulado</th>
      <th>fecha_informe</th>
      <th>MovingAgerage</th>
      <th>acumulado semanal</th>
      <th>desde</th>
      <th>hasta</th>
      <th>PCR</th>
      <th>TEST_RAPIDO_AC</th>
      <th>PRAg</th>
      <th>OTRAS_PRUEBAS_AC</th>
      <th>TOTAL</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2020-05-07</td>
      <td>588.0</td>
      <td>75069.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>494.285714</td>
      <td>3460.0</td>
      <td>2020-05-01</td>
      <td>2020-05-07</td>
      <td>61000</td>
      <td>23107</td>
      <td>0</td>
      <td>0</td>
      <td>84107</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2020-05-14</td>
      <td>667.0</td>
      <td>78476.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>486.714286</td>
      <td>3407.0</td>
      <td>2020-05-08</td>
      <td>2020-05-14</td>
      <td>63915</td>
      <td>34904</td>
      <td>0</td>
      <td>0</td>
      <td>98819</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2020-05-21</td>
      <td>407.0</td>
      <td>81080.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>372.000000</td>
      <td>2604.0</td>
      <td>2020-05-15</td>
      <td>2020-05-21</td>
      <td>60579</td>
      <td>12101</td>
      <td>0</td>
      <td>0</td>
      <td>72680</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2020-05-28</td>
      <td>542.0</td>
      <td>83854.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>396.285714</td>
      <td>2774.0</td>
      <td>2020-05-22</td>
      <td>2020-05-28</td>
      <td>74070</td>
      <td>10835</td>
      <td>0</td>
      <td>0</td>
      <td>84905</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2020-06-04</td>
      <td>467.0</td>
      <td>86730.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>410.857143</td>
      <td>2876.0</td>
      <td>2020-05-29</td>
      <td>2020-06-04</td>
      <td>76314</td>
      <td>7636</td>
      <td>0</td>
      <td>0</td>
      <td>83950</td>
    </tr>
    <tr>
      <th>5</th>
      <td>2020-06-11</td>
      <td>433.0</td>
      <td>89137.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>343.857143</td>
      <td>2407.0</td>
      <td>2020-06-05</td>
      <td>2020-06-11</td>
      <td>38372</td>
      <td>4383</td>
      <td>0</td>
      <td>0</td>
      <td>42755</td>
    </tr>
    <tr>
      <th>6</th>
      <td>2020-06-18</td>
      <td>424.0</td>
      <td>91408.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>324.428571</td>
      <td>2271.0</td>
      <td>2020-06-12</td>
      <td>2020-06-18</td>
      <td>35823</td>
      <td>4242</td>
      <td>0</td>
      <td>0</td>
      <td>40065</td>
    </tr>
    <tr>
      <th>7</th>
      <td>2020-06-25</td>
      <td>358.0</td>
      <td>93433.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>289.285714</td>
      <td>2025.0</td>
      <td>2020-06-19</td>
      <td>2020-06-25</td>
      <td>37667</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>37667</td>
    </tr>
    <tr>
      <th>8</th>
      <td>2020-07-02</td>
      <td>290.0</td>
      <td>95205.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>253.142857</td>
      <td>1772.0</td>
      <td>2020-06-26</td>
      <td>2020-07-02</td>
      <td>25833</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>25833</td>
    </tr>
    <tr>
      <th>9</th>
      <td>2020-07-09</td>
      <td>277.0</td>
      <td>96820.0</td>
      <td>datos/Informe de situación 24 de noviembre 202...</td>
      <td>230.714286</td>
      <td>1615.0</td>
      <td>2020-07-03</td>
      <td>2020-07-09</td>
      <td>32520</td>
      <td>123</td>
      <td>0</td>
      <td>3925</td>
      <td>36568</td>
    </tr>
  </tbody>
</table>
</div>




```python
plt.style.use(['classic'])

DF_Pruebas_casos = DF_Pruebas_casos.sort_values(by='Fecha')

fig=plt.figure(figsize=(20,10))
ax1=fig.add_subplot(111, label="1")
ax2=fig.add_subplot(111, label="1", frame_on=False)


ax1.set_title('Total pruebas diagnosticas y casos detectados semalmente', fontsize=24)
ax1.set_xlabel('fecha');  ax2.set_xlabel('fecha')
#ax1.set_ylabel('pruebas');  ax2.set_ylabel('casos')


g1, = ax1.plot(DF_Pruebas_casos['desde'], DF_Pruebas_casos['TOTAL'], color='red', 
         linewidth=3, 
         label='pruebas realizadas')
g2, = ax2.plot(DF_Pruebas_casos['desde'], DF_Pruebas_casos['acumulado semanal'], color='darkblue', 
         linewidth=3, 
         label='media movil 7 días')

ax1.set_xlabel("Fecha")
#ax1.set_ylabel("miles de pruebas" ,fontsize=15)
#ax1.tick_params(axis='x')
#ax1.tick_params(axis='y')

#ax2.yaxis.tick_right()
#ax2.set_ylabel('Casos diagnosticados',fontsize=15)       
#ax2.yaxis.set_label_position('right') 
#ax2.tick_params(axis='y')

#ax1.yaxis.set_label_position('left') 
#ax2.yaxis.set_label_position('right') 

plt.figlegend((g1,g2), 
              (r'pruebas realizadas (acumulado 7 días)',r'casos diagnosticados (acumulado 7 días)'), 
              loc=(0.095, 0.8))

ax1.set_ylim(0, 200000), ax2.set_ylim(0,  200000)  # y axis limits
#plt.tight_layout()
plt.show()
```

![png](/images/covid_madrid/output_34_1.png)


Dada la diferencia en los números, vamos a probar a hacer un gráfico situando cada una de las series un un eje y distinto:

```python
plt.clf()

plt.style.use(['classic'])

DF_Pruebas_casos = DF_Pruebas_casos.sort_values(by='Fecha')

fig=plt.figure(figsize=(20,10))
ax1=fig.add_subplot(111, label="1")
ax2=fig.add_subplot(111, label="1", frame_on=False)


ax1.set_title('Total pruebas diagnosticas y casos detectados semalmente', fontsize=24)
ax1.set_xlabel('fecha');  ax2.set_xlabel('fecha')
ax1.set_ylabel('pruebas');  ax2.set_ylabel('casos')


g1, = ax1.plot(DF_Pruebas_casos['desde'], DF_Pruebas_casos['TOTAL'], color='red', 
         linewidth=3, 
         label='pruebas realizadas')
g2, = ax2.plot(DF_Pruebas_casos['desde'], DF_Pruebas_casos['acumulado semanal'], color='darkblue', 
         linewidth=3, 
         label='media movil 7 días')

ax1.set_xlabel("Fecha")
ax1.set_ylabel("Pruebas" ,fontsize=15)
#ax1.tick_params(axis='x')
#ax1.tick_params(axis='y')

ax2.yaxis.tick_right()
ax2.set_ylabel('Casos diagnosticados',fontsize=15)       
ax2.yaxis.set_label_position('right') 
ax2.tick_params(axis='y')

ax1.yaxis.set_label_position('left') 
ax2.yaxis.set_label_position('right') 

plt.figlegend((g1,g2), 
              (r'pruebas realizadas (acumulado 7 días)',r'casos diagnosticados (acumulado 7 días)'), 
              loc=(0.095, 0.8))

#ax1.set_ylim(0, 200000), ax2.set_ylim(0,  200000)  # y axis limits
#plt.tight_layout()
plt.show()
```



![png](/images/covid_madrid/output_35_1.png)



```python

```

       
