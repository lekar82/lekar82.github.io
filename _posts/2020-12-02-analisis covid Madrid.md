<style> div { font-family:"Arial"; font-size: 18px; } </style>

# Evolución del Covid-19 en la CCAA de Madrid con Matplotlib

## Indice:

* [Objetivo](#Objetivo)
* [Librerías necesarias](#Librerías-necesarias)
* [Datos de contagios](#Datos-de-contagios)
* [Pruebas diagnosticas realizadas](#Pruebas-diagnosticas-realizadas)
* [Análisis conjunto del número de contagios y las pruebas realizadas](Análisis-conjunto-del-número-de-contagios-y-las-pruebas-realizadas)
* [UCI y defunciones](UCI-y-defunciones)

## Objetivo

Se pretende analizar la evolución del Covid-en la Comunidad Autónoma de Madrid. Tendremos en cuenta:  
* Número de contagios reportados.  
* Número de pruebas diagnosticas reportadas.  

Las fuentes que vamos a usar para realizar el estudio son:

* [Página oficial de la comunidad de Madrid](https://www.comunidad.madrid/servicios/salud/2019-nuevo-coronavirus#situacion-epidemiologica-actual): donde tenemos datos diarios del número de contagios.
* [Página oficial del Ministerio de Salud del Gobierno de España](https://www.mscbs.gob.es/profesionales/saludPublica/ccayes/alertasActual/nCov/): semanalmente publican en sus notas de presa los datos de pruebas realizadas.
* [Epdata](https://www.epdata.es/datos/evolucion-coronavirus-cada-comunidad/518/madrid/304): página que nos ofrece datos de los ingresos diarios en UCI, falleciomientos...

## Librerías necesarias

Importamos librerías necesarias

```python
import pandas as pd
import numpy as np
import math
import matplotlib.pyplot as plt
import string

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

A la fecha de realización de este análisis, el último pdf disponible es el correspondiente al 30 de Noviembre de 2020. Mediante el uso de la librería urllib accederemos a la página donde está contenido dicho pdf, y luego con las librerías io y pyPdf2 guardaremos el pdf en nuestro ordenador.

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
    datos_final = []

    df = tabula.read_pdf(x, pages = "all", multiple_tables = True)
    for num_col in range(9):
        df2 = df[num_col].dropna(axis=0, how='all')   
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
            #en algunos casos, se ha observado que el pdf no separa bien los datos de la primera tabla, 
            #úniendo dos valores en un record separado por espacio. Solucionamos eso tambíen                
            elif (len(df2.columns) == 8):
                if df2.columns[0] == 'Fecha Total':
                    columnas_1_2 = df2['Fecha Total'].str.split(" ", n = 1, expand = True) 
                    df2["col1"]= columnas_1_2[0]
                    df2["col2"]= columnas_1_2[1]
                    columnas = list(df2)
                    for j in range(len(df2)):
                        for i in columnas:
                            datos.append(df2[i].iloc[j])

        except IndexError:
            next
    # eliminamos todo el texto  
    for item in datos:
        if item not in ['Fecha', 'Total', 'Notificación','Diario', 'Acumulado' ]:
            datos_final.append(item)
     # por último, sabemos que nuestros tenemos 3 datos para cada row, hacemos un resaphe de la lista de manera que 
    # nos coloque nuestros datos en una tabla de n_días * 3 columnas
    #return(datos_final)
    datos_np = np.array(datos_final).reshape(-1, 3)
    
    datos_df = pd.DataFrame(datos_np,columns=['Fecha', 'dato_diario', 'acumulado'])
    datos_df['fecha_informe'] = file
    datos_df = datos_df[(datos_df['Fecha'] != 'Notificación') & (datos_df['Fecha'] != 'Fecha')] 
        
    
    return(datos_df)
   ```
Extraemos los datos 

```python
file = 'datos/Informe de situación 30 de noviembre 2020.pdf'
data_final_ultimo = extr_date(file)
```
Analizamos las variables de nuestro dataframe.
* Fecha: Fecha de contagio
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
     0   Fecha          279 non-null    object
     1   dato_diario    279 non-null    object
     2   acumulado      279 non-null    object
     3   fecha_informe  279 non-null    object
    dtypes: object(4)
    memory usage: 10.9+ KB
    
Como podemos observar, todas las variables son de tipo string ahora mismo. Cambiamos los tipos:

```python
data_final_ultimo["dato_diario"] = pd.to_numeric(data_final_ultimo["dato_diario"])
data_final_ultimo["acumulado"] = pd.to_numeric(data_final_ultimo["acumulado"])
data_final_ultimo["Fecha"] = data_final_ultimo["Fecha"].str.replace(' ', '')
data_final_ultimo["Fecha"] = pd.to_datetime(data_final_ultimo["Fecha"], format="%d/%m/%Y")
data_final_ultimo.info()
```
    <class 'pandas.core.frame.DataFrame'>
    Int64Index: 279 entries, 0 to 278
    Data columns (total 4 columns):
     #   Column         Non-Null Count  Dtype         
    ---  ------         --------------  -----         
     0   Fecha          279 non-null    datetime64[ns]
     1   dato_diario    279 non-null    float64       
     2   acumulado      279 non-null    float64       
     3   fecha_informe  279 non-null    object        
    dtypes: datetime64[ns](1), float64(2), object(1)
    memory usage: 10.9+ KB
    
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
      <td>datos/Informe de situación 30 de noviembre 202...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2020-04-12</td>
      <td>408.0</td>
      <td>56595.0</td>
      <td>datos/Informe de situación 30 de noviembre 202...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2020-05-29</td>
      <td>525.0</td>
      <td>84628.0</td>
      <td>datos/Informe de situación 30 de noviembre 202...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2020-02-26</td>
      <td>5.0</td>
      <td>7.0</td>
      <td>datos/Informe de situación 30 de noviembre 202...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2020-04-13</td>
      <td>1208.0</td>
      <td>57803.0</td>
      <td>datos/Informe de situación 30 de noviembre 202...</td>
    </tr>
  </tbody>
</table>
</div>


Haciendo uso de la librería matplotlib, vamos a graficar nuestros datos

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

Vamos a añadir al gráfico una linea suavizada que nos permita ver mejor la tendencia de nuestros datos. Para ello usamos la función rolling. Hay que acordarse de ordenar previamente los datos para que el resultado sea el deseado

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
En el gráfico se observa perfectamente las dos olas reflejadas y como la incidencia en cuanto a número de contagios detectados en la segunda ola ha sido muy superior a la de la primera. ¿Eso significa que la situación en la segunda ola ha sido mucho peor? no necesariamente, simplemente nos indica que se han detectado más casos. 

![png](/images/covid_madrid/output_21_1.png)

## Pruebas diagnosticas realizadas.

La información de las pruebas diagnósticas la podemos encontrar en la página del Ministerio de Sanidad del Gobierno de España. En este caso, no tenemos un informa diario que acumule los datos de los informes anteriores como nos pasaba con el número de contagios, por lo que tendremos que consultar todos y cada uno de los informes. Tampoco en este caso disponemos de datos diarios, los datos que nos ofrece el ministerio son agregaciones semanales.

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
 Manualmente creamos el excel con la información que necesitamos y a continuación importamos los datos y echamos un ojo a ver que pinta tienen nuestros datos. 
 
 Las variables de este dataframe son:
 * desde y hasta: intervalo de tiempo al que se refieren los datos.
 * PCR, TEST_RAPIDO_AC,OTRAS_PRUEBAS_AC : numero de cada tipo de pruebas realizadas       
 * TOTAL: numero total de pruebas realizadas            

```python
import pandas as pd

df_diagnosticas = pd.read_excel('datos/PRUEBAS.xls')
df_diagnosticas = df_diagnosticas.fillna(0) # los na suponemos que son semanas sin pruebas 
df_diagnosticas.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 30 entries, 0 to 29
    Data columns (total 7 columns):
     #   Column            Non-Null Count  Dtype         
    ---  ------            --------------  -----         
     0   desde             30 non-null     datetime64[ns]
     1   hasta             30 non-null     datetime64[ns]
     2   PCR               30 non-null     int64         
     3   TEST_RAPIDO_AC    30 non-null     int64         
     4   PRAg              30 non-null     int64         
     5   OTRAS_PRUEBAS_AC  30 non-null     int64         
     6   TOTAL             30 non-null     int64         
    dtypes: datetime64[ns](2), int64(5)
    memory usage: 1.8 KB
    
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

Viendo el gráfico, da la impresión de que en las últimas semanas la tendencia en el número de pruebas es cecreciente. 

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
      <td>datos/Informe de situación 30 de noviembre 202...</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2020-02-26</td>
      <td>5.0</td>
      <td>7.0</td>
      <td>datos/Informe de situación 30 de noviembre 202...</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>6</th>
      <td>2020-02-27</td>
      <td>5.0</td>
      <td>12.0</td>
      <td>datos/Informe de situación 30 de noviembre 202...</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>9</th>
      <td>2020-02-28</td>
      <td>13.0</td>
      <td>25.0</td>
      <td>datos/Informe de situación 30 de noviembre 202...</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>12</th>
      <td>2020-02-29</td>
      <td>11.0</td>
      <td>36.0</td>
      <td>datos/Informe de situación 30 de noviembre 202...</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>15</th>
      <td>2020-03-01</td>
      <td>26.0</td>
      <td>62.0</td>
      <td>datos/Informe de situación 30 de noviembre 202...</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>18</th>
      <td>2020-03-02</td>
      <td>47.0</td>
      <td>109.0</td>
      <td>datos/Informe de situación 30 de noviembre 202...</td>
      <td>15.571429</td>
      <td>109.0</td>
    </tr>
  </tbody>
</table>
</div>


Ahora, unimos las dos tablas por la fecha hasta, quedandonos únicamente con las fechas que encontramos en la tabla de pruebas diagnósticas

```python
DF_Pruebas_casos = data_final_ultimo.merge(df_diagnosticas, how='right', right_on = ['hasta'], left_on = ['Fecha'])
DF_Pruebas_casos.sort_values(by='Fecha').head(3)
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
      <td>599.0</td>
      <td>75214.0</td>
      <td>datos/Informe de situación 30 de noviembre 202...</td>
      <td>499.857143</td>
      <td>3499.0</td>
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
      <td>677.0</td>
      <td>78657.0</td>
      <td>datos/Informe de situación 30 de noviembre 202...</td>
      <td>491.857143</td>
      <td>3443.0</td>
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
      <td>414.0</td>
      <td>81303.0</td>
      <td>datos/Informe de situación 30 de noviembre 202...</td>
      <td>378.000000</td>
      <td>2646.0</td>
      <td>2020-05-15</td>
      <td>2020-05-21</td>
      <td>60579</td>
      <td>12101</td>
      <td>0</td>
      <td>0</td>
      <td>72680</td>
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

g1, = ax1.plot(DF_Pruebas_casos['desde'], DF_Pruebas_casos['TOTAL'], color='red', 
         linewidth=3, 
         label='pruebas realizadas')
g2, = ax2.plot(DF_Pruebas_casos['desde'], DF_Pruebas_casos['acumulado semanal'], color='darkblue', 
         linewidth=3, 
         label='media movil 7 días')

ax1.set_xlabel("Fecha")

plt.figlegend((g1,g2), 
              (r'pruebas realizadas (acumulado 7 días)',r'casos diagnosticados (acumulado 7 días)'), 
              loc=(0.095, 0.8))

ax1.set_ylim(0, 200000), ax2.set_ylim(0,  200000)  # y axis limits
#plt.tight_layout()
plt.show()
```

![png](/images/covid_madrid/output_57_1.png)

Dada la diferencia en los números, vamos a probar a hacer un gráfico situando cada una de las series un un eje y distinto.

```python
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

plt.show()
```

![png](/images/covid_madrid/output_59_1.png)

El problema de este gráfico es que, a simple vista, puede llevarnos a conclusiones erronéas ya que si no nos fijamos bien en los ejes, da la impresión de que al final de la serie el número de contagios es mayor que el número de pruebas diagnósticas realizadas. Cosa que ni es cierta ni tendría sentido.

Para evitar problemas con la interpretación, siempre podemos representar las dos series en dos gráficos distintos. 

```python
plt.style.use(['classic'])

DF_Pruebas_casos = DF_Pruebas_casos.sort_values(by='Fecha')
fig, (ax1, ax2) = plt.subplots(2, 1, figsize = (20,15))

ax1.set_title('Pruebas realizadas (acumulado 7 días)', fontsize=15)
ax2.set_title('Casos diagnosticados (acumulado 7 días)', fontsize=15)

g1, = ax1.plot(DF_Pruebas_casos['desde'], DF_Pruebas_casos['TOTAL'], color='red', 
         linewidth=3, 
         label='pruebas realizadas')
g2, = ax2.plot(DF_Pruebas_casos['desde'], DF_Pruebas_casos['acumulado semanal'], color='darkblue', 
         linewidth=3, 
         label='media movil 7 días')

plt.show()
```

![png](/images/covid_madrid/output_58_1.png)

## UCI y defunciones  

Uno de los principales problemas que tiene la pandemia es lo agresivo que ha demostrado ser el virus. Analizamos a continuación la evolución del número de ingresados diariamente en UCI y el número de fallecimientos.

Los datos se han obtenido de la página de datos [epdata](www.epdata.es) 

Importamos la información relativa a las defunciones.

```python
df_muertos = pd.read_csv('datos/muertos_diarios_por_coron.csv', sep = ";")
df_muertos.head(5)
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
      <th>Año</th>
      <th>Periodo</th>
      <th>Número de fallecidos por fecha de defunción</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2020</td>
      <td>Día 1 de marzo</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2020</td>
      <td>Día 2 de marzo</td>
      <td>0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2020</td>
      <td>Día 3 de marzo</td>
      <td>1</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2020</td>
      <td>Día 4 de marzo</td>
      <td>0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2020</td>
      <td>Día 5 de marzo</td>
      <td>1</td>
    </tr>
  </tbody>
</table>
</div>

Vemos que la fecha viene como un string dificilmente convertible en datatime de una manera directa. Por ello, nos vamos a crear una nueva columna fecha a la que simplemente asginaremos valores tipo datetime de una manera secuencial:

```python
date_rng = pd.date_range(start='2020/03/01', end='2020/12/31', freq='D')
date_rng
```


    DatetimeIndex(['2020-03-01', '2020-03-02', '2020-03-03', '2020-03-04',
                   '2020-03-05', '2020-03-06', '2020-03-07', '2020-03-08',
                   '2020-03-09', '2020-03-10',
                   ...
                   '2020-12-22', '2020-12-23', '2020-12-24', '2020-12-25',
                   '2020-12-26', '2020-12-27', '2020-12-28', '2020-12-29',
                   '2020-12-30', '2020-12-31'],
                  dtype='datetime64[ns]', length=306, freq='D')


```python
df_muertos['fecha'] = date_rng[0:len(df_muertos)]
df_muertos.head(5)
```
Vemos las 5 primeras lineas para comprobar que ha salido correctamente:

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
      <th>Año</th>
      <th>Periodo</th>
      <th>Número de fallecidos por fecha de defunción</th>
      <th>fecha</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2020</td>
      <td>Día 1 de marzo</td>
      <td>0</td>
      <td>2020-03-01</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2020</td>
      <td>Día 2 de marzo</td>
      <td>0</td>
      <td>2020-03-02</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2020</td>
      <td>Día 3 de marzo</td>
      <td>1</td>
      <td>2020-03-03</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2020</td>
      <td>Día 4 de marzo</td>
      <td>0</td>
      <td>2020-03-04</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2020</td>
      <td>Día 5 de marzo</td>
      <td>1</td>
      <td>2020-03-05</td>
    </tr>
  </tbody>
</table>
</div>

Y hacemos un info de la tabla de dato para comprobar que el tipo asociado a nuestras variables es el deseado:

```python
df_muertos.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 274 entries, 0 to 273
    Data columns (total 4 columns):
     #   Column                                       Non-Null Count  Dtype         
    ---  ------                                       --------------  -----         
     0   Año                                          274 non-null    int64         
     1   Periodo                                      274 non-null    object        
     2   Número de fallecidos por fecha de defunción  274 non-null    int64         
     3   fecha                                        274 non-null    datetime64[ns]
    dtypes: datetime64[ns](1), int64(2), object(1)
    memory usage: 8.7+ KB
    
Ahora repetimos el proceso con la información de los ingresos UCI

```python
df_UCI = pd.read_csv('datos/casos_que_han_requerido_uci.csv', sep = ";")
df_UCI['fecha'] = date_rng[0:len(df_UCI)]
df_UCI.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 275 entries, 0 to 274
    Data columns (total 4 columns):
     #   Column                                                Non-Null Count  Dtype         
    ---  ------                                                --------------  -----         
     0   Año                                                   275 non-null    int64         
     1   Periodo                                               275 non-null    object        
     2   Ingresos en UCI por coronavirus por fecha de ingreso  275 non-null    int64         
     3   fecha                                                 275 non-null    datetime64[ns]
    dtypes: datetime64[ns](1), int64(2), object(1)
    memory usage: 8.7+ KB
    
Unimos las dos tablas y visualizamos los resultados

```python
df_uci_defunciones = df_UCI.merge(df_muertos,on = ['fecha'])
```

```python
plt.clf()

plt.style.use(['classic'])

df_uci_defunciones = df_uci_defunciones.sort_values(by='fecha')
fig, ax = plt.subplots(1, figsize=(24, 10))

ax.plot(df_uci_defunciones['fecha'], df_uci_defunciones['Ingresos en UCI por coronavirus por fecha de ingreso'], color='red', 
         linewidth=3, 
         label='Ingresos diarios UCI')
ax.plot(df_uci_defunciones['fecha'], df_uci_defunciones['Número de fallecidos por fecha de defunción'], color='darkblue', 
         linewidth=3, 
         label='Fallecimientos en hospitales')
plt.title('Ingresos en UCI y fallecimientos en hospitales', fontsize = 24)
plt.legend(fontsize=12)
plt.show()
```
Al igual que vimos con las otras series analizadas, se puede ver perfectamente reflejadas las dos olas, sobre todo en el número de defunciones. Afortunadamente se observa que la segunda ola ha tenido efectos menos nefastos que la primera sobre nuestra población.

![png](/images/covid_madrid/output_51_1.png)
