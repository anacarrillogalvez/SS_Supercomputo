# Documentacion del Programa Bash para extraer datos especificos de fuentes de poder y ventiladores 

## Descripcion General

"El objetivo de este programa es extraer información específica relacionada con ventiladores y fuentes de poder desde una base de datos, posteriormente estos datos seran introducidos a otro programa para realizar la exportacion de metricas a Prometheus y Grafana.

## Estructura del Script 

### Codigo Fuente 
```bash

#!/bin/bash

n=1
	while [ -e "info$n.dat" ]; do
    ((n++))
done
# Este contador se incrementara para poder generar scripts diferentes y saber que script se genero  

# Nombre  del archivo de salida que se va a generar 
 output_file="info$n.dat"

echo "¿Cual es el nombre del archivo de entrada para comprobar el estado de los Ventiladores y Fuentes de Poder?"
#El usuario ingresara el script que contiene la informacion de los componentes de Hardare 

read file
echo "Archivo ingresado: '$file'"

	if [[ -f "$file" ]]; then

# Extraer información de Power Supply y Fans, incluyendo valores de potencia/porcentaje y estado
grep -E "Power Supply|Fan|PSU" "$file" | awk -F'|' '{gsub(/^[ \t]+|[ \t]+$/, "", $0); print $1 " | " $2 " | " $3}' > "$output_file"

# Extrae líneas del archivo que contienen las palabras clave "Power Supply", "Fan" o "PSU"
# Luego, usa awk para formatear las líneas separadas por '|' eliminando espacios en blanco al inicio y final de cada línea, y conserva solo las tres primeras columnas.

# El resultado se guarda en el archivo de salida especificado.
echo "Información de temperaturas y fuentes de poder extraídas y guardadas en $output_file:"

cat "$output_file"
# Se imprime el programa que genero con sus respectivas salidas.

else
	echo "El archivo de entrada no existe"
	fi
# En caso de que se ejecute la instruccion else, significa que no existe el archivo que se proporciono.

```
