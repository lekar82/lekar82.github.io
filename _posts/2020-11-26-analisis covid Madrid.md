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
## Datos contagios

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

![png](/images/covid_madrid/output_18_1.png)

