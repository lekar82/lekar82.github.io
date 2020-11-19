<style>
div { 
  font-family:"Arial";
  font-size: 18px;
  }
</style>


# Análisis de sentimientos con reviews de productos de Amazon España

## Indice:
<ul >
<li>  Objetivo.  </li>
<li>  Librerías necesarias. </li>  
<li>  Lectura y preparación de los datos </li>  
<li> WordMap  </li>  
<li> Modelo no supervisado: análisis de sentimiento con TextBlob  </li>  
<li> Modelos supervisados:</li>  <ul>
    <li> Bag-of-words (BOW) method.  </li>  
    <li> Regresión logística.  </li>  
    <li> KNN.  </li>  
    <li> Árbol de decisión.  </li>  </ul>
<li> Mejor modelo y validación.  </li>  
</ul>
## Objetivo: 

El objetivo de este post utilizar técnicas de procesamiento de lenguaje natural para  es hacer un análisis de sentimiento de productos de amazon.es.  

Para ello, usaremos un dataset de reviews de amazon.es proporcionado por Julio Soto, profesor del máster "MÁSTER EXPERTO BIG DATA & ANALYTICS" en Datahack. 

Dicho dataset contiene 700.000 registros y dos columnas: 
<ul style="font-size:20px">
<li> Número de estrellas dadas por un usuario a un determinado producto, siendo 5 estrellas la mejor valoración posible y 1 la peor. </li>
    <li> Comentario sobre dicho producto. </li>
</ul>
Para facilitar el análisis, supondremos que el comentario es positivo si la valoración tiene 4 o más estrellas, y negativo si tiene menos de 4. 


## Librerías necesarias:
```python

import datetime
import zipfile
import re
import pandas as pd
import numpy as np
import operator

import nltk
from nltk.corpus import stopwords 
nltk.download('stopwords')
from nltk import word_tokenize
from nltk.data import load
from nltk.stem import SnowballStemmer
from string import punctuation

from wordcloud import WordCloud
import matplotlib.pyplot as plt
import seaborn as sn

from textblob import TextBlob

from sklearn.feature_extraction.text import CountVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import confusion_matrix, accuracy_score, roc_auc_score
from sklearn.model_selection import train_test_split, GridSearchCV
from sklearn.neighbors import KNeighborsClassifier
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.tree import DecisionTreeClassifier
```

## Lectura y preparación de los datos

* Importamos datos  
* Recodificamos variables  

El primer paso es, lógicamente, **importar** los datos con los que vamos a trabajar y hacer un pequeño análisis de los mismos. 

```python
DF = pd.read_csv("amazon_es_reviews.csv", sep = ";").sample(n = 100000, random_state = 7) 
DF.head(10)
```
Como hemos comentado antes, disponemos dos dos variables, comentario (string) y estrellas (float).


    <class 'pandas.core.frame.DataFrame'>
    Int64Index: 100000 entries, 698609 to 637124
    Data columns (total 2 columns):
     #   Column      Non-Null Count   Dtype  
    ---  ------      --------------   -----  
     0   comentario  100000 non-null  object 
     1   estrellas   100000 non-null  float64
    dtypes: float64(1), object(1)
    memory usage: 2.3+ MB
    
Una vez tenemos nuestros datos disponibles, el siguiente paso será crear la variable *puntuación*, que valdrá 1 cuando el comentario es positivo (recibe 4 o más estrellas) o 0 si es negativo (recibe menos de 4 estrellas)

```python
# recodificamos valoraciones
DF['puntuacion']  = np.where((DF['estrellas'] >=4.0), 1, 0)
```

```python
DF.head()
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
      <th>comentario</th>
      <th>estrellas</th>
      <th>puntuacion</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>698609</th>
      <td>Muy mal las instrucciones vienen en japonés !!...</td>
      <td>1.0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>2395</th>
      <td>Cumple con la capacidad de 30 corbatas, el sis...</td>
      <td>4.0</td>
      <td>1</td>
    </tr>
    <tr>
      <th>80885</th>
      <td>La única pega es que llegó con un golpe en el ...</td>
      <td>4.0</td>
      <td>1</td>
    </tr>
    <tr>
      <th>157275</th>
      <td>Hola, no ha cumplido mis expectativas. He teni...</td>
      <td>1.0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>154777</th>
      <td>Estaba buscando una correa metálica para el So...</td>
      <td>5.0</td>
      <td>1</td>
    </tr>
  </tbody>
</table>
</div>


## WordMap

Es un gráfico sencillo que nos puede dar en un sólo vistazo una idea de los temas más relevantes de nuestros comentariosn. 

Antes de realizar el gráfico, tendremos que realizar una normalización y tokenización del texto. Los pasos que se han realizado son:

* *Tokenizar*, este paso convierte una cadena de texto en una lista de palabras (tokens). Usaremos un tokenizador modificado que no solo tokeniza (mediante el uso de nltk.word_tokenize), sino que también remueve signos de puntuación. Como estamos tratando con reviews en español, es importante incluir ¿ y ¡ en la lista de signos a eliminar.  
* Convertir todas las palabras en minúsculas.
* *Remover stopwords*. Se llama stopwords a las palabras que son muy frecuentes pero que no aportan gran valor sintáctico. Ejemplos de stopwords serían de, por, con …
* *Stemming*. Stemming es el proceso por el cual transformamos cada palabra en su raiz. Por ejemplo las palabras maravilloso, maravilla o maravillarse comparten la misma raíz y se consideran la misma palabra tras el stemming.


```python
spanish_stopwords = stopwords.words('spanish')
stemmer = SnowballStemmer('spanish')
non_words = list(punctuation) #elimina puntiación 
non_words.extend(['¿', '¡']) 
non_words.extend(map(str,range(10)))

def tokenize(text):
    text = ''.join([c for c in text if c not in non_words])
    tokens =  word_tokenize(text)
    # stem
    try:
        stems = stem_tokens(tokens, stemmer)
    except Exception as e:
        print(e)
        print(text)
        stems = ['']
    return stems

def stem_tokens(tokens, stemmer):
    stemmed = []
    for item in tokens:
        stemmed.append(stemmer.stem(item))
    return stemmed

DF['comentario_normalizado'] = DF['comentario'].apply(lambda x: tokenize(x))
```

```python
DF['comentario_normalizado2'] = DF['comentario_normalizado'].apply(lambda x: ', '.join(x))
```

```python
words = " ".join(str(DF['comentario_normalizado']))
text = "".join([word for word in words.split()])
#text = " ".join(review for review in DF['comentario'].str.lower()) 

wordcloud = WordCloud(
    stopwords=spanish_stopwords, 
    width = 3000,
    height = 2000,
    background_color = 'black').generate(str(text))
fig = plt.figure(
    figsize = (40, 30),
    facecolor = 'k',
    edgecolor = 'k')
plt.imshow(wordcloud, interpolation = 'bilinear')
plt.axis('off')
plt.tight_layout(pad=0)
plt.show()
```


![png](/images/output_15_0.png)


## Modelo no supervisado: análisis de sentimiento con TextBlob

TextBlob es una librería de procesamiento del texto para Python que permite realizar tareas de Procesamiento del Lenguaje Natural tales como análisis morfológico, extracción de entidades, análisis de opinión, traducción automática, etc.

En este caso, utilizaremos la librería para hacer una análisis de opinión de nuestra colección de reviews. El output obtenido será: 

* **Polarity** o polaridad es una medida que va de -1 a 1 y nos indica si el texto analizado es positivo (valores positivos) o negativo (valores negativos) 
* **Subjectivity** o subjetividad nos da una idea de si el texto analizado es una opinion personal, emoción o juicio o hace referencia a información objetiva. Este valor va de 0 a 1, cuanto más cercano a 0 esté nuestro valor tanto más objetivo será el texto analizado.

Al ser una técnica no supervisada, no es necesario dividir la muestra en test y train, por lo que tendremos que buscar un método alternativo para evaluar la eficacia del algoritmo. Para ello, necesitamos definir cuales de nuestros comentarios son positivos y cuales son negativos. Supondremos que cuando la variable puntuación vale 0 el comentarario es negativo y cuando vale 1 es positivo.

**Algortimo**

Definitmos dos funciones que nos calculen las variables polaridad y subjetividad de las que hablamos antes y se la aplicamos a nuestro texto ya normalizado:

```python
def sentiment_calc(text):
    try:
        return TextBlob(text).sentiment
    except:
        return None

DF['sentimental_analysis'] = DF['comentario_normalizado2'].apply(sentiment_calc)

```

```python
def polarity_calc(text):
    try:
        return TextBlob(text).sentiment.polarity
    except:
        return None

DF['polarity'] = DF['comentario_normalizado2'].apply(polarity_calc)

```

**Evaluación**

La polaridad es la variable que nos indica si un comentario es positivo o negativo. Para simplificar, si el valor de la polaridad diremos que el comentario analizado es positivo y en el resto de los casos supondremos que es un valor negativo.

```python
DF['polarity_buena_mala'] = [1 if x > 0 else 0 for x in DF['polarity']]
```

Una vez tenemos clasificadas nuestras reviews, comparamos los resultados con las puntuaciones reales de los usuarios. Realizamos una matriz de confusión y calculamos la accuracy.

*matriz de confusión:*

```python
cnf_matrix_textblob  = confusion_matrix(DF['polarity_buena_mala'], DF['puntuacion'])

df_cm = pd.DataFrame(cnf_matrix_textblob)
# plt.figure(figsize=(10,7))
sn.set(font_scale=1.4) # for label size
sn.heatmap(df_cm,annot=True, 
           fmt="d",
          cmap = "YlGnBu",
          cbar=False) 

plt.show()
```

![png](/images/output_22_0.png)


*Accuracy:*
```python
accuracy_score(DF['puntuacion'], DF['polarity_buena_mala'])
```

0.56934

Obtenemos una puntuación para el modelo de 0.57, los resultados de esta técnica no son buenos.

## Modelos supervisados para el análisis de sentimiento

Estamos ante un problema de clasificación finalmente. Realizaremos un pipeline en el que incluiremos, además de los hiperparámetros correspondientes a cada modelo, hiperarámetros para evaluar diferentes formas de formar nuetra tabla de features.

Por limitaciones del ordenador solamente evaluaremos si es mejor incluir unigrams o bigrams (n-gram o subsecuencias de 1 o 2 palabras) y el número máximas features a tener en cuenta.

Por esta misma limitación, nos limitaremos a testear modelos que "consuman menos recursos" de nuestro ordenador: Regresión Logística, KNN y Árbol de decisión.

### Bag-of-words (BOW) method

Para poder analizar los comentarios, tenemos que extraer y estructurar la información contenida en el texto. Para ello, usaremos la clase sklearn.feature_extraction.CountVectorizer.
CountVectorizer convierte la columna de texto en una matriz en la que cada palabra es una columna cuyo valor es el número de veces que dicha palabra aparece en cada review. De esta forma podemos trabajar con estos vectores en vez de con texto plano.

Modificaremos nuestro CountVectorizer para que aplique los siguientes pasos a cada comentario: 
* *Tokenizar*  
* *Convertir todas las palabras en minúsculas.*
* *Eliminar stopwords*  
* *Stemming*  


```python
vectorizer = CountVectorizer(
                analyzer = 'word',
                tokenizer = tokenize, # función creada para el wordmap
                lowercase = True,
                stop_words = spanish_stopwords)
```
Una vez tenemos nuestras variables listas, el siguiente paso será dividir nuestra muestra en train y test. Usaremos nuestra muestra train para entrenar el modelo y luego testearemos los resultados con la muestra de test:

```python
DF_train, DF_test = train_test_split(DF,test_size=0.20, random_state=42)
```
Además, vamos a crear un diccionario en el que iremos guardando los mejores resultados para cada uno de los algoritmos testeados:
```python
Mejor_modelo = {}
```

### Regresión logística

El primer modelo que vamos a probar es una regresión logistica. En vez de usar los parámetros que por defecto tiene el método, aplicaremos GridSearchCV para seleccionar los parámetros que nos den un mejor resultado. La accuracy para cada uno de los parámetros testeados se medirá mediante la curva ROC(*roc_auc*).

**Algoritmo**

```python
pipeline_reglog = Pipeline([    
    ('vect', vectorizer),    
    ('reglog', LogisticRegression())
])

parameters = {
    #'vect__max_df': (0.5, 1.9),
    #'vect__min_df': (10, 20,50),
    'vect__max_features': (500, 2000),
    'vect__ngram_range': ((1, 1), (1, 2))
}

gs_reglog = GridSearchCV(pipeline_reglog, 
                        param_grid = parameters, 
                        cv = 10, 
                        scoring = "roc_auc", 
                        verbose = 3,
                        n_jobs=-1) 

gs_reglog.fit(DF_train['comentario'], DF_train['puntuacion'])
```

**Resultados:**

```python
print("Los mejores parámetros son %s con un score de %0.3f"
      % (gs_reglog.best_params_,gs_reglog.best_score_))
```

Los mejores parámetros son {'vect__max_features': 2000, 'vect__ngram_range': (1, 2)} con un score de 0.878

Gardamos el resultado en nuestro diccionario

```python
Mejor_modelo['RegresionLogistica'] = gs_reglog.best_score_
```

### KNN

El siguiente algoritmo que vamos a probar es  k-nearest neighbors (KNN)

**Algoritmo:**

```python
pipeline_KNN = Pipeline([
    ('vect', vectorizer),  
    ('KNN', KNeighborsClassifier())
])

grid_hyper_KNN = {   
    #'vect__max_df': (0.5, 1.9),
    #'vect__min_df': (10, 20,50),
    'vect__max_features': (500, 2000),
    'vect__ngram_range': ((1, 1), (1, 2)),
    'KNN__n_neighbors':[1, 3, 5, 7, 11, 13, 15]}  

gs_KNN = GridSearchCV(pipeline_KNN, 
                        param_grid = grid_hyper_KNN, 
                        cv = 10, 
                        scoring = "roc_auc", 
                        verbose = 1,
                        n_jobs=-1)

gs_KNN.fit(DF_train['comentario'], DF_train['puntuacion'])
```

**Resultados:**

```python
print("Los mejores parámetros son %s con un score de %0.3f"
      % (gs_KNN.best_params_,gs_KNN.best_score_))
```

Los mejores parámetros son {'KNN__n_neighbors': 15, 'vect__max_features': 500, 'vect__ngram_range': (1, 2)} con un score de 0.742
    
Guardamos los resultados en nuestro diccionario:

```python
Mejor_modelo['KNN'] = gs_KNN.best_score_
```

### Árbol de decisión

```python
pipeline_ArbolDecison = Pipeline([
    ('vect', vectorizer),  
    ('ArbolDecison', DecisionTreeClassifier())
])

grid_ArbolDecison= {
    #'vect__max_df': (0.5, 1.9),
    #'vect__min_df': (10, 20,50),
    'vect__max_features': (500, 2000),
    'vect__ngram_range': ((1, 1), (1, 2)),
    'ArbolDecison__max_depth':np.arange(3,9)}

gs_ArbolDecison = GridSearchCV(pipeline_ArbolDecison, 
                        param_grid = grid_ArbolDecison, 
                        cv = 10, 
                        scoring = "roc_auc", 
                        verbose = 1,
                        n_jobs=-1
                              ) 

gs_ArbolDecison.fit(DF_train['comentario'], DF_train['puntuacion'])
```

**Resultado:**

```python
Mejor_modelo['ArbolDecison'] = gs_ArbolDecison.best_score_
```

## Mejor modelo y validación

Dados los datos, el modelo que nos ha dado mejor resultado has sido la regresión logistica con un AUC igual a 0.88:

```python

Modelos_orden = sorted(Mejor_modelo.items(), key=operator.itemgetter(1), reverse=True)
for modelo in enumerate(Modelos_orden):
    print(modelo[1][0], ' tiene un AUC igual a ', round(Mejor_modelo[modelo[1][0]], 4))
```

    RegresionLogistica  tiene un AUC igual a  0.8783
    KNN  tiene un AUC igual a  0.7419
    ArbolDecison  tiene un AUC igual a  0.7338
    

```python
mejor_modelo = gs_reglog.best_estimator_

```
Una vez tenemos seleccionado nuestro modelo, usamos la muestra de train para entrenarlo:

```python
mejor_modelo.fit(DF_train['comentario'], DF_train['puntuacion'])
```

Y hacemos predicciones sobre nuestra muestra de test:

```python
predicciones_test = mejor_modelo.predict(DF_test['comentario'])
DF_test['predicciones'] = predicciones_test

```
      
Para terminar, evaluamos los resultados:    
    
**Matriz de confusión:**

```python

cnf_matrix  = confusion_matrix(DF_test['predicciones'], DF_test['puntuacion'])#.ravel()



df_cm = pd.DataFrame(cnf_matrix)
# plt.figure(figsize=(10,7))
sn.set(font_scale=1.4) # for label size
sn.heatmap(df_cm,annot=True, 
           fmt="d",
          cmap = "YlGnBu",
          cbar=False) 

plt.show()
```

![png](/images/output_48_0.png)

```python
from sklearn.metrics  import accuracy_score
accuracy_score(DF_test['puntuacion'], DF_test['predicciones'])

```

0.80715

**AUC:**
```python
AUC = roc_auc_score(DF_test['puntuacion'], DF_test['predicciones'])
print(AUC)
```

0.8067914315098941


