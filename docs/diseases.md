# 🦠 Enfermedades
Para la arquitectura de Leishmaniapp, una enfermedad será considerada como un módulo que define una serie de reglas que permiten llegar al diagnóstico de una enfermedad en particular, pues incluye una serie de elementos de interés (considerados dentro de Leishamiaapp como **elementos diagnósticos**). Los cuales pueden incluir: macrófagos, parásitos, gametocitos, leucocitos, etc.

Para realizar el diagnóstico, Leishmaniapp asigna un **identificador** único a cada enfermedad, este identificador sólo puede contener caractéres alfanuméricos y puntos, a través del identificador se emplea un modelo de detección asociado para cada enfermedad (Véase [modelos de detección](models.md)) el cual recibe una imagen y sus metadatos, mediante los cuales genera como resultado un mapa llave-valor donde la llave representa el nombre del elemento diagnóstico y el valor una lista con las coordenadas de las ocurrencias de ese elemento.

En este sentido, el diagnóstico de la enfermedad ocurre cuando el modelo de detección identifica sobre la imagen recibida las coordenadas correspondientes a los puntos donde se encuentran los elementos diagnósticos característicos de la enfermedad.

Ejemplo de una muestra de _leishmaniasis.giemsa_ con el elemento _parasite_ enmarcado
![Ejemplo Leishmaniasis](assets/diseases/leishmaniasis.png)

## Enfermedades de Prueba
Conjunto de enfermedades ficticias cuyo objetivo es realizar pruebas sobre la arquitectura de Leishmaniapp con imágenes diagnósticas de elementos cotidianos

| Enfermedad          | Identificador | Elementos Diagnósticos                  |
| ------------------- | ------------- | --------------------------------------- |
| Detector de colores | mock.spots    | red, yellow, green, cyan, blue, magenta |

## Listado de Enfermedades Soportadas
| Enfermedad                       | Identificador        | Elementos Diagnósticos                       |
| -------------------------------- | -------------------- | -------------------------------------------- |
| Leishmaniasis con tinción Giemsa | leishmaniasis.giemsa | macrophage, parasite                         |
| Malaria con tinción Romanowsky   | malaria.romanowsky   | leukocyte, trophozoite, schizont, gametocyte |