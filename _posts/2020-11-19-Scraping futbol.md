<style>
div { 
  font-family:"Arial";
  font-size: 18px;
  }
</style>


# Web scraping datos de futbol con BeautifulSoup

## Indice:
<ul >
<li>  Objetivo.  </li>
<li>  Librerías necesarias. </li>  
<li>  Código fuente de la página. </li>  
<li>  Extracción URLs de cada partido </li>
<li> Información de una jornada </li>
</ul>


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

Lo primero que tenemos que indicar es de que temporada queremos extraer los datos. Para este post hemos elegido 2014.

Como hemos comentado, no todas las temporadas tienen disponible el mismo número de métricas, por lo que incluiremos un loop en el que indicaremos al programa el número de métricas disponibles según la temporada que estemos analizando.

```python
temporada = 2014

numero_metricas = 0
if temporada > 2017:
    numero_metricas = 16
if temporada <= 2017 and temporada > 2015:
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

Si miramos el código fuente de la página, las urls que buscamos están en bajo la etiquita <a> con el atributo 'href'. A su vez, esta etiqueta está bajo  la tiqueta de agrupación div con atributo class iguala 'jornada:

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

Igual que hicimos para sacar las url, vamos mirando en el código funte de la página las étiquetas y atributos que nos ayudan a encontrar nuestros datos.

**Información del árbitro:**

![png](/images/futbol/Captura_arbitro.PNG)

```python
arbitro = soup2.find_all('span', {'class':'link'})
```

```python
print(arbitro)
```

    [<span class="link">Juan Martínez Munuera</span>]
    
----------------------------------------- PENDIENTE DESDE ESTE PUNTO ---------------------------------------------

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


```python
df = pd.DataFrame(list(zip(metricas, local, visitante)), 
               columns =['nombre_metrica', 'val_local', 'val_visitante']) 

```


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


```python
#diccionario para añadir los datos a la tabla

d = {}
for i in range(len(variables)):
    d[variables[i]] = valores[i]

```


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



```python
#df_metricas = df_metricas.append(d, ignore_index=True)
```


```python
### ahora tenemos que popularizar esto para todas las jornadas.
### pendiente añadir la jornada también
```


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


```python
#print(links_ok)
```


```python
#print(links_no_ok)
```


```python
jornadas_ok = 380 - len(links_no_ok)
len(links_ok[0:jornadas_ok])

```




    380




```python
df_metricas = pd.DataFrame()

for i in links_ok[0:jornadas_ok]:
    print(i)
    #print(metricas)
    
    metricas = extraccion_metricas_jonada(i)
    diccionario = diccionario_valores(metricas[0])
      
    if df_metricas is None:
        df_metricas = pd.DataFrame(columns = diccionario[1])
        df_metricas = df_metricas.append(diccionario[0], ignore_index=True)
    
    if df_metricas is not None:
        df_metricas = df_metricas.append(diccionario[0], ignore_index=True)
```

    https://www.futbolfantasy.com/partidos/1019-real-sociedad-getafe
    https://www.futbolfantasy.com/partidos/1020-valladolid-athletic
    https://www.futbolfantasy.com/partidos/1016-valencia-malaga
    https://www.futbolfantasy.com/partidos/1015-barcelona-levante
    https://www.futbolfantasy.com/partidos/1017-real-madrid-betis
    https://www.futbolfantasy.com/partidos/1021-osasuna-granada
    https://www.futbolfantasy.com/partidos/1018-sevilla-atletico
    https://www.futbolfantasy.com/partidos/1024-rayo-elche
    https://www.futbolfantasy.com/partidos/1022-celta-espanyol
    https://www.futbolfantasy.com/partidos/1023-almeria-villarreal
    https://www.futbolfantasy.com/partidos/1030-getafe-almeria
    https://www.futbolfantasy.com/partidos/1028-athletic-osasuna
    https://www.futbolfantasy.com/partidos/1031-elche-real-sociedad
    https://www.futbolfantasy.com/partidos/1025-espanyol-valencia
    https://www.futbolfantasy.com/partidos/1029-villarreal-valladolid
    https://www.futbolfantasy.com/partidos/1032-atletico-rayo
    https://www.futbolfantasy.com/partidos/1033-levante-sevilla
    https://www.futbolfantasy.com/partidos/1034-malaga-barcelona
    https://www.futbolfantasy.com/partidos/1026-betis-celta
    https://www.futbolfantasy.com/partidos/1027-granada-real-madrid
    https://www.futbolfantasy.com/partidos/1037-almeria-elche
    https://www.futbolfantasy.com/partidos/1039-rayo-levante
    https://www.futbolfantasy.com/partidos/1043-celta-granada
    https://www.futbolfantasy.com/partidos/1036-valladolid-getafe
    https://www.futbolfantasy.com/partidos/1044-osasuna-villarreal
    https://www.futbolfantasy.com/partidos/1035-real-madrid-athletic
    https://www.futbolfantasy.com/partidos/1042-espanyol-betis
    https://www.futbolfantasy.com/partidos/1038-real-sociedad-atletico
    https://www.futbolfantasy.com/partidos/1040-sevilla-malaga
    https://www.futbolfantasy.com/partidos/1041-valencia-barcelona
    https://www.futbolfantasy.com/partidos/1051-atletico-almeria
    https://www.futbolfantasy.com/partidos/1052-levante-real-sociedad
    https://www.futbolfantasy.com/partidos/1053-barcelona-sevilla
    https://www.futbolfantasy.com/partidos/1048-villarreal-real-madrid
    https://www.futbolfantasy.com/partidos/1046-granada-espanyol
    https://www.futbolfantasy.com/partidos/1049-getafe-osasuna
    https://www.futbolfantasy.com/partidos/1054-malaga-rayo
    https://www.futbolfantasy.com/partidos/1045-betis-valencia
    https://www.futbolfantasy.com/partidos/1050-elche-valladolid
    https://www.futbolfantasy.com/partidos/1047-athletic-celta
    https://www.futbolfantasy.com/partidos/1058-osasuna-elche
    https://www.futbolfantasy.com/partidos/1062-real-sociedad-malaga
    https://www.futbolfantasy.com/partidos/1061-almeria-levante
    https://www.futbolfantasy.com/partidos/1063-rayo-barcelona
    https://www.futbolfantasy.com/partidos/1060-valladolid-atletico
    https://www.futbolfantasy.com/partidos/1059-betis-granada
    https://www.futbolfantasy.com/partidos/1056-celta-villarreal
    https://www.futbolfantasy.com/partidos/1057-real-madrid-getafe
    https://www.futbolfantasy.com/partidos/1064-valencia-sevilla
    https://www.futbolfantasy.com/partidos/1055-espanyol-athletic
    https://www.futbolfantasy.com/partidos/1077-barcelona-real-sociedad
    https://www.futbolfantasy.com/partidos/1083-levante-valladolid
    https://www.futbolfantasy.com/partidos/1076-malaga-almeria
    https://www.futbolfantasy.com/partidos/1078-atletico-osasuna
    https://www.futbolfantasy.com/partidos/1079-granada-valencia
    https://www.futbolfantasy.com/partidos/1084-sevilla-rayo
    https://www.futbolfantasy.com/partidos/1082-elche-real-madrid
    https://www.futbolfantasy.com/partidos/1080-athletic-betis
    https://www.futbolfantasy.com/partidos/1075-getafe-celta
    https://www.futbolfantasy.com/partidos/1081-villarreal-espanyol
    https://www.futbolfantasy.com/partidos/1092-valladolid-malaga
    https://www.futbolfantasy.com/partidos/1094-valencia-rayo
    https://www.futbolfantasy.com/partidos/1087-almeria-barcelona
    https://www.futbolfantasy.com/partidos/1093-real-sociedad-sevilla
    https://www.futbolfantasy.com/partidos/1090-real-madrid-atletico
    https://www.futbolfantasy.com/partidos/1091-osasuna-levante
    https://www.futbolfantasy.com/partidos/1085-celta-elche
    https://www.futbolfantasy.com/partidos/1089-espanyol-getafe
    https://www.futbolfantasy.com/partidos/1088-betis-villarreal
    https://www.futbolfantasy.com/partidos/1086-granada-athletic
    https://www.futbolfantasy.com/partidos/1100-villarreal-granada
    https://www.futbolfantasy.com/partidos/1097-malaga-osasuna
    https://www.futbolfantasy.com/partidos/1101-elche-espanyol
    https://www.futbolfantasy.com/partidos/1104-rayo-real-sociedad
    https://www.futbolfantasy.com/partidos/1098-levante-real-madrid
    https://www.futbolfantasy.com/partidos/1102-barcelona-valladolid
    https://www.futbolfantasy.com/partidos/1096-atletico-celta
    https://www.futbolfantasy.com/partidos/1103-sevilla-almeria
    https://www.futbolfantasy.com/partidos/1095-getafe-betis
    https://www.futbolfantasy.com/partidos/1099-athletic-valencia
    https://www.futbolfantasy.com/partidos/1153-real-madrid-malaga
    https://www.futbolfantasy.com/partidos/1156-valencia-real-sociedad
    https://www.futbolfantasy.com/partidos/1154-osasuna-barcelona
    https://www.futbolfantasy.com/partidos/1150-espanyol-atletico
    https://www.futbolfantasy.com/partidos/1148-granada-getafe
    https://www.futbolfantasy.com/partidos/1151-almeria-rayo
    https://www.futbolfantasy.com/partidos/1149-betis-elche
    https://www.futbolfantasy.com/partidos/1155-valladolid-sevilla
    https://www.futbolfantasy.com/partidos/1152-celta-levante
    https://www.futbolfantasy.com/partidos/1147-athletic-villarreal
    https://www.futbolfantasy.com/partidos/1165-sevilla-osasuna
    https://www.futbolfantasy.com/partidos/1164-malaga-celta
    https://www.futbolfantasy.com/partidos/1157-villarreal-valencia
    https://www.futbolfantasy.com/partidos/1161-barcelona-real-madrid
    https://www.futbolfantasy.com/partidos/1160-real-sociedad-almeria
    https://www.futbolfantasy.com/partidos/1159-elche-granada
    https://www.futbolfantasy.com/partidos/1162-atletico-betis
    https://www.futbolfantasy.com/partidos/1166-rayo-valladolid
    https://www.futbolfantasy.com/partidos/1158-getafe-athletic
    https://www.futbolfantasy.com/partidos/1163-levante-espanyol
    https://www.futbolfantasy.com/partidos/1169-espanyol-malaga
    https://www.futbolfantasy.com/partidos/1170-valladolid-real-sociedad
    https://www.futbolfantasy.com/partidos/1171-villarreal-getafe
    https://www.futbolfantasy.com/partidos/1172-granada-atletico
    https://www.futbolfantasy.com/partidos/1176-valencia-almeria
    https://www.futbolfantasy.com/partidos/1167-athletic-elche
    https://www.futbolfantasy.com/partidos/1168-betis-levante
    https://www.futbolfantasy.com/partidos/1173-celta-barcelona
    https://www.futbolfantasy.com/partidos/1174-real-madrid-sevilla
    https://www.futbolfantasy.com/partidos/1175-osasuna-rayo
    https://www.futbolfantasy.com/partidos/1177-getafe-valencia
    https://www.futbolfantasy.com/partidos/1179-real-sociedad-osasuna
    https://www.futbolfantasy.com/partidos/1182-atletico-athletic
    https://www.futbolfantasy.com/partidos/1180-almeria-valladolid
    https://www.futbolfantasy.com/partidos/1183-levante-granada
    https://www.futbolfantasy.com/partidos/1186-rayo-real-madrid
    https://www.futbolfantasy.com/partidos/1178-malaga-betis
    https://www.futbolfantasy.com/partidos/1184-barcelona-espanyol
    https://www.futbolfantasy.com/partidos/1181-elche-villarreal
    https://www.futbolfantasy.com/partidos/1185-sevilla-celta
    https://www.futbolfantasy.com/partidos/1196-espanyol-sevilla
    https://www.futbolfantasy.com/partidos/1188-real-madrid-real-sociedad
    https://www.futbolfantasy.com/partidos/1190-valencia-valladolid
    https://www.futbolfantasy.com/partidos/1191-getafe-elche
    https://www.futbolfantasy.com/partidos/1192-villarreal-atletico
    https://www.futbolfantasy.com/partidos/1189-osasuna-almeria
    https://www.futbolfantasy.com/partidos/1193-athletic-levante
    https://www.futbolfantasy.com/partidos/1194-betis-barcelona
    https://www.futbolfantasy.com/partidos/1187-granada-malaga
    https://www.futbolfantasy.com/partidos/1195-celta-rayo
    https://www.futbolfantasy.com/partidos/1202-levante-villarreal
    https://www.futbolfantasy.com/partidos/1199-barcelona-granada
    https://www.futbolfantasy.com/partidos/1204-rayo-espanyol
    https://www.futbolfantasy.com/partidos/1205-real-sociedad-celta
    https://www.futbolfantasy.com/partidos/1197-elche-valencia
    https://www.futbolfantasy.com/partidos/1200-almeria-real-madrid
    https://www.futbolfantasy.com/partidos/1203-sevilla-betis
    https://www.futbolfantasy.com/partidos/1206-valladolid-osasuna
    https://www.futbolfantasy.com/partidos/1198-malaga-athletic
    https://www.futbolfantasy.com/partidos/1201-atletico-getafe
    https://www.futbolfantasy.com/partidos/1215-betis-rayo
    https://www.futbolfantasy.com/partidos/1207-elche-atletico
    https://www.futbolfantasy.com/partidos/1208-granada-sevilla
    https://www.futbolfantasy.com/partidos/1216-celta-almeria
    https://www.futbolfantasy.com/partidos/1211-valencia-osasuna
    https://www.futbolfantasy.com/partidos/1210-real-madrid-valladolid
    https://www.futbolfantasy.com/partidos/1212-getafe-levante
    https://www.futbolfantasy.com/partidos/1213-villarreal-malaga
    https://www.futbolfantasy.com/partidos/1214-athletic-barcelona
    https://www.futbolfantasy.com/partidos/1209-espanyol-real-sociedad
    https://www.futbolfantasy.com/partidos/1232-almeria-espanyol
    https://www.futbolfantasy.com/partidos/1228-osasuna-real-madrid
    https://www.futbolfantasy.com/partidos/1233-real-sociedad-betis
    https://www.futbolfantasy.com/partidos/1229-rayo-granada
    https://www.futbolfantasy.com/partidos/1234-sevilla-athletic
    https://www.futbolfantasy.com/partidos/1230-barcelona-villarreal
    https://www.futbolfantasy.com/partidos/1227-levante-elche
    https://www.futbolfantasy.com/partidos/1235-atletico-valencia
    https://www.futbolfantasy.com/partidos/1231-malaga-getafe
    https://www.futbolfantasy.com/partidos/1236-valladolid-celta
    https://www.futbolfantasy.com/partidos/1242-espanyol-valladolid
    https://www.futbolfantasy.com/partidos/1238-villarreal-sevilla
    https://www.futbolfantasy.com/partidos/1243-getafe-barcelona
    https://www.futbolfantasy.com/partidos/1239-betis-almeria
    https://www.futbolfantasy.com/partidos/1244-athletic-rayo
    https://www.futbolfantasy.com/partidos/1240-atletico-levante
    https://www.futbolfantasy.com/partidos/1237-elche-malaga
    https://www.futbolfantasy.com/partidos/1245-valencia-real-madrid
    https://www.futbolfantasy.com/partidos/1246-celta-osasuna
    https://www.futbolfantasy.com/partidos/1241-granada-real-sociedad
    https://www.futbolfantasy.com/partidos/1267-malaga-atletico
    https://www.futbolfantasy.com/partidos/1268-valladolid-betis
    https://www.futbolfantasy.com/partidos/1269-valencia-levante
    https://www.futbolfantasy.com/partidos/1270-almeria-granada
    https://www.futbolfantasy.com/partidos/1271-sevilla-getafe
    https://www.futbolfantasy.com/partidos/1272-barcelona-elche
    https://www.futbolfantasy.com/partidos/1273-osasuna-espanyol
    https://www.futbolfantasy.com/partidos/1274-real-sociedad-athletic
    https://www.futbolfantasy.com/partidos/1275-real-madrid-celta
    https://www.futbolfantasy.com/partidos/1276-rayo-villarreal
    https://www.futbolfantasy.com/partidos/1277-granada-valladolid
    https://www.futbolfantasy.com/partidos/1278-athletic-almeria
    https://www.futbolfantasy.com/partidos/1279-celta-valencia
    https://www.futbolfantasy.com/partidos/1280-atletico-barcelona
    https://www.futbolfantasy.com/partidos/1281-elche-sevilla
    https://www.futbolfantasy.com/partidos/1282-getafe-rayo
    https://www.futbolfantasy.com/partidos/1283-betis-osasuna
    https://www.futbolfantasy.com/partidos/1284-espanyol-real-madrid
    https://www.futbolfantasy.com/partidos/1285-levante-malaga
    https://www.futbolfantasy.com/partidos/1286-villarreal-real-sociedad
    https://www.futbolfantasy.com/partidos/1287-malaga-valencia
    https://www.futbolfantasy.com/partidos/1288-betis-real-madrid
    https://www.futbolfantasy.com/partidos/1289-elche-rayo
    https://www.futbolfantasy.com/partidos/1290-granada-osasuna
    https://www.futbolfantasy.com/partidos/1291-espanyol-celta
    https://www.futbolfantasy.com/partidos/1292-getafe-real-sociedad
    https://www.futbolfantasy.com/partidos/1293-villarreal-almeria
    https://www.futbolfantasy.com/partidos/1294-levante-barcelona
    https://www.futbolfantasy.com/partidos/1295-atletico-sevilla
    https://www.futbolfantasy.com/partidos/1296-athletic-valladolid
    https://www.futbolfantasy.com/partidos/1218-celta-betis
    https://www.futbolfantasy.com/partidos/1219-real-madrid-granada
    https://www.futbolfantasy.com/partidos/1221-valladolid-villarreal
    https://www.futbolfantasy.com/partidos/1217-valencia-espanyol
    https://www.futbolfantasy.com/partidos/1225-sevilla-levante
    https://www.futbolfantasy.com/partidos/1222-almeria-getafe
    https://www.futbolfantasy.com/partidos/1220-osasuna-athletic
    https://www.futbolfantasy.com/partidos/1224-rayo-atletico
    https://www.futbolfantasy.com/partidos/1226-barcelona-malaga
    https://www.futbolfantasy.com/partidos/1223-real-sociedad-elche
    https://www.futbolfantasy.com/partidos/1305-granada-celta
    https://www.futbolfantasy.com/partidos/1303-barcelona-valencia
    https://www.futbolfantasy.com/partidos/1301-levante-rayo
    https://www.futbolfantasy.com/partidos/1298-getafe-valladolid
    https://www.futbolfantasy.com/partidos/1302-malaga-sevilla
    https://www.futbolfantasy.com/partidos/1299-elche-almeria
    https://www.futbolfantasy.com/partidos/1304-betis-espanyol
    https://www.futbolfantasy.com/partidos/1300-atletico-real-sociedad
    https://www.futbolfantasy.com/partidos/1297-athletic-real-madrid
    https://www.futbolfantasy.com/partidos/1306-villarreal-osasuna
    https://www.futbolfantasy.com/partidos/1308-espanyol-granada
    https://www.futbolfantasy.com/partidos/1307-valencia-betis
    https://www.futbolfantasy.com/partidos/1316-rayo-malaga
    https://www.futbolfantasy.com/partidos/1310-real-madrid-villarreal
    https://www.futbolfantasy.com/partidos/1313-almeria-atletico
    https://www.futbolfantasy.com/partidos/1311-osasuna-getafe
    https://www.futbolfantasy.com/partidos/1312-valladolid-elche
    https://www.futbolfantasy.com/partidos/1314-real-sociedad-levante
    https://www.futbolfantasy.com/partidos/1315-sevilla-barcelona
    https://www.futbolfantasy.com/partidos/1309-celta-athletic
    https://www.futbolfantasy.com/partidos/1320-elche-osasuna
    https://www.futbolfantasy.com/partidos/1322-atletico-valladolid
    https://www.futbolfantasy.com/partidos/1323-levante-almeria
    https://www.futbolfantasy.com/partidos/1325-barcelona-rayo
    https://www.futbolfantasy.com/partidos/1318-villarreal-celta
    https://www.futbolfantasy.com/partidos/1321-granada-betis
    https://www.futbolfantasy.com/partidos/1319-getafe-real-madrid
    https://www.futbolfantasy.com/partidos/1317-athletic-espanyol
    https://www.futbolfantasy.com/partidos/1326-sevilla-valencia
    https://www.futbolfantasy.com/partidos/1324-malaga-real-sociedad
    https://www.futbolfantasy.com/partidos/1335-valladolid-levante
    https://www.futbolfantasy.com/partidos/1334-real-madrid-elche
    https://www.futbolfantasy.com/partidos/1327-celta-getafe
    https://www.futbolfantasy.com/partidos/1329-real-sociedad-barcelona
    https://www.futbolfantasy.com/partidos/1328-almeria-malaga
    https://www.futbolfantasy.com/partidos/1336-rayo-sevilla
    https://www.futbolfantasy.com/partidos/1332-betis-athletic
    https://www.futbolfantasy.com/partidos/1331-valencia-granada
    https://www.futbolfantasy.com/partidos/1330-osasuna-atletico
    https://www.futbolfantasy.com/partidos/1333-espanyol-villarreal
    https://www.futbolfantasy.com/partidos/1338-athletic-granada
    https://www.futbolfantasy.com/partidos/1344-malaga-valladolid
    https://www.futbolfantasy.com/partidos/1343-levante-osasuna
    https://www.futbolfantasy.com/partidos/1341-getafe-espanyol
    https://www.futbolfantasy.com/partidos/1337-elche-celta
    https://www.futbolfantasy.com/partidos/1340-villarreal-betis
    https://www.futbolfantasy.com/partidos/1342-atletico-real-madrid
    https://www.futbolfantasy.com/partidos/1345-sevilla-real-sociedad
    https://www.futbolfantasy.com/partidos/1339-barcelona-almeria
    https://www.futbolfantasy.com/partidos/1346-rayo-valencia
    https://www.futbolfantasy.com/partidos/1354-valladolid-barcelona
    https://www.futbolfantasy.com/partidos/1347-betis-getafe
    https://www.futbolfantasy.com/partidos/1348-celta-atletico
    https://www.futbolfantasy.com/partidos/1352-granada-villarreal
    https://www.futbolfantasy.com/partidos/1353-espanyol-elche
    https://www.futbolfantasy.com/partidos/1355-almeria-sevilla
    https://www.futbolfantasy.com/partidos/1350-real-madrid-levante
    https://www.futbolfantasy.com/partidos/1351-valencia-athletic
    https://www.futbolfantasy.com/partidos/1349-osasuna-malaga
    https://www.futbolfantasy.com/partidos/1356-real-sociedad-rayo
    https://www.futbolfantasy.com/partidos/1358-getafe-granada
    https://www.futbolfantasy.com/partidos/1362-levante-celta
    https://www.futbolfantasy.com/partidos/1361-rayo-almeria
    https://www.futbolfantasy.com/partidos/1363-malaga-real-madrid
    https://www.futbolfantasy.com/partidos/1360-atletico-espanyol
    https://www.futbolfantasy.com/partidos/1359-elche-betis
    https://www.futbolfantasy.com/partidos/1364-barcelona-osasuna
    https://www.futbolfantasy.com/partidos/1365-sevilla-valladolid
    https://www.futbolfantasy.com/partidos/1366-real-sociedad-valencia
    https://www.futbolfantasy.com/partidos/1357-villarreal-athletic
    https://www.futbolfantasy.com/partidos/1374-celta-malaga
    https://www.futbolfantasy.com/partidos/1369-granada-elche
    https://www.futbolfantasy.com/partidos/1373-espanyol-levante
    https://www.futbolfantasy.com/partidos/1376-valladolid-rayo
    https://www.futbolfantasy.com/partidos/1368-athletic-getafe
    https://www.futbolfantasy.com/partidos/1375-osasuna-sevilla
    https://www.futbolfantasy.com/partidos/1372-betis-atletico
    https://www.futbolfantasy.com/partidos/1367-valencia-villarreal
    https://www.futbolfantasy.com/partidos/1371-real-madrid-barcelona
    https://www.futbolfantasy.com/partidos/1370-almeria-real-sociedad
    https://www.futbolfantasy.com/partidos/1379-malaga-espanyol
    https://www.futbolfantasy.com/partidos/1377-elche-athletic
    https://www.futbolfantasy.com/partidos/1383-barcelona-celta
    https://www.futbolfantasy.com/partidos/1385-rayo-osasuna
    https://www.futbolfantasy.com/partidos/1382-atletico-granada
    https://www.futbolfantasy.com/partidos/1384-sevilla-real-madrid
    https://www.futbolfantasy.com/partidos/1380-real-sociedad-valladolid
    https://www.futbolfantasy.com/partidos/1381-getafe-villarreal
    https://www.futbolfantasy.com/partidos/1378-levante-betis
    https://www.futbolfantasy.com/partidos/1386-almeria-valencia
    https://www.futbolfantasy.com/partidos/1394-espanyol-barcelona
    https://www.futbolfantasy.com/partidos/1395-celta-sevilla
    https://www.futbolfantasy.com/partidos/1392-athletic-atletico
    https://www.futbolfantasy.com/partidos/1396-real-madrid-rayo
    https://www.futbolfantasy.com/partidos/1390-valladolid-almeria
    https://www.futbolfantasy.com/partidos/1389-osasuna-real-sociedad
    https://www.futbolfantasy.com/partidos/1391-villarreal-elche
    https://www.futbolfantasy.com/partidos/1387-valencia-getafe
    https://www.futbolfantasy.com/partidos/1393-granada-levante
    https://www.futbolfantasy.com/partidos/1388-betis-malaga
    https://www.futbolfantasy.com/partidos/1399-almeria-osasuna
    https://www.futbolfantasy.com/partidos/1402-atletico-villarreal
    https://www.futbolfantasy.com/partidos/1404-barcelona-betis
    https://www.futbolfantasy.com/partidos/1398-real-sociedad-real-madrid
    https://www.futbolfantasy.com/partidos/1405-rayo-celta
    https://www.futbolfantasy.com/partidos/1397-malaga-granada
    https://www.futbolfantasy.com/partidos/1401-elche-getafe
    https://www.futbolfantasy.com/partidos/1406-sevilla-espanyol
    https://www.futbolfantasy.com/partidos/1400-valladolid-valencia
    https://www.futbolfantasy.com/partidos/1403-levante-athletic
    https://www.futbolfantasy.com/partidos/1416-osasuna-valladolid
    https://www.futbolfantasy.com/partidos/1415-celta-real-sociedad
    https://www.futbolfantasy.com/partidos/1412-villarreal-levante
    https://www.futbolfantasy.com/partidos/1409-granada-barcelona
    https://www.futbolfantasy.com/partidos/1410-real-madrid-almeria
    https://www.futbolfantasy.com/partidos/1413-betis-sevilla
    https://www.futbolfantasy.com/partidos/1407-valencia-elche
    https://www.futbolfantasy.com/partidos/1411-getafe-atletico
    https://www.futbolfantasy.com/partidos/1414-espanyol-rayo
    https://www.futbolfantasy.com/partidos/1408-athletic-malaga
    https://www.futbolfantasy.com/partidos/1417-atletico-elche
    https://www.futbolfantasy.com/partidos/1421-osasuna-valencia
    https://www.futbolfantasy.com/partidos/1422-levante-getafe
    https://www.futbolfantasy.com/partidos/1419-real-sociedad-espanyol
    https://www.futbolfantasy.com/partidos/1426-almeria-celta
    https://www.futbolfantasy.com/partidos/1425-rayo-betis
    https://www.futbolfantasy.com/partidos/1418-sevilla-granada
    https://www.futbolfantasy.com/partidos/1424-barcelona-athletic
    https://www.futbolfantasy.com/partidos/1423-malaga-villarreal
    https://www.futbolfantasy.com/partidos/1420-valladolid-real-madrid
    https://www.futbolfantasy.com/partidos/1427-elche-levante
    https://www.futbolfantasy.com/partidos/1432-granada-rayo
    https://www.futbolfantasy.com/partidos/1429-getafe-malaga
    https://www.futbolfantasy.com/partidos/1436-real-madrid-osasuna
    https://www.futbolfantasy.com/partidos/1433-betis-real-sociedad
    https://www.futbolfantasy.com/partidos/1434-espanyol-almeria
    https://www.futbolfantasy.com/partidos/1428-valencia-atletico
    https://www.futbolfantasy.com/partidos/1431-athletic-sevilla
    https://www.futbolfantasy.com/partidos/1430-villarreal-barcelona
    https://www.futbolfantasy.com/partidos/1435-celta-valladolid
    https://www.futbolfantasy.com/partidos/1441-rayo-athletic
    https://www.futbolfantasy.com/partidos/1439-barcelona-getafe
    https://www.futbolfantasy.com/partidos/1438-malaga-elche
    https://www.futbolfantasy.com/partidos/1445-osasuna-celta
    https://www.futbolfantasy.com/partidos/1444-valladolid-espanyol
    https://www.futbolfantasy.com/partidos/1443-almeria-betis
    https://www.futbolfantasy.com/partidos/1437-levante-atletico
    https://www.futbolfantasy.com/partidos/1440-sevilla-villarreal
    https://www.futbolfantasy.com/partidos/1446-real-madrid-valencia
    https://www.futbolfantasy.com/partidos/1442-real-sociedad-granada
    https://www.futbolfantasy.com/partidos/1451-villarreal-rayo
    https://www.futbolfantasy.com/partidos/1447-levante-valencia
    https://www.futbolfantasy.com/partidos/1452-athletic-real-sociedad
    https://www.futbolfantasy.com/partidos/1448-atletico-malaga
    https://www.futbolfantasy.com/partidos/1449-elche-barcelona
    https://www.futbolfantasy.com/partidos/1450-getafe-sevilla
    https://www.futbolfantasy.com/partidos/1453-granada-almeria
    https://www.futbolfantasy.com/partidos/1454-betis-valladolid
    https://www.futbolfantasy.com/partidos/1455-espanyol-osasuna
    https://www.futbolfantasy.com/partidos/1456-celta-real-madrid
    https://www.futbolfantasy.com/partidos/1465-malaga-levante
    https://www.futbolfantasy.com/partidos/1458-real-madrid-espanyol
    https://www.futbolfantasy.com/partidos/1457-barcelona-atletico
    https://www.futbolfantasy.com/partidos/1464-valencia-celta
    https://www.futbolfantasy.com/partidos/1463-real-sociedad-villarreal
    https://www.futbolfantasy.com/partidos/1459-rayo-getafe
    https://www.futbolfantasy.com/partidos/1460-osasuna-betis
    https://www.futbolfantasy.com/partidos/1461-almeria-athletic
    https://www.futbolfantasy.com/partidos/1466-valladolid-granada
    https://www.futbolfantasy.com/partidos/1462-sevilla-elche
    


```python
print(len(links_ok))
print(len(links_no_ok))
```

    431
    0
    


```python
len(df_metricas)

```




    380




```python
#df_metricas = df_metricas.dropna()
df_metricas.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 380 entries, 0 to 379
    Data columns (total 28 columns):
     #   Column                         Non-Null Count  Dtype 
    ---  ------                         --------------  ----- 
     0   Balones_robados_local          380 non-null    object
     1   Balones_robados_visitante      380 non-null    object
     2   Centros_local                  380 non-null    object
     3   Centros_precisos_local         380 non-null    object
     4   Centros_precisos_visitante     380 non-null    object
     5   Centros_visitante              380 non-null    object
     6   Córners_local                  380 non-null    object
     7   Córners_visitante              380 non-null    object
     8   Faltas_local                   380 non-null    object
     9   Faltas_visitante               380 non-null    object
     10  Pases_interceptados_local      380 non-null    object
     11  Pases_interceptados_visitante  380 non-null    object
     12  Penaltis_local                 380 non-null    object
     13  Penaltis_visitante             380 non-null    object
     14  Tarjetas_amarillas_local       380 non-null    object
     15  Tarjetas_amarillas_visitante   380 non-null    object
     16  Tarjetas_rojas_local           380 non-null    object
     17  Tarjetas_rojas_visitante       380 non-null    object
     18  Tiros_a_puerta_local           380 non-null    object
     19  Tiros_a_puerta_visitante       380 non-null    object
     20  Tiros_local                    380 non-null    object
     21  Tiros_visitante                380 non-null    object
     22  arbitro                        378 non-null    object
     23  equipo_local                   380 non-null    object
     24  equipo_visitante               380 non-null    object
     25  fecha_jornada                  380 non-null    object
     26  goles_local                    380 non-null    object
     27  goles_visitante                380 non-null    object
    dtypes: object(28)
    memory usage: 83.2+ KB
    


```python
#df_metricas['arbitro'] =  df_metricas['arbitro'].apply(lambda x: re.escape(str(x)))
df_metricas['arbitro'] = df_metricas['arbitro'].apply(lambda x: str(x).replace('<span class="link">', '').replace('</span>', ''))
```


```python
# separamos jornada de fecha
df_metricas['fecha_jornada'] =  df_metricas['fecha_jornada'].apply(lambda x: re.sub(r'[\n\t]','', str(x)))

fecha = df_metricas['fecha_jornada'].str.split(".", n = 1, expand = True) 
df_metricas["Jornada"]= fecha[0]
df_metricas['Jornada'] =  df_metricas['Jornada'].apply(lambda x: re.sub('Jornada ','', str(x)))

df_metricas["fecha"]= fecha[1] 
```


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


```python
df_metricas = df_metricas.dropna().drop(['fecha'], axis = 1)
#display(df_metricas)
```


```python
df_metricas.to_excel('C:/Users/Raquel/Documents/Futbol/datos_futbol.xlsx',
                    index = False)
```


```python

```


```python
lista_links_informacion_df = pd.DataFrame( links_ok[0:jornadas_ok], columns = ['links'])
len(lista_links_informacion_df)
```




    380




```python
lista_links_informacion_df.to_excel('C:/Users/Raquel/Documents/Futbol/links.xlsx',
                    index = False)
```
