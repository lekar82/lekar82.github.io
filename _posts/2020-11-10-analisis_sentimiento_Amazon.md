# Análisis de sentimientos con reviews de productos de Amazon España

## Indice:
* Objectivo.  
* Librerías necesarias  
* Lectura y preparación de los datos


## Introducción: 

El objetivo de este post utilizar técnicas de procesamiento de lenguaje natural para  es hacer un análisis de sentimiento de productos de amazon.es.  

Para ello, usaremos un dataset de reviews de amazon.es proporcionado por Julio Soto, profesor del máster "MÁSTER EXPERTO BIG DATA & ANALYTICS" en Datahack. 

Dicho dataset contiene 700.000 registros y dos columnas: 
* Número de estrellas dadas por un usuario a un determinado producto, siendo 5 estrellas la mejor valoración posible y 1 la peor.
* Comentario sobre dicho producto; exactamente igual que en el ejercico de scraping.

Para facilitar el análisis, supondremos que la valoración es positiva si la valoración tiene 4 o más estrellas, y negativa si tiene menos de 4. 


## Librerías necesarias:
```python
import datetime
import zipfile
import re
import pandas as pd
import numpy as np

import nltk
from nltk.corpus import stopwords 
nltk.download('stopwords')
from nltk import word_tokenize
from nltk.data import load
from nltk.stem import SnowballStemmer
from string import punctuation

from wordcloud import WordCloud
import matplotlib.pyplot as plt

from textblob import TextBlob

from sklearn.feature_extraction.text import CountVectorizer

from sklearn.model_selection import train_test_split

from sklearn.pipeline import Pipeline
from sklearn.svm import LinearSVC
from sklearn.model_selection import GridSearchCV

from sklearn.tree import DecisionTreeClassifier
from sklearn.neighbors import KNeighborsClassifier
from sklearn.svm import SVC
from sklearn.linear_model import LogisticRegression
from sklearn.preprocessing import StandardScaler

from sklearn.metrics import confusion_matrix
from sklearn.metrics  import accuracy_score

#from sklearn.utils import parallel_backend
#parallel_backend('multiprocessing')

```

## Lectura y preparación de los datos

* Importamos datos  
* Recodificamos variables  

```python
DF = pd.read_csv("amazon_es_reviews.csv", sep = ";").sample(n = 100000, random_state = 7) 
DF.head(10)
```
```python
DF.info()
```

* Recodificamos variables  
```python
DF['puntuacion']  = np.where((DF['estrellas'] >=4.0), 1, 0)
```

## WordMap

Es un gráfico sencillo que nos puede dar en un sólo vistazo una idea de los temas más relevantes de nuestros comentariosn. 

Antes de realizar el gráfico, tendremos que realizar una normalización y tokenización del texto. Los pasos que se han realizado son:

* *Tokenizar*, este paso convierte una cadena de texto en una lista de palabras (tokens). Usaremos un tokenizador modificado que no solo tokeniza (mediante el uso de nltk.word_tokenize), sino que también remueve signos de puntuación. Como estamos tratando con tweets en español, es importante incluir ¿ y ¡ en la lista de signos a eliminar.  
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
#DF['comentario_limpio'] = DF['comentario'].str.lower()
#DF['comentario_limpio'] = DF['comentario_limpio'].apply(lambda x: re.sub("[^a-zA-Z-áéíóúÁÉÍÓÚñÑ]", " ", x))
#DF.head()
```


```python
#DF['comentario_limpio2'] = DF['comentario_limpio'].apply(lambda x: [item for item in x if item not in stop])

#DF['comentario_limpio2'] = DF['comentario_limpio'].apply(lambda x: ' '.join([word for word in x.split() if word not in (stop)]))
#DF.head()

```


```python
#nltk.download('wordnet') #problema, no está en español

#w_tokenizer = nltk.tokenize.WhitespaceTokenizer()
#lemmatizer = nltk.stem.WordNetLemmatizer()

#def lemmatize_text(text):
    #return [lemmatizer.lemmatize(text) for w in w_tokenizer.tokenize(text)]

#DF['comentario_limpio3'] =  DF['comentario_limpio2'].apply(lemmatize_text)

#DF.head()    
    
```


```python
#from nltk.stem import LancasterStemmer
#from nltk import wordpunct_tokenize
#from nltk.corpus import stopwords
#import nltk
#from nltk.stem import PorterStemmer
#from nltk.stem import SnowballStemmer

#st = SnowballStemmer('spanish')

#st = PorterStemmer()

#DF['comentario_limpio4'] = DF['comentario_limpio2'].apply(lambda x: " ".join([st.stem(word) for word in x.split()]))
#DF.head()

#https://www.datacamp.com/community/tutorials/stemming-lemmatization-python
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


![png](output_15_0.png)


# Analisis de sentimiento con TextBlob

Al ser una técnica no supervisada, no es necesario dividir la muestra en test y train

El output obtenido es : Sentiment(polarity=, subjectivity=)

* **Polarity** o polaridad es una medida que va de -1 a 1 y nos indica si el texto analizado es positivo (valores positivos) o negativo (valores negativos) 
* **Subjectivity** o subjetividad nos da una idea de si el texto analizado es una opinion personal, emoción o juicio o hace referencia a información objetiva. Este valor va de 0 a 1, cuanto más cercano a 0 esté nuestro valor tanto más objetivo será el texto analizado.


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

Para analizar la capacidad de predicción de este algoritmo necesitamos saber cuales de nuestros comentarios son positivos y cuales son negativos. Supondremos que cuando la variable puntuación vale 0 el comentarario es negativo y cuando vale 1 es positivo.


```python
DF['polarity_buena_mala'] = [1 if x > 0 else 0 for x in DF['polarity']]
```

Tras calcular la matriz de confusión y la accuracy de esté approach vemos que los resultados no son buenos


```python
cnf_matrix_textblob  = confusion_matrix(DF['polarity_buena_mala'], DF['puntuacion'])

import seaborn as sn
import pandas as pd
import matplotlib.pyplot as plt

df_cm = pd.DataFrame(cnf_matrix_textblob)
# plt.figure(figsize=(10,7))
sn.set(font_scale=1.4) # for label size
sn.heatmap(df_cm,annot=True, 
           fmt="d",
          cmap = "YlGnBu",
          cbar=False) 

plt.show()
```


![png](output_22_0.png)



```python
accuracy_score(DF['puntuacion'], DF['polarity_buena_mala'])
```




    0.56934



# Modelos supervisados para el análisis de sentimiento

Estamos ante un problema de clasificación finalmente. Realizaremos un pipeline en el que incluiremos, además de los hiperparámetros correspondientes a cada modelo, hiperarámetros para evaluar diferentes formas de formar nuetra tabla de features.

Por limitaciones del ordenador solamente evaluaremos si es mejor incluir unigrams o bigrams (n-gram o subsecuencias de 1 o 2 palabras) y el número máximas features a tener en cuenta.

Por esta misma limitación, nos limitaremos a testear modelos que "consuman menos recursos" de nuestro ordenador: Regresión Logística, KNN y Árbol de decisión.



## Bag-of-words (BOW) method

Para poder analizar los comentarios, tenemos que extraer y estructurar la información contenida en el texto. Para ello, usaremos la clase sklearn.feature_extraction.CountVectorizer.
CountVectorizer convierte la columna de texto en una matriz en la que cada palabra es una columna cuyo valor es el número de veces que dicha palabra aparece en cada tweet.

De esta forma podemos trabajar con estos vectores en vez de con texto plano.
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


```python
DF_train, DF_test = train_test_split(DF,test_size=0.20, random_state=42)
```


```python
Mejor_modelo = {}
```

## Regresión logística:


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

    Fitting 10 folds for each of 4 candidates, totalling 40 fits
    

    [Parallel(n_jobs=-1)]: Using backend LokyBackend with 4 concurrent workers.
    [Parallel(n_jobs=-1)]: Done  24 tasks      | elapsed: 37.6min
    [Parallel(n_jobs=-1)]: Done  40 out of  40 | elapsed: 60.8min finished
    C:\Users\rdelval\Anaconda3\lib\site-packages\sklearn\feature_extraction\text.py:385: UserWarning: Your stop_words may be inconsistent with your preprocessing. Tokenizing the stop words generated tokens ['algun', 'com', 'contr', 'cuand', 'desd', 'dond', 'durant', 'eram', 'estab', 'estais', 'estam', 'estan', 'estand', 'estaran', 'estaras', 'esteis', 'estem', 'esten', 'estes', 'estuv', 'fuer', 'fues', 'fuim', 'fuist', 'hab', 'habr', 'habran', 'habras', 'hast', 'hem', 'hub', 'mas', 'mia', 'mias', 'mio', 'mios', 'much', 'nad', 'nosotr', 'nuestr', 'par', 'per', 'poc', 'porqu', 'qui', 'seais', 'seam', 'sent', 'ser', 'seran', 'seras', 'si', 'sient', 'sint', 'sobr', 'som', 'suy', 'tambien', 'tant', 'ten', 'tendr', 'tendran', 'tendras', 'teng', 'tien', 'tod', 'tuv', 'tuy', 'vosotr', 'vuestr'] not in stop_words.
      'stop_words.' % sorted(inconsistent))
    C:\Users\rdelval\Anaconda3\lib\site-packages\sklearn\linear_model\_logistic.py:940: ConvergenceWarning: lbfgs failed to converge (status=1):
    STOP: TOTAL NO. of ITERATIONS REACHED LIMIT.
    
    Increase the number of iterations (max_iter) or scale the data as shown in:
        https://scikit-learn.org/stable/modules/preprocessing.html
    Please also refer to the documentation for alternative solver options:
        https://scikit-learn.org/stable/modules/linear_model.html#logistic-regression
      extra_warning_msg=_LOGISTIC_SOLVER_CONVERGENCE_MSG)
    




    GridSearchCV(cv=10, error_score=nan,
                 estimator=Pipeline(memory=None,
                                    steps=[('vect',
                                            CountVectorizer(analyzer='word',
                                                            binary=False,
                                                            decode_error='strict',
                                                            dtype=<class 'numpy.int64'>,
                                                            encoding='utf-8',
                                                            input='content',
                                                            lowercase=True,
                                                            max_df=1.0,
                                                            max_features=None,
                                                            min_df=1,
                                                            ngram_range=(1, 1),
                                                            preprocessor=None,
                                                            stop_words=['de', 'la',
                                                                        'que', 'el',
                                                                        'en', 'y',
                                                                        'a', 'los',
                                                                        '...
                                                               l1_ratio=None,
                                                               max_iter=100,
                                                               multi_class='auto',
                                                               n_jobs=None,
                                                               penalty='l2',
                                                               random_state=None,
                                                               solver='lbfgs',
                                                               tol=0.0001,
                                                               verbose=0,
                                                               warm_start=False))],
                                    verbose=False),
                 iid='deprecated', n_jobs=-1,
                 param_grid={'vect__max_features': (500, 2000),
                             'vect__ngram_range': ((1, 1), (1, 2))},
                 pre_dispatch='2*n_jobs', refit=True, return_train_score=False,
                 scoring='roc_auc', verbose=3)




```python
print("Los mejores parámetros son %s con un score de %0.3f"
      % (gs_reglog.best_params_,gs_reglog.best_score_))
```

    Los mejores parámetros son {'vect__max_features': 2000, 'vect__ngram_range': (1, 2)} con un score de 0.878
    


```python
Mejor_modelo['RegresionLogistica'] = gs_reglog.best_score_
```

## KNN


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

    [Parallel(n_jobs=-1)]: Using backend LokyBackend with 4 concurrent workers.
    

    Fitting 10 folds for each of 28 candidates, totalling 280 fits
    

    [Parallel(n_jobs=-1)]: Done  42 tasks      | elapsed: 70.5min
    [Parallel(n_jobs=-1)]: Done 192 tasks      | elapsed: 659.6min
    [Parallel(n_jobs=-1)]: Done 280 out of 280 | elapsed: 806.6min finished
    C:\Users\rdelval\Anaconda3\lib\site-packages\sklearn\feature_extraction\text.py:385: UserWarning: Your stop_words may be inconsistent with your preprocessing. Tokenizing the stop words generated tokens ['algun', 'com', 'contr', 'cuand', 'desd', 'dond', 'durant', 'eram', 'estab', 'estais', 'estam', 'estan', 'estand', 'estaran', 'estaras', 'esteis', 'estem', 'esten', 'estes', 'estuv', 'fuer', 'fues', 'fuim', 'fuist', 'hab', 'habr', 'habran', 'habras', 'hast', 'hem', 'hub', 'mas', 'mia', 'mias', 'mio', 'mios', 'much', 'nad', 'nosotr', 'nuestr', 'par', 'per', 'poc', 'porqu', 'qui', 'seais', 'seam', 'sent', 'ser', 'seran', 'seras', 'si', 'sient', 'sint', 'sobr', 'som', 'suy', 'tambien', 'tant', 'ten', 'tendr', 'tendran', 'tendras', 'teng', 'tien', 'tod', 'tuv', 'tuy', 'vosotr', 'vuestr'] not in stop_words.
      'stop_words.' % sorted(inconsistent))
    




    GridSearchCV(cv=10, error_score=nan,
                 estimator=Pipeline(memory=None,
                                    steps=[('vect',
                                            CountVectorizer(analyzer='word',
                                                            binary=False,
                                                            decode_error='strict',
                                                            dtype=<class 'numpy.int64'>,
                                                            encoding='utf-8',
                                                            input='content',
                                                            lowercase=True,
                                                            max_df=1.0,
                                                            max_features=None,
                                                            min_df=1,
                                                            ngram_range=(1, 1),
                                                            preprocessor=None,
                                                            stop_words=['de', 'la',
                                                                        'que', 'el',
                                                                        'en', 'y',
                                                                        'a', 'los',
                                                                        '...
                                            KNeighborsClassifier(algorithm='auto',
                                                                 leaf_size=30,
                                                                 metric='minkowski',
                                                                 metric_params=None,
                                                                 n_jobs=None,
                                                                 n_neighbors=5, p=2,
                                                                 weights='uniform'))],
                                    verbose=False),
                 iid='deprecated', n_jobs=-1,
                 param_grid={'KNN__n_neighbors': [1, 3, 5, 7, 11, 13, 15],
                             'vect__max_features': (500, 2000),
                             'vect__ngram_range': ((1, 1), (1, 2))},
                 pre_dispatch='2*n_jobs', refit=True, return_train_score=False,
                 scoring='roc_auc', verbose=1)




```python
print("Los mejores parámetros son %s con un score de %0.3f"
      % (gs_KNN.best_params_,gs_KNN.best_score_))
```

    Los mejores parámetros son {'KNN__n_neighbors': 15, 'vect__max_features': 500, 'vect__ngram_range': (1, 2)} con un score de 0.742
    


```python
Mejor_modelo['KNN'] = gs_KNN.best_score_
```

## Árbol de decisión


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

    Fitting 10 folds for each of 24 candidates, totalling 240 fits
    

    [Parallel(n_jobs=-1)]: Using backend LokyBackend with 4 concurrent workers.
    [Parallel(n_jobs=-1)]: Done  42 tasks      | elapsed: 67.4min
    [Parallel(n_jobs=-1)]: Done 192 tasks      | elapsed: 296.4min
    [Parallel(n_jobs=-1)]: Done 240 out of 240 | elapsed: 370.2min finished
    C:\Users\rdelval\Anaconda3\lib\site-packages\sklearn\feature_extraction\text.py:385: UserWarning: Your stop_words may be inconsistent with your preprocessing. Tokenizing the stop words generated tokens ['algun', 'com', 'contr', 'cuand', 'desd', 'dond', 'durant', 'eram', 'estab', 'estais', 'estam', 'estan', 'estand', 'estaran', 'estaras', 'esteis', 'estem', 'esten', 'estes', 'estuv', 'fuer', 'fues', 'fuim', 'fuist', 'hab', 'habr', 'habran', 'habras', 'hast', 'hem', 'hub', 'mas', 'mia', 'mias', 'mio', 'mios', 'much', 'nad', 'nosotr', 'nuestr', 'par', 'per', 'poc', 'porqu', 'qui', 'seais', 'seam', 'sent', 'ser', 'seran', 'seras', 'si', 'sient', 'sint', 'sobr', 'som', 'suy', 'tambien', 'tant', 'ten', 'tendr', 'tendran', 'tendras', 'teng', 'tien', 'tod', 'tuv', 'tuy', 'vosotr', 'vuestr'] not in stop_words.
      'stop_words.' % sorted(inconsistent))
    




    GridSearchCV(cv=10, error_score=nan,
                 estimator=Pipeline(memory=None,
                                    steps=[('vect',
                                            CountVectorizer(analyzer='word',
                                                            binary=False,
                                                            decode_error='strict',
                                                            dtype=<class 'numpy.int64'>,
                                                            encoding='utf-8',
                                                            input='content',
                                                            lowercase=True,
                                                            max_df=1.0,
                                                            max_features=None,
                                                            min_df=1,
                                                            ngram_range=(1, 1),
                                                            preprocessor=None,
                                                            stop_words=['de', 'la',
                                                                        'que', 'el',
                                                                        'en', 'y',
                                                                        'a', 'los',
                                                                        '...
                                                                   min_samples_split=2,
                                                                   min_weight_fraction_leaf=0.0,
                                                                   presort='deprecated',
                                                                   random_state=None,
                                                                   splitter='best'))],
                                    verbose=False),
                 iid='deprecated', n_jobs=-1,
                 param_grid={'ArbolDecison__max_depth': array([3, 4, 5, 6, 7, 8]),
                             'vect__max_features': (500, 2000),
                             'vect__ngram_range': ((1, 1), (1, 2))},
                 pre_dispatch='2*n_jobs', refit=True, return_train_score=False,
                 scoring='roc_auc', verbose=1)




```python
Mejor_modelo['ArbolDecison'] = gs_ArbolDecison.best_score_
```

# Mejor modelo y validación


```python
import operator
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


```python
mejor_modelo
```




    Pipeline(memory=None,
             steps=[('vect',
                     CountVectorizer(analyzer='word', binary=False,
                                     decode_error='strict',
                                     dtype=<class 'numpy.int64'>, encoding='utf-8',
                                     input='content', lowercase=True, max_df=1.0,
                                     max_features=2000, min_df=1,
                                     ngram_range=(1, 2), preprocessor=None,
                                     stop_words=['de', 'la', 'que', 'el', 'en', 'y',
                                                 'a', 'los', 'del', 'se', 'las',
                                                 'por', 'un', 'para', 'con', 'no',...
                                     token_pattern='(?u)\\b\\w\\w+\\b',
                                     tokenizer=<function tokenize at 0x000001D02871FB70>,
                                     vocabulary=None)),
                    ('reglog',
                     LogisticRegression(C=1.0, class_weight=None, dual=False,
                                        fit_intercept=True, intercept_scaling=1,
                                        l1_ratio=None, max_iter=100,
                                        multi_class='auto', n_jobs=None,
                                        penalty='l2', random_state=None,
                                        solver='lbfgs', tol=0.0001, verbose=0,
                                        warm_start=False))],
             verbose=False)




```python
DF_train.head()
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
      <th>comentario_normalizado</th>
      <th>comentario_normalizado2</th>
      <th>sentimental_analysis</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>427463</th>
      <td>Decepcionado por este artículo, en la descripc...</td>
      <td>1.0</td>
      <td>0</td>
      <td>[decepcion, por, este, articul, en, la, descri...</td>
      <td>decepcion, por, este, articul, en, la, descrip...</td>
      <td>(0.0, 0.0)</td>
    </tr>
    <tr>
      <th>349346</th>
      <td>El tamaño es correcto, para llevar dos platos,...</td>
      <td>2.0</td>
      <td>0</td>
      <td>[el, tamañ, es, correct, par, llev, dos, plat,...</td>
      <td>el, tamañ, es, correct, par, llev, dos, plat, ...</td>
      <td>(-0.25, 0.25)</td>
    </tr>
    <tr>
      <th>342403</th>
      <td>He recibido esta bolsa y me gusta, de verdad. ...</td>
      <td>5.0</td>
      <td>1</td>
      <td>[he, recib, esta, bols, y, me, gust, de, verd,...</td>
      <td>he, recib, esta, bols, y, me, gust, de, verd, ...</td>
      <td>(0.5, 1.0)</td>
    </tr>
    <tr>
      <th>46925</th>
      <td>A mi no me dio problemas, si es cierto que dur...</td>
      <td>4.0</td>
      <td>1</td>
      <td>[a, mi, no, me, dio, problem, si, es, ciert, q...</td>
      <td>a, mi, no, me, dio, problem, si, es, ciert, qu...</td>
      <td>(0.0, 0.0)</td>
    </tr>
    <tr>
      <th>312325</th>
      <td>Compre 2 tarjetas para mi reflex y la velocida...</td>
      <td>2.0</td>
      <td>0</td>
      <td>[compr, tarjet, par, mi, reflex, y, la, veloc,...</td>
      <td>compr, tarjet, par, mi, reflex, y, la, veloc, ...</td>
      <td>(0.2, 0.2)</td>
    </tr>
  </tbody>
</table>
</div>




```python
mejor_modelo.fit(DF_train['comentario'], DF_train['puntuacion'])
```

    C:\Users\rdelval\Anaconda3\lib\site-packages\sklearn\linear_model\_logistic.py:940: ConvergenceWarning: lbfgs failed to converge (status=1):
    STOP: TOTAL NO. of ITERATIONS REACHED LIMIT.
    
    Increase the number of iterations (max_iter) or scale the data as shown in:
        https://scikit-learn.org/stable/modules/preprocessing.html
    Please also refer to the documentation for alternative solver options:
        https://scikit-learn.org/stable/modules/linear_model.html#logistic-regression
      extra_warning_msg=_LOGISTIC_SOLVER_CONVERGENCE_MSG)
    




    Pipeline(memory=None,
             steps=[('vect',
                     CountVectorizer(analyzer='word', binary=False,
                                     decode_error='strict',
                                     dtype=<class 'numpy.int64'>, encoding='utf-8',
                                     input='content', lowercase=True, max_df=1.0,
                                     max_features=2000, min_df=1,
                                     ngram_range=(1, 2), preprocessor=None,
                                     stop_words=['de', 'la', 'que', 'el', 'en', 'y',
                                                 'a', 'los', 'del', 'se', 'las',
                                                 'por', 'un', 'para', 'con', 'no',...
                                     token_pattern='(?u)\\b\\w\\w+\\b',
                                     tokenizer=<function tokenize at 0x000001D02871FB70>,
                                     vocabulary=None)),
                    ('reglog',
                     LogisticRegression(C=1.0, class_weight=None, dual=False,
                                        fit_intercept=True, intercept_scaling=1,
                                        l1_ratio=None, max_iter=100,
                                        multi_class='auto', n_jobs=None,
                                        penalty='l2', random_state=None,
                                        solver='lbfgs', tol=0.0001, verbose=0,
                                        warm_start=False))],
             verbose=False)




```python
predicciones_test = mejor_modelo.predict(DF_test['comentario'])
DF_test['predicciones'] = predicciones_test

```

    C:\Users\rdelval\Anaconda3\lib\site-packages\ipykernel_launcher.py:2: SettingWithCopyWarning: 
    A value is trying to be set on a copy of a slice from a DataFrame.
    Try using .loc[row_indexer,col_indexer] = value instead
    
    See the caveats in the documentation: https://pandas.pydata.org/pandas-docs/stable/user_guide/indexing.html#returning-a-view-versus-a-copy
      
    


```python
from sklearn.metrics import roc_auc_score

AUC = roc_auc_score(DF_test['puntuacion'], DF_test['predicciones'])
print(AUC)

```

    0.8067914315098941
    


```python


cnf_matrix  = confusion_matrix(DF_test['predicciones'], DF_test['puntuacion'])#.ravel()

import seaborn as sn
import pandas as pd
import matplotlib.pyplot as plt

df_cm = pd.DataFrame(cnf_matrix)
# plt.figure(figsize=(10,7))
sn.set(font_scale=1.4) # for label size
sn.heatmap(df_cm,annot=True, 
           fmt="d",
          cmap = "YlGnBu",
          cbar=False) 

plt.show()
```


![png](output_48_0.png)



```python
from sklearn.metrics  import accuracy_score
accuracy_score(DF_test['puntuacion'], DF_test['predicciones'])

```




    0.80715


