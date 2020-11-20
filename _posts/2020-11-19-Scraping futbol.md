<style>
div { 
  font-family:"Arial";
  font-size: 18px;
  }
</style>

<h1 style="background-color:MediumSeaGreen;">Web scraping datos de futbol con BeautifulSoup </h1>

## Indice:

* [Objetivo](#Objetivo)
* [Librerías necesarias](#Librerías-necesarias)
* [Extracción URLs de cada partido](#Extracción-URLs-de-cada-partido)
* [Información de una jornada](#Información-de-una-jornada)
* [Información de la temporada completa](#Información-de-la-temporada-completa)


## Objetivo

El proposito de este post es extraer información de los resultados de la liga nacional de futbol española (Liga Santander) de la página http://www.futbolfantasy.com.

Visualmente, la página de la que queremos extraer los datos es la siguiten:

![png](/images/futbol/Captura_futbol.PNG)

Podemos ver como hay un scroll en la parte de arriba que nos servirá para ir moviendonos de una temporada a otra, y luego podemos ver diferentes cajas con enlaces a cada uno de los partidos. La idea es generar un código que sea capaz de, para una temporada dada, recorrer cada uno de los partidos e ir sacando la siguiente información:

<ul>
<li> arbitro	</li>
<li> fecha_jornada	</li>
<li> Jornada	</li>
<li> dia_semana	</li>
<li> hora	</li>
<li> anno	</li>
<li> dia	</li>
<li> mes </li>
<li>metricas para el equipo  local y el visitante: </li> <ul>
  <li> equipo </li>
  <li> goles </li>
  <li> Balones_robados </li>
  <li> Centros_local	</li>
  <li> Centros_precisos	</li>
  <li> Córners </li>
  <li> Faltas </li>
  <li> Pases_interceptados	</li>
  <li> Penaltis	</li>
  <li> Tarjetas_amarillas </li>
  <li> Tarjetas_rojas	</li>
  <li> Tiros_a_puerta </li>
  <li> Tiros </li> </ul>
</ul>
*No todas las métricas están disponibles para todas las jornadas*

## Librerías necesarias

```python
from bs4 import BeautifulSoup
import urllib.request
import pandas as pd
import re
```

## Extracción URLs de cada partido

Lo primero que tenemos que indicar es de que temporada queremos extraer los datos. Para este post hemos elegido 2019.

Como hemos comentado, no todas las temporadas tienen disponible el mismo número de métricas, por lo que incluiremos un loop en el que indicaremos al programa el número de métricas disponibles según la temporada que estemos analizando.

```python
temporada = 2019

numero_metricas = 0
if temporada > 2017:
    numero_metricas = 16
elif temporada <= 2017 and temporada > 2015:
    numero_metricas = 15
else:
    numero_metricas = 11
```
A continuación, realizaremos la conexión con nuestra página para poder comenzar a buscar los datos deseados:

```python
url = 'http://www.futbolfantasy.com/laliga/calendario/' + str(temporada)
ourUrl = urllib.request.urlopen(url)
soup = BeautifulSoup(ourUrl, 'html.parser')
```

La información de cada una de las jornadas está recogida en un url diferente, por lo que crearemos una lista en la que iremos guardando las urls de cada una de las jornadas. 

Si miramos el código fuente de la página, las urls que buscamos están en bajo la etiqueta <a> con el atributo 'href'. A su vez, esta etiqueta está bajo  la etiqueta de agrupación div con atributo class iguala 'jornada:

![png](/images/futbol/Captura_link_a_jornada.PNG)

En un primer intento, el código que se intentó usar para obtener la información era el siguiente:

```python
lista_links_informacion = []
for link in soup.find_all('div', {'class': 'jornada'}):
    links = link.find('a')
    lista_links_informacion.append(links.get('href'))
    
print(len(lista_links_informacion))
```
89

Pero de los 380 partidos que tiene una temporada completa únicamente detectó 89.

En un segundo intento, se intentó crear una lista en la que buscase todos los atributos href que comenzaran por "www.futbolfantasy.com/partidos/". 

```python
lista = soup.find_all(attrs={'href': re.compile("www.futbolfantasy.com/partidos/")})
```

```python
print(lista[0])
```
```
<a class="partido terminado" data-tooltip="Valencia 1-0 Las Palmas" href="https://www.futbolfantasy.com/partidos/3944-valencia-las-palmas">
    <div class="equipo local">
    <img alt="Valencia" src="https://static.futbolfantasy.com/uploads/images/equipos/escudom/18.png"/>
    </div>
    <div class="info">
    <div class="fase">
    					Jornada 1
    				</div>
    <div class="resultado">
    						1-0
    					</div>
    </div>
    <div class="equipo visitante">
    <img alt="Las Palmas" src="https://static.futbolfantasy.com/uploads/images/equipos/escudom/27.png"/>
    </div>
    <div class="clearfix"></div>
    </a> 
```
Y luego crear una función para limpiar el resultado y quedarnos solo con los urls:

```python
def extraer_links(x):
    try:
        found = re.search('href="(.+?)">', text).group(1)
    except AttributeError:
        found = 'no encontrado' # apply your error handling
    return(found)
```

```python
lista_links_informacion = []
for link in lista:
    text = str(link)
    #print(text)
    lista_links_informacion.append(extraer_links(text))  
```
Con este código extraeremos, además de los links de los partidos de la liga, links a otros partidos que se jugaban en la misma semana (amistosos, champions league...)

```python
print(len(lista_links_informacion))
```
431

Si analizamos las urls que nos redireccionan a los partidos, tienen una estructura común : https://www.futbolfantasy.com/partidos/ + id partido + equipos enfrentados. Los ids de los partidos son consecutivos, el id más bajo se corresponde con el primer partido de la temporada. Usaremos es información para filtrar únicamente los partidos de liga. 
 
Crearemos una lista con los ids, buscaremos el más bajo y luego filtraremos todas las urls cuyo id esté contenido entre en el id del primer partido y el id del primer partido + 379 (38jornadas x 10 partidos por jornada) 
 
```python 
lista_ids = []
for id in lista_links_informacion:
    codigo = int(re.sub("[^0-9]", "", id))
    lista_ids.append(codigo) 

primer_partido = min(lista_ids)
ultimo_partido = min(lista_ids) + 379
```

```python 
lista_links_informacion_liga = []
for i in lista_links_informacion:
    codigo = int(re.sub("[^0-9]", "", i))
    if int(codigo) <= ultimo_partido:
        lista_links_informacion_liga.append(i)

print(len(lista_links_informacion_liga))
```
380

Ahora si tenemos filtradas correctamente nuestras 380 urls.

## Información de una jornada

Vamos a sacar la información correspondiente al primer url de la lista que hemos creado anteriormente. 

**Conectamos con la URL:**

```python
url2 = lista_links_informacion_liga[0]
ourUrl2 = urllib.request.urlopen(url2)
soup2 = BeautifulSoup(ourUrl2, 'html.parser')
```
¡Ahora ya podemos empezar a extraer información!

En la siguiente imagen podemos ver la información que podemos sacar del partido: árbitro, fecha y diferentes métricas de cada uno de los dos equipos. Igual que hicimos para sacar las url, vamos mirando en el código funte de la página las étiquetas y atributos que nos ayudan a encontrar nuestros datos.

![png](/images/futbol/Captura_metricas.PNG)

**Información del árbitro:**

![png](/images/futbol/Captura_arbitro.PNG)

```python
arbitro = soup2.find_all('span', {'class':'link'})
```

```python
print(arbitro)
```

    [<span class="link">Juan Martínez Munuera</span>]
    
**Metricas del pártido:**

![png](/images/futbol/Captura_metricas2.PNG)

Todas las métricas están bajo la misma etiqueta, por lo que haremos un loop para ir extrayendo cada una de las métricas. Crearemos 3 listas, una con los nombres de las métricas y cada una de las otras dos con los valores de las métricas para el equipo local y el equipo visitante. Una vez el loop haya recorido todos los enlaces, nos valdremos de esas listas para crear un dataframe.

```python
local = []
visitante = []
metricas = []

for j in soup2.find_all('div', attrs={'class':'stat'}):
    estadisticas_local = j.find('div', attrs = {'class': 'local'})
    #print(estadisticas_local)
    if estadisticas_local != None:
        estadisticas_local = str(estadisticas_local).replace('<div class="local">', '').replace('</div>', '')
        local.append(estadisticas_local)
        
    estadisticas_visitante = j.find('div', {'class': 'visitante'})
    
    if estadisticas_visitante != None:
        estadisticas_visitante = str(estadisticas_visitante).replace('<div class="visitante">', '').replace('</div>', '')
        visitante.append(estadisticas_visitante)

    nombre_metrica = j.find('div', {'class': 'name'})
    if nombre_metrica != None:
        nombre_metrica = str(nombre_metrica).replace('<div class="name">', '').replace('</div>', '')
        metricas.append(nombre_metrica)
```
Creamos el datadrame:

```python
df = pd.DataFrame(list(zip(metricas, local, visitante)), 
               columns =['nombre_metrica', 'val_local', 'val_visitante']) 

```
El resultado, como podemos ver, es una tabla con 3 columnas y tantas filas como métricas haya encontrado el programa.

```python
display(df)
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
      <th>nombre_metrica</th>
      <th>val_local</th>
      <th>val_visitante</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Tiros</td>
      <td>16</td>
      <td>15</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Tiros a puerta</td>
      <td>7</td>
      <td>1</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Centros</td>
      <td>19</td>
      <td>27</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Centros precisos</td>
      <td>3</td>
      <td>5</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Córners</td>
      <td>6</td>
      <td>5</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Faltas</td>
      <td>13</td>
      <td>6</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Pases interceptados</td>
      <td>18</td>
      <td>8</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Balones robados</td>
      <td>13</td>
      <td>18</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Tarjetas amarillas</td>
      <td>1</td>
      <td>1</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Tarjetas rojas</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>10</th>
      <td>Penaltis</td>
      <td>0</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
</div>

Vamos a dar formato a la tabla. Cuando ejecutemos el código para todos los partidos, el resultado que buscaremos será una fila por cada partido, por lo que lo que tendremos que creear una variable por cada métrica para el equipo local y otra para el equipo visitante.

Crearemos dos listas, una con los nombres de las variables y otra con los valores.

```python
variables = []
valores = []

for i in range(len(df)):    
    nombre_local = df.iloc[i]['nombre_metrica'].replace(' ', '_') + str('_local')
    variables.append(nombre_local)
    nombre_visitante = df.iloc[i]['nombre_metrica'].replace(' ', '_') + str('_visitante')
    variables.append(nombre_visitante)
        
    valor_local = df.iloc[i]['val_local']
    valores.append(valor_local)
    valor_visitante = df.iloc[i]['val_visitante']
    valores.append(valor_visitante)    
```
 Haciendo uso de estás dos listas, crearemos un diccionario en el que las claves serán igual a las diferntes variables.

```python
#diccionario para añadir los datos a la tabla
d = {}
for i in range(len(variables)):
    d[variables[i]] = valores[i]
```
Continuamos añadiendo las variables restantes (resultado, equipo, goles) a este diccionario antes de crear nuetro dataframe. 

![png](/images/futbol/Captura_goles.PNG)


```python
for j in soup2.find_all('div', {'class':'resultado'}):
    equipo_local = j.find('div', {'class': 'equipo local'})
    equipo_local = equipo_local.find('div', {'class': 'nombre'})#.replace('<div class="nombre">', '').replace('</div>', '')
    #print(equipo_local.text)
    d['equipo_local'] = equipo_local
    
    equipo_visitante = j.find('div', {'class': 'equipo visitante'})
    equipo_visitante = equipo_visitante.find('div', {'class': 'nombre'})#.replace('<div class="nombre">', '').replace('</div>', '')
    #print(equipo_visitante.text)
    d['equipo_visitante'] = equipo_visitante
    
    goles_local = j.find('div', {'class': 'local score'})#.replace('<div class="local score">', '').replace('</div>', '')
    d['goles_local'] = goles_local
    #print(goles_local.text)
    
    goles_visitante = j.find('div', {'class': 'visitante score'})#.replace('<div class="visitante score">', '').replace('</div>', '')
    d['goles_visitante'] = goles_visitante
```
Ya tenemos todas las métricas recogidas en nuestro diccionario. Ahora procedemos a crear el dataframe. Pimero crearemos un dataframe vacío que tenga por variables (columnas) los nombres de las métricas que hemos ido recogiendo 

```python
# creamos un nuevo df con los nombres de esas variables:
df_metricas = pd.DataFrame(columns = variables)
display(df_metricas)
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
      <th>Tiros_local</th>
      <th>Tiros_visitante</th>
      <th>Tiros_a_puerta_local</th>
      <th>Tiros_a_puerta_visitante</th>
      <th>Centros_local</th>
      <th>Centros_visitante</th>
      <th>Centros_precisos_local</th>
      <th>Centros_precisos_visitante</th>
      <th>Córners_local</th>
      <th>Córners_visitante</th>
      <th>...</th>
      <th>Pases_interceptados_local</th>
      <th>Pases_interceptados_visitante</th>
      <th>Balones_robados_local</th>
      <th>Balones_robados_visitante</th>
      <th>Tarjetas_amarillas_local</th>
      <th>Tarjetas_amarillas_visitante</th>
      <th>Tarjetas_rojas_local</th>
      <th>Tarjetas_rojas_visitante</th>
      <th>Penaltis_local</th>
      <th>Penaltis_visitante</th>
    </tr>
  </thead>
  <tbody>
  </tbody>
</table>
<p>0 rows × 22 columns</p>
</div>

Usamos el diccionario que hemos creado previamente para insertar nuestros valores en el dataframe

```python
df_metricas = df_metricas.append(d, ignore_index=True)
print(df_metricas)
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
      <th>Tiros_local</th>
      <th>Tiros_visitante</th>
      <th>Tiros_a_puerta_local</th>
      <th>Tiros_a_puerta_visitante</th>
      <th>Centros_local</th>
      <th>Centros_visitante</th>
      <th>Centros_precisos_local</th>
      <th>Centros_precisos_visitante</th>
      <th>Córners_local</th>
      <th>Córners_visitante</th>
      <th>...</th>
      <th>Puntos_Futmondo_Prensa_local</th>
      <th>Puntos_Futmondo_Prensa_visitante</th>
      <th>Puntos_Comunio_local</th>
      <th>Puntos_Comunio_visitante</th>
      <th>Puntos_Futmondo_Mixto_local</th>
      <th>Puntos_Futmondo_Mixto_visitante</th>
      <th>equipo_local</th>
      <th>equipo_visitante</th>
      <th>goles_local</th>
      <th>goles_visitante</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>13</td>
      <td>2</td>
      <td>1</td>
      <td>1</td>
      <td>27</td>
      <td>16</td>
      <td>7</td>
      <td>2</td>
      <td>3</td>
      <td>2</td>
      <td>...</td>
      <td>55</td>
      <td>60</td>
      <td>68</td>
      <td>52</td>
      <td>86.5</td>
      <td>60.8</td>
      <td>[Girona]</td>
      <td>[Valladolid]</td>
      <td>[0]</td>
      <td>[0]</td>
    </tr>
  </tbody>
</table>
<p>1 rows × 36 columns</p>
</div>

## Información de la temporada completa

Ahora que ya hemos visto como se extraería la información para una jornada, vamos a crear una función que nos sirva para recolectar la información de todas las jornadas

```python
def extraccion_metricas_jonada(x):
    local = []
    visitante = []
    metricas = []
    
    url_jornada = x
    ourUrl_jornada= urllib.request.urlopen(url_jornada)
    soup_jornada = BeautifulSoup(ourUrl_jornada, 'html.parser')

    for j in soup_jornada.find_all('div', attrs={'class':'stat'}):
        estadisticas_local = j.find('div', attrs = {'class': 'local'})
        if estadisticas_local != None:
            estadisticas_local = str(estadisticas_local).replace('<div class="local">', '').replace('</div>', '')
            local.append(estadisticas_local)
        
        estadisticas_visitante = j.find('div', {'class': 'visitante'})
    
        if estadisticas_visitante != None:
            estadisticas_visitante = str(estadisticas_visitante).replace('<div class="visitante">', '').replace('</div>', '')
            visitante.append(estadisticas_visitante)

        nombre_metrica = j.find('div', {'class': 'name'})
        if nombre_metrica != None:
            nombre_metrica = str(nombre_metrica).replace('<div class="name">', '').replace('</div>', '')
            metricas.append(nombre_metrica)
    
    df = pd.DataFrame(list(zip(metricas, local, visitante)), 
               columns =['nombre_metrica', 'val_local', 'val_visitante'])   
       
    for gol in soup_jornada.find_all('div', {'class':'resultado'}):
        equipo_local = gol.find('div', {'class': 'equipo local'})
        equipo_local = equipo_local.find('div', {'class': 'nombre'})#.replace('<div class="nombre">', '').replace('</div>', '')
    
        equipo_visitante = gol.find('div', {'class': 'equipo visitante'})
        equipo_visitante = equipo_visitante.find('div', {'class': 'nombre'})#.replace('<div class="nombre">', '').replace('</div>', '')
    
        goles_local = gol.find('div', {'class': 'local score'})#.replace('<div class="local score">', '').replace('</div>', '')
    
        goles_visitante = gol.find('div', {'class': 'visitante score'})#.replace('<div class="visitante score">', '').replace('</div>', '')
    
    fecha = soup_jornada.find('div', {'class':'fecha'})
    
    arbitro = soup_jornada.find('span', {'class':'link'})
    #print(x)
    #print(len(df))
    
    if len(df) == numero_metricas:
        return(df,equipo_local.text, equipo_visitante.text, goles_local.text, goles_visitante.text, fecha.text, arbitro)
         
```
Creamos una segunda función para generar el diccionario de valores:

```python
def diccionario_valores(data):
    valores = []
    variables = []
    
    for linea in range(len(data)):
        nombre_local = data.iloc[linea]['nombre_metrica'].replace(' ', '_') + str('_local')
        variables.append(nombre_local)
        nombre_visitante = data.iloc[linea]['nombre_metrica'].replace(' ', '_') + str('_visitante')
        variables.append(nombre_visitante)
        variables.append('equipo_local')
        variables.append('equipo_visitante')
        variables.append('goles_local')
        variables.append('goles_visitante')
        variables.append('fecha_jornada')
        variables.append('arbitro')       
        
        valor_local = data.iloc[linea]['val_local']
        valores.append(valor_local)
        valor_visitante = data.iloc[linea]['val_visitante']
        valores.append(valor_visitante)
        valores.append(metricas[1])
        valores.append(metricas[2])
        valores.append(metricas[3])
        valores.append(metricas[4])
        valores.append(metricas[5])
        valores.append(metricas[6])

        
    d = {} #creamos un diccionario vacío y lo rellenamos con los datos extraidos:
    for l in range(len(variables)):
        d[variables[l]] = valores[l]
    
    return(d, variables)
    
```
Por último, para evitar errores en la ejecución si alguna url está corrupta, creamos una lista con los links que están correctos:

```python
links_ok = []
links_no_ok = []

for i in lista_links_informacion:
    try:
        ourUrl_jornada= urllib.request.urlopen(i)
        links_ok.append(i)
        
    except urllib.request.HTTPError:
        links_no_ok.append(i)
        print(i)
```
Vemos que para esta jornada en cuestión todas las URLs funcionan correctamente.

```python
print(len(links_ok), len(links_no_ok))
```
    380, 0

Aplicamos las funciones creadas a nuestras URLs

```python
df_metricas = pd.DataFrame()

for i in links_ok:
    
    metricas = extraccion_metricas_jonada(i)
    diccionario = diccionario_valores(metricas[0])
      
    if df_metricas is None:
        df_metricas = pd.DataFrame(columns = diccionario[1])
        df_metricas = df_metricas.append(diccionario[0], ignore_index=True)
    
    if df_metricas is not None:
        df_metricas = df_metricas.append(diccionario[0], ignore_index=True)
```

Vamos a echar un ojo al dataframe que hemos creado. Se nos ha creado un dataframe con 380 filas (38 Jornadas x 10 partidos) y 38 columnas. 
Observamos que nuestro script no ha conseguidod obtener el nombre del árbitro en 1 de los 380 partidos.

```python
#df_metricas = df_metricas.dropna()
df_metricas.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 380 entries, 0 to 379
    Data columns (total 38 columns):
     #   Column                            Non-Null Count  Dtype 
    ---  ------                            --------------  ----- 
     0   Balones_robados_local             380 non-null    object
     1   Balones_robados_visitante         380 non-null    object
     2   Centros_local                     380 non-null    object
     3   Centros_precisos_local            380 non-null    object
     4   Centros_precisos_visitante        380 non-null    object
     5   Centros_visitante                 380 non-null    object
     6   Córners_local                     380 non-null    object
     7   Córners_visitante                 380 non-null    object
     8   Faltas_local                      380 non-null    object
     9   Faltas_visitante                  380 non-null    object
     10  Pases_interceptados_local         380 non-null    object
     11  Pases_interceptados_visitante     380 non-null    object
     12  Penaltis_local                    380 non-null    object
     13  Penaltis_visitante                380 non-null    object
     14  Puntos_Comunio_local              380 non-null    object
     15  Puntos_Comunio_visitante          380 non-null    object
     16  Puntos_Futmondo_Mixto_local       380 non-null    object
     17  Puntos_Futmondo_Mixto_visitante   380 non-null    object
     18  Puntos_Futmondo_Prensa_local      380 non-null    object
     19  Puntos_Futmondo_Prensa_visitante  380 non-null    object
     20  Puntos_Futmondo_Stats_local       380 non-null    object
     21  Puntos_Futmondo_Stats_visitante   380 non-null    object
     22  Puntos_Picas_local                380 non-null    object
     23  Puntos_Picas_visitante            380 non-null    object
     24  Tarjetas_amarillas_local          380 non-null    object
     25  Tarjetas_amarillas_visitante      380 non-null    object
     26  Tarjetas_rojas_local              380 non-null    object
     27  Tarjetas_rojas_visitante          380 non-null    object
     28  Tiros_a_puerta_local              380 non-null    object
     29  Tiros_a_puerta_visitante          380 non-null    object
     30  Tiros_local                       380 non-null    object
     31  Tiros_visitante                   380 non-null    object
     32  arbitro                           379 non-null    object
     33  equipo_local                      380 non-null    object
     34  equipo_visitante                  380 non-null    object
     35  fecha_jornada                     380 non-null    object
     36  goles_local                       380 non-null    object
     37  goles_visitante                   380 non-null    object
    dtypes: object(38)
    memory usage: 112.9+ KB
    
Echamos un vistazo a la tabla que hemos obtenido. Como podemos comprobar, el texto no ha quedado del todo limpio en las variables árbitro y fecha_jornada. 

```python
df_metricas.head()
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
      <th>Balones_robados_local</th>
      <th>Balones_robados_visitante</th>
      <th>Centros_local</th>
      <th>Centros_precisos_local</th>
      <th>Centros_precisos_visitante</th>
      <th>Centros_visitante</th>
      <th>Córners_local</th>
      <th>Córners_visitante</th>
      <th>Faltas_local</th>
      <th>Faltas_visitante</th>
      <th>...</th>
      <th>Tiros_a_puerta_local</th>
      <th>Tiros_a_puerta_visitante</th>
      <th>Tiros_local</th>
      <th>Tiros_visitante</th>
      <th>arbitro</th>
      <th>equipo_local</th>
      <th>equipo_visitante</th>
      <th>fecha_jornada</th>
      <th>goles_local</th>
      <th>goles_visitante</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>8</td>
      <td>17</td>
      <td>27</td>
      <td>7</td>
      <td>2</td>
      <td>16</td>
      <td>3</td>
      <td>2</td>
      <td>21</td>
      <td>20</td>
      <td>...</td>
      <td>1</td>
      <td>1</td>
      <td>13</td>
      <td>2</td>
      <td>[Cuadra Fernández]</td>
      <td>Girona</td>
      <td>Valladolid</td>
      <td>\n\t\t\tJornada 1. Viernes, 17 de agosto del 2...</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>13</td>
      <td>7</td>
      <td>24</td>
      <td>4</td>
      <td>1</td>
      <td>10</td>
      <td>5</td>
      <td>3</td>
      <td>10</td>
      <td>10</td>
      <td>...</td>
      <td>8</td>
      <td>4</td>
      <td>22</td>
      <td>6</td>
      <td>[Ignacio Iglesias Villanueva]</td>
      <td>Betis</td>
      <td>Levante</td>
      <td>\n\t\t\tJornada 1. Viernes, 17 de agosto del 2...</td>
      <td>0</td>
      <td>3</td>
    </tr>
    <tr>
      <th>2</th>
      <td>5</td>
      <td>11</td>
      <td>25</td>
      <td>4</td>
      <td>4</td>
      <td>20</td>
      <td>8</td>
      <td>7</td>
      <td>13</td>
      <td>14</td>
      <td>...</td>
      <td>2</td>
      <td>5</td>
      <td>12</td>
      <td>14</td>
      <td>[Santiago Jaime Latre]</td>
      <td>Celta</td>
      <td>Espanyol</td>
      <td>\n\t\t\tJornada 1. Sábado, 18 de agosto del 20...</td>
      <td>1</td>
      <td>1</td>
    </tr>
    <tr>
      <th>3</th>
      <td>9</td>
      <td>4</td>
      <td>17</td>
      <td>3</td>
      <td>5</td>
      <td>13</td>
      <td>4</td>
      <td>6</td>
      <td>16</td>
      <td>10</td>
      <td>...</td>
      <td>7</td>
      <td>4</td>
      <td>16</td>
      <td>8</td>
      <td>[Mario Melero López]</td>
      <td>Villarreal</td>
      <td>Real Sociedad</td>
      <td>\n\t\t\tJornada 1. Sábado, 18 de agosto del 20...</td>
      <td>1</td>
      <td>2</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>10</td>
      <td>10</td>
      <td>2</td>
      <td>1</td>
      <td>7</td>
      <td>7</td>
      <td>1</td>
      <td>6</td>
      <td>13</td>
      <td>...</td>
      <td>9</td>
      <td>0</td>
      <td>25</td>
      <td>3</td>
      <td>[José María Sánchez Martínez]</td>
      <td>Barcelona</td>
      <td>Alavés</td>
      <td>\n\t\t\tJornada 1. Sábado, 18 de agosto del 20...</td>
      <td>3</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 38 columns</p>
</div>

Limpiamos la variable árbitro:

```python
#df_metricas['arbitro'] =  df_metricas['arbitro'].apply(lambda x: re.escape(str(x)))
df_metricas['arbitro'] = df_metricas['arbitro'].apply(lambda x: str(x).replace('<span class="link">', '').replace('</span>', ''))
```
Dividimos la variable fecha_jornada en dos variables: fecha y jornada

```python
# separamos jornada de fecha
df_metricas['fecha_jornada'] =  df_metricas['fecha_jornada'].apply(lambda x: re.sub(r'[\n\t]','', str(x)))

fecha = df_metricas['fecha_jornada'].str.split(".", n = 1, expand = True) 
df_metricas["Jornada"]= fecha[0]
df_metricas['Jornada'] =  df_metricas['Jornada'].apply(lambda x: re.sub('Jornada ','', str(x)))

df_metricas["fecha"]= fecha[1] 
```
y a partir de la fecha generamos las variables hora, mes, año y día de la semana:
```python
dia_semana = df_metricas['fecha'].str.split(",", n = 1, expand = True) 
df_metricas["dia_semana"]= dia_semana[0] 
df_metricas["fecha"]= dia_semana[1] 
```

```python
fecha_time = df_metricas['fecha'].str.split("a las", n = 1, expand = True) 
df_metricas["fecha"]= fecha_time[0] 
df_metricas["hora"]= fecha_time[1].apply(lambda x: re.sub('h','', str(x)))
```

```python
anno = df_metricas['fecha'].str.split("del ", n = 1, expand = True) 
df_metricas["fecha"]= anno[0] 
df_metricas["anno"]= anno[1]
```

```python
dia_mes = df_metricas['fecha'].str.split(" de ", n = 1, expand = True) 
df_metricas["dia"]= dia_mes[0] 
df_metricas["mes"]= dia_mes[1]
```

La tabla final nos queda de la siguiente manera:

```python
df_metricas = df_metricas.dropna().drop(['fecha'], axis = 1)
df_metricas.head()
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
      <th>Balones_robados_local</th>
      <th>Balones_robados_visitante</th>
      <th>Centros_local</th>
      <th>Centros_precisos_local</th>
      <th>Centros_precisos_visitante</th>
      <th>Centros_visitante</th>
      <th>Córners_local</th>
      <th>Córners_visitante</th>
      <th>Faltas_local</th>
      <th>Faltas_visitante</th>
      <th>...</th>
      <th>equipo_visitante</th>
      <th>fecha_jornada</th>
      <th>goles_local</th>
      <th>goles_visitante</th>
      <th>Jornada</th>
      <th>dia_semana</th>
      <th>hora</th>
      <th>anno</th>
      <th>dia</th>
      <th>mes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>8</td>
      <td>17</td>
      <td>27</td>
      <td>7</td>
      <td>2</td>
      <td>16</td>
      <td>3</td>
      <td>2</td>
      <td>21</td>
      <td>20</td>
      <td>...</td>
      <td>Valladolid</td>
      <td>Jornada 1. Viernes, 17 de agosto del 2018 a la...</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>Viernes</td>
      <td>20:15</td>
      <td>2018</td>
      <td>17</td>
      <td>agosto</td>
    </tr>
    <tr>
      <th>1</th>
      <td>13</td>
      <td>7</td>
      <td>24</td>
      <td>4</td>
      <td>1</td>
      <td>10</td>
      <td>5</td>
      <td>3</td>
      <td>10</td>
      <td>10</td>
      <td>...</td>
      <td>Levante</td>
      <td>Jornada 1. Viernes, 17 de agosto del 2018 a la...</td>
      <td>0</td>
      <td>3</td>
      <td>1</td>
      <td>Viernes</td>
      <td>22:15</td>
      <td>2018</td>
      <td>17</td>
      <td>agosto</td>
    </tr>
    <tr>
      <th>2</th>
      <td>5</td>
      <td>11</td>
      <td>25</td>
      <td>4</td>
      <td>4</td>
      <td>20</td>
      <td>8</td>
      <td>7</td>
      <td>13</td>
      <td>14</td>
      <td>...</td>
      <td>Espanyol</td>
      <td>Jornada 1. Sábado, 18 de agosto del 2018 a las...</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>Sábado</td>
      <td>18:15</td>
      <td>2018</td>
      <td>18</td>
      <td>agosto</td>
    </tr>
    <tr>
      <th>3</th>
      <td>9</td>
      <td>4</td>
      <td>17</td>
      <td>3</td>
      <td>5</td>
      <td>13</td>
      <td>4</td>
      <td>6</td>
      <td>16</td>
      <td>10</td>
      <td>...</td>
      <td>Real Sociedad</td>
      <td>Jornada 1. Sábado, 18 de agosto del 2018 a las...</td>
      <td>1</td>
      <td>2</td>
      <td>1</td>
      <td>Sábado</td>
      <td>20:15</td>
      <td>2018</td>
      <td>18</td>
      <td>agosto</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>10</td>
      <td>10</td>
      <td>2</td>
      <td>1</td>
      <td>7</td>
      <td>7</td>
      <td>1</td>
      <td>6</td>
      <td>13</td>
      <td>...</td>
      <td>Alavés</td>
      <td>Jornada 1. Sábado, 18 de agosto del 2018 a las...</td>
      <td>3</td>
      <td>0</td>
      <td>1</td>
      <td>Sábado</td>
      <td>22:15</td>
      <td>2018</td>
      <td>18</td>
      <td>agosto</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 44 columns</p>
</div>


