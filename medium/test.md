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
