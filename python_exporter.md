# Documentacion del Programa Python Exporter 

## Descripcion General

Este programa tiene como objetivo crear un **exporter personalizado** que expone métricas sobre ventiladores y fuentes de poder para ser monitoreados con la ayuda de **Prometheus** y **Grafana**.

## Estructura del Script 

El script solicita al usuario el nombre del archivo y luego expone métricas usando `prometheus_client`. Las métricas están relacionadas con:

- Velocidad y estado del ventilador

- Potencia y estado de la fuente de poder


### Codigo Fuente 
```python
#!/usr/bin/python3

from prometheus_client import start_http_server, Gauge
import time
import re

#Para la obtencion de las metricas se utiliza una libreria basada en solicitudes http, Prometheus puede consultarla de manera periodica cada cierto tiempo 

programa = input("Ingrese el nombre del script generado para Grafana: ")
print("El script que ingresaste fue: ", programa)

#El usuario ingresa un script con el formato adecuado para ver el estado de las metricas y el programa te indica que script le indicaste.

# Definir métricas en Prometheus
fan_speed = Gauge("fan_speed", "Velocidad del ventilador en porcentaje", ["fan_id"])
fan_status = Gauge("fan_status", "Estado del ventilador", ["fan_id"])

#los laels en Prometheus estan representados por el nombre y estan entre comillas dobles por lo cual son pares clave-valor

# Gauge es una métrica de Prometheus que permite aumentar o disminuir valores. En este caso, fan_speed representa el porcentaje de velocidad de un ventilador. 

# El estado del ventilador sera representado por numeros booleanos; cero o uno.

power_supply = Gauge("power_supply", "Potencia de la fuente de poder en Watts", ["PSU_id"])
supply_status = Gauge("supply_status", "Estado de la fuente de poder", ["PSU_id"])

# power_supply representa representa la potencia de la fuente de poder 

# El estado de la fuente de poder sera representado por numeros booleanos; cero o uno.

def parse_info_file():
    try:
        with open(programa, "r") as f:
            for line in f:
                parts = line.split("|")
                if len(parts) == 3:    
                    name, value, status = [p.strip() for p in parts]

# Se define la funcion que leera el archivo ingresado.Recorre linea por linea. Se Verifica que la línea tenga extrictamente 3 elementos (name, value, status).Elimina espacios en blanco y se asignan a variables.

                    # Si es un ventilador

                    if "Fan" in name:
                        fan_id = name.split()[-1]
                        
                        # Extrae el ID del ventilador tomando la última palabra del nombre
                        # P. ej. si name = "Fan ID 1", entonces fan_id = "1"


                        if "ns" in status.lower() or "not readable" in value.lower():
                            fan_speed.labels(fan_id=fan_id).set(-1)
                            fan_status.labels(fan_id=fan_id).set(0)

                #Si al leer los datos se encuentra "ns" se le asignara el estado 0, ya que tiene la variable fan_status con valor booleano.
                            
                #Si se encuentra "not readable" este se le asignara un valor de -1 para que el valor no se malinterprete como cero de velocidad dando a entender que si esta en funcionamiento sin embargo no lo esta.


                        elif "ok" in status.lower():
                            try:
                                speed = float(value.split()[0])
                                fan_speed.labels(fan_id=fan_id).set(speed)
                                fan_status.labels(fan_id=fan_id).set(1)
                
                #Si despues de verificar que el estado es "ok" en fan_speed se le asignara el valor que tiene y en fan_status tendra el valor de 1 como disponible.

                            except ValueError:
                                print(f"Error al convertir la velocidad del ventilador: {value}")
                                fan_speed.labels(fan_id=fan_id).set(-1)
                                fan_status.labels(fan_id=fan_id).set(0)
        
                 #En la base de datos proporcionada, hay valores de los cuales no son legibles por lo que se creo esta condicional de la cual aplica el fan_speed como no legible con el valor de -1 e imprime el mensaje correspondiente


                    # Si es una fuente de poder

                    # Se verifica si el nombre contiene "Power Supply" o "PSU"
                    elif "Power Supply" in name or "PSU" in name:
                        match = re.search(r'(\d+)', name) 
                    # Se busca el primer número en la cadena 'name' usando una expresión regular

                        PSU_id = match.group(1) if match else "unknown"
                    # Si se encuentra un número, se asigna como el ID de la PSU, de lo contrario se asigna "unknown"

                        if "ns" in status.lower() or "not readable" in value.lower():
                            supply_status.labels(PSU_id=PSU_id).set(0)
                            power_supply.labels(PSU_id=PSU_id).set(-1)
                    
                # Si al leer los datos se encuentra la palabra "ns" se le asignara el estado 0 a supply_status

                # Si se encuentra "not readable" este se le asignara un valor de -1 para que el valor no se malinterprete como cero watts dando se da a entender que si esta en funcionamiento sin embargo no lo esta.


                        elif "ok" in status.lower():
                            try:
                                watts = float(value.split()[0]) if value else 0  
                                power_supply.labels(PSU_id=PSU_id).set(watts)
                                supply_status.labels(PSU_id=PSU_id).set(1)
                
                #Si despues de verificar que el estado es "ok" en power_supply se le asignara el valor que tiene y en supply_status tendra el valor de 1 como disponible.

                            except ValueError:
                                print(f"Error al convertir el valor de la Fuente de poder: {value}")
                                power_supply.labels(PSU_id=PSU_id).set(-1)
                                supply_status.labels(PSU_id=PSU_id).set(0)
                        
                # Hay valores de los cuales no son legibles por lo que se creo esta condicional de la cual aplica power_supply como no legible con el valor de -1 e imprime el mensaje correspondiente 

                        elif "PSU" in name and "Status" in name:
                
                # Si el nombre del dato contiene "PSU" y "Status" pero el estado no es reconocido,se asume un estado disponible.

                            supply_status.labels(PSU_id=PSU_id).set(0) 
                # Se establece el estado del PSU en 0, indicando que el estado no esta disponible.
 
                            power_supply.labels(PSU_id=PSU_id).set(-1)
                
                # Se establece el valor de potencia suministrada en -1, indicando que no se encuentra en funcionamiento.

    
#En caso de que no se encuentre el archivo se imprimira un mensaje 
    except FileNotFoundError:
        print(f"El archivo {programa} no fue encontrado.")

# la exposición de métricas se puede consultar en http:/localhost:8002/metrics
#Se realiza una actualizacion cada 5 min. si es que hay nuevos datos estos se actualizaran

def main():
    start_http_server(8002) 
    while True:
        parse_info_file()
        time.sleep(300)  

if __name__ == "__main__":
    main()

```

# Configuracion del programa en Prometheus 

Accedemos al archivo de configuracion de Prometheus denominado prometheus.yml.

### Codigo Fuente 
```YAML

# NOTA: Este archivo de configuracion es YAMl que es un lenguaje de serializacion de datos versatil y legible. Cuidar las sangrias  

- job_name: 'python_exporter'
# Tal y como nombramos nuestro programa, este tendra el mismo nombre en este archivo de configuracion.
    static_configs:
      - targets: ['localhost:8002'] #Se tiene que ocupar otro puerto diferente al de prometheus o al de otro servicio 
    scrape_interval: 5m
    # Este es el intevalo de tiempo en el cual Prometheus preguntara si hay nuevos datos cada 5 minutos para que se actualicen por el contrario se mostraran los datos que se le dieron. 

  ```

### Configuracion del programa Exporter en Grafana 

Ingresamos a la pagina oficial de Grafana con las credenciales de usuario y contraseña que se le asignaron anteriormente.

Buscamos la seccion Dashboard y seleccionamos la opcion de New, damos clic en New Dashboard, posteriormente damos clic en el recuadro de + Add Visualization, seleccionamos la base de datos que utilizaremos, en nuestro caso sera prometheus.

En la parte superior derecha, encontraremos las diferentes formas en las cuales podemos representar nuestros datos, en este caso seleccionaremos Gauge. 

Nos aparecera una grafica sin embargo para poder visualizar nuestras metricas, utilizaremos las labels que definimos en nuestro programa. 

En el caso de los ventiladores, los labels son:

- fan_speed
- fan_status 

En el casode las fuentes de poder, los labels son: 

- power_supply
- supply_status

La ventaja que tiene Grafana es que no solo se tienen estas opciones para visualizar las metricas, por lo que podemos realizar distintas conbinaciones, como por ejemplo:

Introducir estas metricas en el apartado llamado Metrics Browser en Grafana.

Ventiladores:

- avg(fan_speed) by (fan_id) > 0
- avg(fan_status) by (fan_id) > 0


Fuentes de poder:

- avg(power_supply) by (PSU_id) > 0 
- avg(supply_status) by (PSU_id) > 0


Esta metrica nos ayudara a visualizar los datos validos de los cuales tenemos datos que son validos.

Nota: Se recomienda realizar un Dashboard para cada caso.




