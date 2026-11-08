
Se buscaba la standarización, por eso se basó en primero en opentracing(standard for tracing) y luego open censos --> otro standard para metricas y tracex --> tenia build -in exporters -- no cubrñian todo y tenian bastantes gaps


los mantainers de estos do sproyectos se juntaron y crearon OpenTelemetry, juntaron lo mejor de las otras dos apps

Open Telemetry es un standard  --> defino como se extraen, formato y como se exporta la telemetria.  -> es vendor neutral. --> prometheus por ejemplo tiene lenguaje de query y built in dashboarding capability

opentelemetry ofrece standards e implemtnaciones de estos estandars pero no es una APP es un estandar. --> te da APIS, SDKS Y TOOLS PARA INSTRUMENTAR, COLECTAR Y  EXPORTAR metricas, logs y trazas

![[Pasted image 20261108134838.png]]


como antes había 300 usb y ahora solo hay el tipo c, se bisca algo de eso, un estandar para toda la telemetría

OTeL: OpenTelemtry --> OTCA --> es el único caso donde se llama OT

Caracteristicas OTel

	- vendor neutral open source
	- language agnostic y portable
	- trabaja con cualquier backend comercial ---> prometheus
	- no es un backend itsels(app) --> no tien storage ni visualization
	- se diseña para flexibilidad con la infrastuctura moderna


# Goals, Mission, Vision and Values

Goal(como): Make instrumentation easy, consistent y universal

![[Pasted image 20261108135909.png]]

Mission(que): hacer posible una observabilidad efectiva consiguiendo que haya telemetría portable y de alta calidad en todas partes

vision: como funciona en action
- telemetria debe ser easy --> rapadida
- universal --> protocolos convencionales para todos los lenguajes
- vendor neutral: no depender de un solo backend y facil switch
- muy desacoplada --> usas lo que necesites
- telemetría tiene que ser built-in: instrumentación nativa con el stack

Observability es lo que conducie la innovacion y reliability pero. necesita una telemtria consistente.

## Engineering values

![[Pasted image 20261108153249.png]]

# Specificacion y standarsds

![[Pasted image 20261108163338.png]]![[Pasted image 20261108175636.png]]