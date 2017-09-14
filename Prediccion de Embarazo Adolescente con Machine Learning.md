# Predicción de Embarazo Adolescente con Machine Learning #

En este caso de estudio, me gustaría contarte cómo hicimos para detectar jóvenes adolescentes en riesgo de quedar embarazada utilizando técnicas de Machine Learning.

Puedes usar este caso como una guía para crear tu propio modelo, o simplemente leerlo para conocer la lógica y secuencia de pasos que seguimos para llegar a nuestro resultado. En la misma línea y si es la primera vez trabajas con Machine Learning, te recomiendo que leas el [caso de predicción de deserción escolar](https://github.com/marcelofelman/case-studies/blob/master/Desercion%20escolar%20con%20Machine%20Learning.md) documentado por mi colega Marcelo Felman con quien tuvimos el agrado de trabajar junto al Ministerio de Primera Infancia en Salta.

## Resumen ##

En colaboración con el Ministerio de Primera Infancia del [Gobierno Provincial de Salta](http://www.salta.gov.ar), definimos como objetivo utilizar inteligencia artificial para identificar aquellas adolescentes con mayor riesgo de quedar embarazada.

Afortunadamente, tuvimos acceso a un amplio espectro de datos. Utilizamos un *dataset* de más de 200.000 residentes de la ciudad de Salta con más de 12.000 mujeres de entre 10 y 19 años , Argentina. **Estos datos no contienen información personal identificable sobre las personas, tal como reconoce [habeas data](https://es.wikipedia.org/wiki/Habeas_data)**.

A través de las herramientas [Azure Machine Learning](https://azure.microsoft.com/en-us/services/machine-learning/
) y [SQL Server 2016](https://www.microsoft.com/en-us/sql-server/sql-server-2016
), logramos crear diferentes modelos predictivos que permiten detectar hasta el 90% de los casos en riesgo de embarazo adolescente.

## Contexto ##

[Salta](https://es.wikipedia.org/wiki/Salta_(ciudad)) es una de las ciudades más pobladas de Argentina y capital de la provincia con igual nombre.

Con una población que supera los 500.000 habitantes, su Ministerio de Primera Infancia tiene por misión erradicar la pobreza en la provincia. Con este objetivo en mente, el desarrollo de las capacidades de los individuos es algo fundamental. El Ministerio de Primera Infancia propone actuar de manera proactiva identificando con Inteligencia Artificial aquellas jóvenes en riesgo de quedar embarazada durante su adolescencia.

La duración del proyecto fueron dos semanas, en las cuales iniciamos desde la exploración del dominio hasta la publicación de un servicio web predictivo.

## Software y herramientas ##

- [Azure Machine Learning](https://azure.microsoft.com/en-us/services/machine-learning/)
- [SQL Server 2016](https://www.microsoft.com/en-us/sql-server/sql-server-2016)

## Fases del proyecto ##

- [Exploración del dominio](#exploración-del-dominio)
- [Preparación de los datos](#preparación-de-los-datos)
- [Creación de modelos](#creación-de-modelos)
- [Integración](#integración)

## Exploración del dominio ##

Antes de empezar, debemos comprender el dominio en el cual estamos trabajando. Si bien lo ideal es contar con un experto en el dominio, no siempre lo tendremos a disposición. Para esto, es importante tener en cuenta diferentes técnicas así como también ejercitar nuestro sentido común.

Para comenzar a visualizar los datos, hemos utilizado un motor de bases de datos: SQL Server. De esta forma puedo escribir consultas SQL fácilmente y además exportar como CSV o inclusive conectar con Azure Machine Learning.

Para este proyecto, contamos con una tabla principal la cual contiene información de las personas: edad, etnia, país de origen, discapacidad, cantidad de personas con las que vive, si tiene agua caliente en el baño, barrio/zona en donde residen, si tuvo embarazo adolescente y si el jefe de hogar abandonó los estudios. A su vez, contiene información referencial (*foreign keys*) hacia sus padres o jefes de hogar.

Una práctica que encuentramos conveniente es unir todos los datos en una única tabla o proyección a través de la cláusula *JOIN*. De esta forma, podrás portarlo de manera más simple a Azure Machine Learning.

>**Tip:** Unir todos los datos en una única tabla o proyección en lugar de lidiar con muchos conjuntos de datos.

### Ejemplo #1 ###

Supongamos que sólo quiero conocer la relación entre las adolescentes embarazadas y saber si sus madres abandonaron los estudios, haríamos algo así:

```sql
SELECT
	chica.Embarazo AS EmbarazadaAdolescente,
	madre.AbandonoEstudios AS MadreAbandono,
	COUNT(*) AS Interseccion
FROM 
	PersonasSaltaCapital chica
INNER JOIN
	PersonasSaltaCapital madre
ON
	chica.CodMama = madre.CodPersona
WHERE
		chica.Embarazo = 'SI'
	AND chica.Edad <= 19
	AND chica.Sexo='Femenino'
GROUP BY
	chica.Embarazo,
	madre.AbandonoEstudios
ORDER BY
```

Esto es importante saberlo, ya que en la fase inicial lo haremos muchas veces.

### Pruebas simples ###

Para ir familiarizándote con el conjunto de datos, es bueno que ejecutes algunas consultas. Estas corrí yo como ejemplo:

- Cantidad total de personas
- Cantidad de jóvenes embarazadas versus las que no (menores de 20 años)
- Cantidad de jóvenes embarazadas que abandonaron los estuidos versus las que no
- Tasa de embarazo por zona en donde viven
- Cantidad de jóvenes embarazadas agrupadas por país de origen
- Cantidad de jóvenes embarazadas por etnia

>**Tip:** Invierte todo el tiempo que creas necesario para entender las relaciones entre los datos. 2 o 3 días enfocado en esto puede ser un tiempo razonable (aunque parezca mucho), dependiendo del dominio.

### Ejemplo #2 ###

Este es un ejemplo para calcular la cantidad total de jóvenes embarazadas:

```sql
SELECT COUNT(*)
FROM Personas
WHERE Embarazo = 'SI'
	AND Edad <= 19
	AND Sexo='Femenino'
```

Este es un ejemplo para calcular los embarazos por zona:

```sql
SELECT TOP 10
	Barrio,
	COUNT(*) AS EmbarazadasAdolescentes,
	(SELECT COUNT(*) FROM PersonasSaltaCapital psc2 WHERE psc2.Barrio = psc.Barrio AND psc2.Sexo='Femenino' AND psc2.Edad <= 19) AS TotalBarrio,
	CAST(COUNT(*) AS FLOAT) / CAST((SELECT COUNT(*) FROM PersonasSaltaCapital psc2 WHERE psc2.Barrio = psc.Barrio AND psc2.Sexo='Femenino' AND psc2.Edad <= 19) AS FLOAT) * 100 AS Porcentaje
FROM PersonasSaltaCapital psc
WHERE
	psc.Embarazo = 'SI'
	AND psc.Edad <= 19
	AND psc.Sexo='Femenino'
GROUP BY
	Barrio
ORDER BY
	Porcentaje DESC
```

Aquí, obtuvimos resultados similares a los siguientes (muestro el TOP 10 para simplificar):

| Barrio	    | Embarazadas Adolescentes	| TotalBarrio	| Porcentaje |
| ------------- |:-------------:| :-----:| :------:|
| Virgen de Urkupina |	2 |	28 |	7,14285714285714 |
| Finca La Paz |	1 |	17 |	5,88235294117647 |
| 16 de Septiembre |	1 |	21 |	4,76190476190476 |
| Isla Soledad |	2 |	43 |	4,65116279069767 |
| El Circulo VII |	1 |	23 |	4,34782608695652 | 
| Las Costas |	3 |	88 |	3,40909090909091 |
| Martin M. de Güemes |	2 |	67 |	2,98507462686567 |
| Manantial del Sur |	3 |	118 |	2,54237288135593 |
| Santa Rita S | 1 | 40 | 2,5 |
| San Isidro | 2 | 86 | 2,32558139534884 |

Como puedes ver, distintas zonas tienen distintas tasas de embarazo. Esto pueda llevarnos a pensar que la zona donde una adolescente resida, tenga que ver con su probabilidad de quedar embarazada. Por ahora, es sólo una hipótesis que luego probaremos.

>**Tip:** Seguro encuentres mejores maneras de escribir estas consultas. No te preocupes, la performance aquí no es importante. 

### Iteración ###

A estas alturas, empezarás a darte cuenta de ciertas variables que pueden llegar a ser o no relevantes en nuestro modelo. Proyécatalas todas y luego en Azure Machine Learning validaremos cuáles son las que sirven.

Para nuestro modelo, utilizaremos inicialmente las siguientes variables:

- Edad
- Barrio donde reside
- Etnia
- Pais de Origen
- Tipo de discapacidad, si es que tiene
- Tiene Agua Caliente en el Baño
- Cantidad de personas con quien vive
- Jefe de hogar abandonó estudios

Luego iteraremos para entender si estas variables nos sirven o no.

## Preparación de los datos ##

Ahora que sabemos qué datos queremos utilizar, debemos prepararlos para ir hacia Azure Machine Learning. Estando en SQL Server, una manera simple de hacerlo es guardando los resultados de nuestra consulta como CSV. 

Esto nos generará un archivo de tipo .rpt, al cual simplemente cambiar su extensión a .csv y abrirlo como tal.

>**Tip:** Los archivos CSV pueden abrirse con Excel. Si te hace sentir más cómodo, chequealo antes de seguir con el próximo paso.

Ahora, procedemos a ingresar al [Azure Machine Learning Studio](https://studio.azureml.net). Si no tienes una cuenta, puedes crearla de forma gratuita allí mismo.

Al ingresar, hacemos click en *New* y luego en *Dataset*

![Subir el conjunto de datos](https://github.com/facundod/case-studies/blob/master/images/1-upload-dataset.PNG?raw=true)

Subimos nuestro conjunto de datos, y al cabo de unos segundos estará en Azure.

Ahora puedes crear un nuevo experimento haciendo click en *+NEW > Blank Experiment*

>**Tip:** Puedes ponerle el nombre que quieras haciendo click en el nombre por defecto.

Ya subido el dataset, el mismo aparecerá en la solapa *My datasets* y lo podrás utilizar inmediatamente. Para esto, simplemente lo arrastras a la ventana principal.

![Nuevo experimento](https://github.com/facundod/case-studies/blob/master/images/2-new-experiment.PNG?raw=true)

Si quieres visualizarlo, puedes hacer click en la salida del módulo y otro click en *visualize*.

![Visualizar conjunto de datos](https://github.com/facundod/case-studies/blob/master/images/3-visualize-dataset.PNG?raw=true)

Deberías ver una pantalla similar a esta, pero con tus propios datos. Si haces click en cada una de las columnas, podrás ver la distribución junto con un histograma a la derecha.

![Visualizar los datos](https://github.com/facundod/case-studies/blob/master/images/4-visualize-chart.PNG?raw=true)

## Creación de modelos ##

Ahora empezaremos a crear nuestros modelos predictivos. Como mencionamos anteriormente, lo que estamos buscando es predecir *si una joven tiene o no un embarazo durante la adolescencia*. En otras palabras, es una clasificación binaria (de dos clases). Para más información sobre clasificación binaria y otros tipos de problemas, puedes ver [aquí](https://docs.microsoft.com/en-us/azure/machine-learning/machine-learning-algorithm-choice).

Para resolver este problema, existen diferentes algoritmos. Recuerda que los diferentes algoritmos son simplemente distintas formas de abordar a un resultado. Algunos llegan a mejores resultados, pero esto no está garantizado. Si quieres ver el listado y ventajas de cada uno de los algoritmos, lo puedes ver [aquí](https://docs.microsoft.com/en-us/azure/machine-learning/machine-learning-algorithm-choice). En este caso comenzamos utilizando un *Two-class Boosted Decision Tree*.

Para identificar la columna a predecir utilizaremos *Edit Metadata*. Debemos conectar la salida del dataset con la entrada de *Edit Metadata*. Usa el selector de columna a la derecha para elegir el campo.

![Seleccionar columna](https://github.com/facundod/case-studies/blob/master/images/5-select-edit-metadata-column.PNG?raw=true)

El dataset con el cual estamos trabajando tiene un 7% de jóvenes mujeres que tienen o tuvieron un embarazo adolescente. Si lo pensamos en frío, con tener un modelo que diga no ya estariamos cubriendo el 93% de los casos, pero lo que corresponde es balancear el conjunto de datos como se explica a continuación.

### Balancear el conjunto de datos ###

Los conjuntos de datos desbalanceados son un problema típico. Esta situación puede generar un balanceo que favorezca el escenario mayoritario, es decir el 'NO curso un embarazo adolescente'. Para ello, podemos aplicar dos técnicas distintas:

- *Undersampling*: tomar menos casos del escenario mayoritario a fin de reducir las ocurrencias.
- *Oversampling*: aumentar o simular más ocurrencias del caso minoritario.

En este caso, optamos por usar *Oversampling*. De esta manera, logramos un conjunto de datos más balanceado. Una manera simple y casi automática de lograr esto, es utilizando el módulo *SMOTE*, que significa *Synthetic Minority Oversampling* o bien sobre-muestreo sintético de minorías. No olvides aquí también editar metadata. Más información sobre SMOTE, [aquí](https://msdn.microsoft.com/en-us/library/azure/mt429826.aspx).

![SMOTE](https://github.com/facundod/case-studies/blob/master/images/6-smote.PNG?raw=true)

>**Tip:** En la solapa Propiedades de *SMOTE*, puedes ajustar el porcentaje de aumento de las ocurrencias minoritarias. Puedes jugar con este número hasta alcanzar un resultado que te sirva.


### Realizar las predicciones y Evaluar el comportamiento del modelo ###

Utilizaremos el módulo *Cross-Validate Model* el cual toma como entrada la salida de *Two-Class Boosted Decision* y  de *SMOTE*. Acto seguido, no olvidar seleccionar la columna que corresponda, en nuestro caso, la que indica si cursa o cursó un embarazo adolescente.

Por último evaluaremos el modelo utilizando el módulo *Evaluate model*, el cual tendrá conectada en su entrada la salida proveniente de *Cross-Validate Model*

![Evaluar modelo](https://github.com/facundod/case-studies/blob/master/images/7-evaluate-model.PNG?raw=true)

Si damos *Run* visualizamos la salida de *Evaluate model*, podremos ver la siguiente gráfica:

![Comportamiento](https://github.com/facundod/case-studies/blob/master/images/8-auc.PNG?raw=true)

Esta gráfica demuestra los casos que fueron correctamente identificados, respecto los que no. Esencialmente, el área debajo de la curva (*Area under the curve* o también *AUC*) debe ser lo mayor posible: no queremos dejar casos afuera.

A simple vista, nuestro modelo parece comportarse de una manera muy acertada: el **98,2%** de las veces está realizando una predicción correcta.

No obstante, si nos movemos hacia abajo veremos más métricas que definen el comportamiento de nuestro modelo predictivo.

![Más métricas](https://github.com/facundod/case-studies/blob/master/images/9-more-metrics.PNG?raw=true)

Como podemos apreciar, lo que estamos haciendo es identificar cuatro casos distintos:

- Jóvenes mujeres que dijimos que tienen o tuvieron embarazo adolescente y efectivamente tuvieron un embarazo adolescente (verdadero positivo): 1516.
- Jóvenes mujeres que dijimos que NO tienen o tuvieron embarazo adolescente pero tuvieron un embarazo adolescente (falso negativo): 169.
- Jóvenes mujeres que dijimos que tienen o tuvieron embarazo adolescente pero NO tuvieron un embarazo adolescente (falso positivo): 72.
- Jóvenes mujeres que dijimos que NO tienen o tuvieron embarazo adolescente y efectivamente fue así (verdadero negativo): 11702.

Debemos ser cautelosos y evaluar estos puntos. Una buena pregunta para hacernos es cuál es el costo de cada escenario. En este caso, es mucho peor NO ayudar a una joven mujer en riesgo de quedar embarazada durante su adolescencia que ayudar por demás a una adolescente que NO quedará embarazada durante su adolescencia. En otras palabras, el costo de los falsos negativos supera el de los falsos positivos.

 Como parte del proceso hemos experimentado modificando los valores de *SMOTE*, pero la mejor *sencibilidad* (o recall) que obtuvimos es de 0,9 que nos indica la proporción de eventos positivos identificados correctamente.

## Integración ##

### Creación del Web Service ###

Hemos llegado a la instancia donde estamos conformes sobre nuestro modelo, y queremos que sea consumido. La forma más simple será a través de un *Web Service REST*.

Para crear el servicio, debemos correr nuestro experimento (si es que no lo hicimos) y hacer click en el botón de *SET UP WEB SERVICE*.

![Preparar el Web Service](https://github.com/facundod/case-studies/blob/master/images/10-predictive-service.PNG?raw=true)

Esto nos generará una animación y creará una pestaña con el servicio web.

>**Tip:** No te preocupes por hacer modificaciones en esta instancia.

Deberás darle *Run* al servicio nuevamente, que es como si fuera "compilar" nuestra nueva API.

Finalmente, aparecerá el botón *DEPLOY WEB SERVICE* sobre el cual haremos click. Al cabo de unos segundos, nuestro servicio web estará listo para consumir.

![Desplegar el Web Service](https://github.com/facundod/case-studies/blob/master/images/11-deploy-service.PNG?raw=true)


### Probar el Web Service ###

Una vez desplegado, verás la siguiente pantalla.

![API del servicio](https://github.com/facundod/case-studies/blob/master/images/12-published-service.PNG?raw=true)

Notarás que tienes dos formas de utilizarlo:

- *Request/Response*: puedes generar la predicción para un único caso.
- *Batch execution*: puedes realizar muchas predicciones con un mismo pedido a la API.

>**Tip:** Te recomiendo arrancar por *request/response* si es tu primera vez.

Estos servicios pueden probarse directamente desde el portal. Para ello, puedes hacer click en el botón *TEST*. Te permitirá ingresar algunos campos, para finalmente darte una respuesta en formato *JSON*.

![Probar el web service](https://github.com/facundod/case-studies/blob/master/images/13-test-service.PNG?raw=true)

>**Tip:** También puedes probarlo integrándote con Excel. Es bastante simple y puede ahorrarte tiempo para ingresar atos.

De esta manera, tu servicio web predictivo ya es visible para el mundo exterior. ¡Felicitaciones!

## Conclusiones ##

Machine Learning es todo un mundo diferente, y puede resultar complejo para quienes venimos del ámbito de desarrollo de software.

A través de un proceso iterativo de prueba y error, podemos ir acercándonos a una respuesta correcta y finalmente determinar si nuestro modelo es bueno o no.

En nuestro caso, logramos un modelo predictivo que identifica correctamente a aproximadamente el 90% de las jóvenes mujeres están en riesgo quedar embarazadas durante su adolescencia. Esta herramienta permite al Gobierno tomar decisiones en tiempo real, y ayudar a aquellos que más lo necesiten.

## Agradecimientos ##

El mayor agradecimiento al Ministerio de Primera Infancia del Gobierno Provincial de Salta, quienes sin lugar a dudas quieren cambiar este mundo para el bien de todos y a Microsoft por posibilitarnos el hecho de ser parte de dicho cambio.