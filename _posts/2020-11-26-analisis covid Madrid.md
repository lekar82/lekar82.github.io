<style> div { font-family:"Arial"; font-size: 18px; } </style>

# Evolución del Covid-19 en la CCAA de Madrid

## Indice:

* [Objetivo](#Objetivo)
* [Librerías necesarias](#Librerías-necesarias)

## Objetivo

Se pretende analizar la evolución del Covid-en la Comunidad Autónoma de Madrid. Tendremos en cuenta:  
* Número de contagios reportados.  
* Número de pruebas diagnosticas reportadas.  

Las fuentes que vamos a usar para realizar el estudio son:

* [Página oficial de la comunidad de Madrid](https://www.comunidad.madrid/servicios/salud/2019-nuevo-coronavirus#situacion-epidemiologica-actual): en esta página podemos 


## Librerías necesarias

Importamos librerías necesarias


```python
from bs4 import BeautifulSoup
import urllib.request
import pandas as pd
import re

import pandas as pd
import numpy as np

#!pip install tabula
#!pip install pyPdf2
#!pip install StringIO
import tabula
from PyPDF2 import PdfFileWriter, PdfFileReader
import io

import math

import matplotlib.pyplot as plt
```

### Guardamos todos los informes de contagios de la comunidad de Madrid

```python
url = 'https://www.comunidad.madrid/servicios/salud/2019-nuevo-coronavirus#situacion-epidemiologica-actual'
ourUrl = urllib.request.urlopen(url)
soup = BeautifulSoup(ourUrl, 'html.parser')

links_pdfs = []

for link in soup.find_all('li'):
    links = link.find('a')  
    try:
        #links.strip() if links is not None else ''
        if 'cam_covid19.pdf' in links.get('href'):
            links_pdfs.append([links.get('href'), links.get_text("target")])
    except:
        AttributeError
    
len(links_pdfs)
```
    182

A excepcción del informe del 14 de agosto que tiene un id raro mayor que el resto, el resto de los ficheros son consecutivos. Por lo que el segundo fichero por la cola será el último fichero disponible

```python
links_pdfs.sort()
```

```python
links_pdfs[-2]

```




    ['/sites/default/files/doc/sanidad/201124_cam_covid19.pdf',
     'Informe de situación 24 de noviembre 2020']



Guardamos todos los pdfs por si nos hiceran falta en algún momento:


```python
links_contagios = []

for i in range(len(links_pdfs)):
    url = "https://www.comunidad.madrid" + links_pdfs[i][0]
    writer = PdfFileWriter()

    remoteFile =  urllib.request.urlopen((url)).read()
    #memoryFile = io.StringIO(remoteFile)
    memoryFile = io.BytesIO(remoteFile)
    pdfFile = PdfFileReader(memoryFile)
    for pageNum in range(pdfFile.getNumPages()):
        currentPage = pdfFile.getPage(pageNum)
        #currentPage.mergePage(watermark.getPage(0))
        writer.addPage(currentPage)

    name_path = "datos/" +  str(links_pdfs[i][1]) + ".pdf"
    links_contagios.append(name_path)
        
    outputStream = open(name_path,"wb")
    writer.write(outputStream)
    outputStream.close()
```

   


```python
#datos_df_fin = []

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
            print(num_col, len(df2.columns))
            #las hojas con 9 o más columnas son las que contienen nuestros datos:
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


```python
fichero = links_pdfs[-2][1]
fichero
```




    'Informe de situación 24 de noviembre 2020'




```python
import tabula

#revisar 23 de noviembre

fichero = links_pdfs[-2][1]


file = 'datos/' + fichero + '.pdf'

data_final_ultimo = extr_date(file)

```

    0 1
    1 11
    3 9
    4 2
    5 2
    6 1
    7 7
    8 6
    

    FutureWarning: elementwise comparison failed; returning scalar instead, but in the future will perform elementwise comparison [array_ops.py:253]
    


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
    

Cambiamos los tipos de nuestras variables:


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


plt.xticks(rotation= 45 )
plt.xticks(fontsize=30)
plt.yticks(fontsize=30)

plt.xlabel('Fecha', fontsize=50)
plt.ylabel('Numero de contagios', fontsize=50)
```




    Text(0, 0.5, 'Numero de contagios')




![png](output_18_1.png)



```python
#logra obtener las medias móviles a partir del promedio de los datos de t, ... t-6

data_final_ultimo = data_final_ultimo.sort_values(by='Fecha')

data_final_ultimo['MovingAgerage'] = data_final_ultimo['dato_diario'].rolling(min_periods=7, window=7).mean()
```


```python
print(plt.style.available)
```

    ['Solarize_Light2', '_classic_test_patch', 'bmh', 'classic', 'dark_background', 'fast', 'fivethirtyeight', 'ggplot', 'grayscale', 'seaborn', 'seaborn-bright', 'seaborn-colorblind', 'seaborn-dark', 'seaborn-dark-palette', 'seaborn-darkgrid', 'seaborn-deep', 'seaborn-muted', 'seaborn-notebook', 'seaborn-paper', 'seaborn-pastel', 'seaborn-poster', 'seaborn-talk', 'seaborn-ticks', 'seaborn-white', 'seaborn-whitegrid', 'tableau-colorblind10']
    


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




    <matplotlib.legend.Legend at 0x25814442850>




![png](output_21_1.png)


Vamos a añadir la información de las pruebas realizadas.

Esta información nos la obrece el gobierno de España de forma semanal.
Para hacer las dos series comprables, agregaremos la serie de contagios para tener un dato semanal



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
    


```python
data_final_ultimo = data_final_ultimo.sort_values(by='Fecha')

data_final_ultimo['acumulado semanal'] = data_final_ultimo['dato_diario'].rolling(min_periods=7, window=7).sum()
```

como podemos ver, en la fecha = '2020-03-02' hemos acumulado la incidencia observada en los 7 días  

ahora ya es comparable con el dato de pruebas realizadas entre el 2020-02-05 (desde) y el 2020-03-02 (hasta)


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




```python
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
    


```python
df_diagnosticas
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
      <td>2020-07-03</td>
      <td>2020-07-09</td>
      <td>32520</td>
      <td>123</td>
      <td>0</td>
      <td>3925</td>
      <td>36568</td>
    </tr>
    <tr>
      <th>10</th>
      <td>2020-07-10</td>
      <td>2020-07-16</td>
      <td>26841</td>
      <td>1550</td>
      <td>0</td>
      <td>2341</td>
      <td>30732</td>
    </tr>
    <tr>
      <th>11</th>
      <td>2020-07-17</td>
      <td>2020-07-23</td>
      <td>32661</td>
      <td>3505</td>
      <td>2191</td>
      <td>0</td>
      <td>38357</td>
    </tr>
    <tr>
      <th>12</th>
      <td>2020-07-24</td>
      <td>2020-07-30</td>
      <td>38572</td>
      <td>3201</td>
      <td>0</td>
      <td>2111</td>
      <td>43884</td>
    </tr>
    <tr>
      <th>13</th>
      <td>2020-07-31</td>
      <td>2020-08-06</td>
      <td>44693</td>
      <td>2042</td>
      <td>0</td>
      <td>3076</td>
      <td>49811</td>
    </tr>
    <tr>
      <th>14</th>
      <td>2020-08-07</td>
      <td>2020-08-13</td>
      <td>58601</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>58601</td>
    </tr>
    <tr>
      <th>15</th>
      <td>2020-08-14</td>
      <td>2020-08-20</td>
      <td>78871</td>
      <td>2698</td>
      <td>0</td>
      <td>3782</td>
      <td>85351</td>
    </tr>
    <tr>
      <th>16</th>
      <td>2020-08-21</td>
      <td>2020-08-27</td>
      <td>99828</td>
      <td>2483</td>
      <td>0</td>
      <td>3595</td>
      <td>105906</td>
    </tr>
    <tr>
      <th>17</th>
      <td>2020-08-28</td>
      <td>2020-09-03</td>
      <td>115121</td>
      <td>2863</td>
      <td>0</td>
      <td>4256</td>
      <td>122240</td>
    </tr>
    <tr>
      <th>18</th>
      <td>2020-09-04</td>
      <td>2020-09-10</td>
      <td>123233</td>
      <td>3576</td>
      <td>0</td>
      <td>5168</td>
      <td>131977</td>
    </tr>
    <tr>
      <th>19</th>
      <td>2020-09-11</td>
      <td>2020-09-17</td>
      <td>139792</td>
      <td>3434</td>
      <td>0</td>
      <td>5006</td>
      <td>148232</td>
    </tr>
    <tr>
      <th>20</th>
      <td>2020-09-18</td>
      <td>2020-09-24</td>
      <td>154765</td>
      <td>5180</td>
      <td>0</td>
      <td>7772</td>
      <td>167717</td>
    </tr>
    <tr>
      <th>21</th>
      <td>2020-09-25</td>
      <td>2020-10-01</td>
      <td>153856</td>
      <td>21337</td>
      <td>0</td>
      <td>10628</td>
      <td>185821</td>
    </tr>
    <tr>
      <th>22</th>
      <td>2020-10-02</td>
      <td>2020-10-08</td>
      <td>92261</td>
      <td>0</td>
      <td>0</td>
      <td>11260</td>
      <td>103521</td>
    </tr>
    <tr>
      <th>23</th>
      <td>2020-10-09</td>
      <td>2020-10-15</td>
      <td>58198</td>
      <td>0</td>
      <td>62232</td>
      <td>0</td>
      <td>120430</td>
    </tr>
    <tr>
      <th>24</th>
      <td>2020-10-16</td>
      <td>2020-10-22</td>
      <td>64142</td>
      <td>0</td>
      <td>107273</td>
      <td>1131</td>
      <td>172546</td>
    </tr>
    <tr>
      <th>25</th>
      <td>2020-10-23</td>
      <td>2020-10-29</td>
      <td>61754</td>
      <td>0</td>
      <td>130834</td>
      <td>0</td>
      <td>192588</td>
    </tr>
    <tr>
      <th>26</th>
      <td>2020-10-30</td>
      <td>2020-11-05</td>
      <td>54798</td>
      <td>123008</td>
      <td>0</td>
      <td>0</td>
      <td>177806</td>
    </tr>
    <tr>
      <th>27</th>
      <td>2020-11-06</td>
      <td>2020-11-12</td>
      <td>57790</td>
      <td>0</td>
      <td>132547</td>
      <td>0</td>
      <td>190337</td>
    </tr>
    <tr>
      <th>28</th>
      <td>2020-11-13</td>
      <td>2020-11-19</td>
      <td>57212</td>
      <td>0</td>
      <td>115370</td>
      <td>0</td>
      <td>172582</td>
    </tr>
  </tbody>
</table>
</div>




```python
df_diagnosticas["desde"] = pd.to_datetime(df_diagnosticas["desde"], format="%d/%m/%Y")
df_diagnosticas["hasta"] = pd.to_datetime(df_diagnosticas["hasta"], format="%d/%m/%Y")
```


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


![png](output_30_0.png)


unimos las dos tablas por la fecha hasta, quedandonos únicamente con las fechas que encontramos en la tabla de pruebas diagnósticas



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
DF_Pruebas_casos.count()
```




    Fecha                29
    dato_diario          29
    acumulado            29
    fecha_informe        29
    MovingAgerage        29
    acumulado semanal    29
    desde                29
    hasta                29
    PCR                  29
    TEST_RAPIDO_AC       29
    PRAg                 29
    OTRAS_PRUEBAS_AC     29
    TOTAL                29
    dtype: int64




```python
plt.clf()

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


    <Figure size 640x480 with 0 Axes>



![png](output_34_1.png)



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


    <Figure size 640x480 with 0 Axes>



![png](output_35_1.png)



```python

```
