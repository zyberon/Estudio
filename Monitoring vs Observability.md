

# Monitoring
el monitoreo es la parte de la observabilidad que vigila de forma continua unas señales que has elegido de antemano, como métricas, umbrales o comprobaciones de salud, y te avisa cuando algo se sale de lo normal

1. collect data
2. abalizar dara
3. usar esa dara para saber estado y proceso hasta los objetivos


4. se predefinen metricas --> se sacan known issues (que se puede complemntar con logs) --> se recciona

La monitorizacón basta cuendo el error es conocido y tiene solución.



Ambas se complementan entre SÍ, el monitoreo te dice el qué a traves de las metricas(known unknoen) y la observabilidad te dice el por qué (unknown unknown)

# Observabilidad
Capacidad de entender un sistema a traves de la información que genera el propio sistema

se usa la telemetría para entender el sistema, alta demanda debido a sistema actuales muy fragmentados y de alta complejidad.
	- Solo con monitoring en sistemas complejos no bastaría
	- Se usa oara resiolver problenas y ver servicos


telemetría = datos que se extraen del entorno = métricas trazas y logs

señales:
	- metricas: propiedades del sistema medibles que permiten conocer la salud; Son numéricas y se agregan --> RED: Rate Errors, Duration
		- Esta compuesta por:
			- nombre
			- valor
			- timestamp
			- dimension
	- logs: registro. con mamrza de tiempo de un evento que emite una app --> encapsulan info del evento
	- trazas: recorrido de una request, sirven para conectar los puntos
	- luggage
	- profiling



# Aportacion PROPIA

La teoría los separa para explicar qué aporta cada uno, pero en la práctica el monitoreo es una pieza de la observabilidad. 

Lo único que sí se complementa son las dos partes de dentro: el monitoreo, que te avisa de qué ha pasado, y la investigación con trazas y datos detallados, que te explica por qué. **Las dos forman parte de la observabilidad.**

La observabilidad en verdad es la caracteristica del sustema de cuan observable es, después está la instrumentación que es la obtencion de información(por ejemplo en prometheus es el exporter y client libraries) y el monitoreo trabaja con la información obtenida para generar graficos y alertas.