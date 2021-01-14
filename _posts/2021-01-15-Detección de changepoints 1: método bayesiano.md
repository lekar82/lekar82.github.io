<style> div { font-family:"Arial"; font-size: 18px; } </style>

# WORK IN PROGRESS....

# Detección de changepoints 1: método bayesiano

## Indice:

* [Objetivo](#Objetivo)
* [Librerías necesarias](#Librerías-necesarias)
* [Datos](#Datos)

## Objetivo

En este post vamos a analizar como detectar Changepoint en nuestros datos mediante el método Bayesiano. 

Los métodos bayesianos para detectar cambios en la tendecia de nuestros datos pueden ser offline u online. La principal diferencia entre los dos métodos reside en la capacidad de poder procesar los datos según lleguen, es decir, en "tiempo real" (online) o si el proceso va a procesar todos los datos de una vez (offline). 

Uno u otro método se usará dependiendo del objetivo del análisis. Los métodos online suelen estar enfocados a la detección de roturas en la serie de datos  tan pronto como sea posible, intentando minimizar el número de falsos positivos. Por otro lado, los métodos offlines suelen tener como objetivo detectar todas los cambios de tendencia originados en una serie, para así definir los criterios de sensibilidad (detección de cambios de tendencia reales) y especefeceidad (evitando falsos positivos).

## Librerias

import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from datetime import datetime

import ruptures as rpt
from tabula.io import read_pdf
#import tabula
from PyPDF2 import PdfFileWriter, PdfFileReader
import io


import math

import cProfile
import bayesian_changepoint_detection.offline_changepoint_detection as offcd
import bayesian_changepoint_detection.online_changepoint_detection as oncd
from functools import partial

