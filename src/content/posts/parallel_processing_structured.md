---
title: Parallel Data Processing with Docker Swarm
published: 2025-07-12
description: Guía completa para dividir, procesar y combinar DataFrames usando un clúster Docker Swarm.
tags: [Docker, Python, NFS, Data Processing, Swarm]
category: Tutorial
draft: false
---


## Objetivo
El objetivo es realizar el procesamiento de un  dataframe aprovechando la capacidad de computo que pueden brindar los nodos del cluster,
trabajando todos ellos concurentemente en  un proceso necesario dentro de un proceso màs grande (este esquema de trabajo se le conoce
 como computaciòn paralela),  se consigue no solo  disponiendo de una arquitectura a nivel fisico y/o a nivel de red que permita 
conectividad entre los nodos del cluster, sino desde un inicio diseñar aplicaciones pensadas para desempeñar sus actividades de 
manera descentralizada, este enfoque de desarrollo que reparte los servicios de una aplicaciòn en unos màs pequeños se le llama 
 arquitectura de micro-servicios y es el enfoque que se llevara a cabo en esta aplicaciòn. Por tanto, el primer paso es crear un 
directorio en el nodo manager al cual tengan acceso todos los nodos trabajadores.

(manager es el termino que se utiliza para referirse al nodo maestro de un cluster creado  con Docker Swarm y trabajadores el termino usando para los nodos esclavos, Swarm  es el orquestador de contenedores por defecto en Docker, y sobre el cual se construira esta aplicaciòn, asì que se parte del hecho de ya haber creado el Swarm con anterioridad y lo que esto implicaca como; una red que permite el trafico de datos libremente entre los nodos, como una red LAN, W-LAN, la asignaciòn de IPS estaticas, que en el caso de Swarm debe hacerse al menos en el nodo maestro.
Para màs informaciòn revisar la documentaciòn oficial de Docker Swarm: https://docs.docker.com/engine/swarm/
)




EL primer paso es crear un volumen compartido al cual puedan acceder todos los nodos, esto se harà por medio de NFS(Network File System), por lo cual se instalara y configurara NFS, esto se harà en el nodo manager(El sistema operativo del manager y los trabajadores es Ubuntu 22.04):


## Paso 1: Instalar y configurar NFS
```bash
sudo apt update
sudo apt install nfs-kernel-server

```
## Paso 2: Construir la imagen y desplegar el servicio
Crear un directorio para compartir:

```bash
sudo mkdir -p /srv/nfs/shared
sudo chown nobody:nogroup /srv/nfs/shared
```

Crear un directorio de montaje:

```bash
sudo mkdir -p /mnt/shared
```

Montar el NFS: Monta el recurso NFS en el directorio que se acaba de montar (No incluir < > )

```bash
sudo mount <IP DEL MANAGER>:/srv/nfs/shared /mnt/shared
```

Verificar la Montura:

```bash
df -h
```

Deberìa verse algo asì: <IP DEL MANAGER>:/srv/nfs/shared  XXG  XXG  XXG  XX% /mnt/shared
La linea "sudo mount <IP DEL MANAGER>:/srv/nfs/shared /mnt/shared" monta manualmente el directorio compartido, pero siempre que se reinicie el dispositivo dejara de estar montado y es necesario ejecutarla de nuevo para que lo este, una forma de automatizar este procedimiento es agregar la configuraciòn en el archivo /etc/fstab. Asì el sistema intentara montarlo automaticamente en cada arranque.
Para hacer esto se debe editar el archivo /etc/fstab, de la siguiente manera:

```bash
sudo nano /etc/fstab
```
Al final del archivo, agregar una línea para el montaje NFS. Reemplazar <IP DEL MANAGER>:/srv/nfs/shared y /mnt/shared con la ruta correcta del servidor y el punto de montaje en cada sistema si es el caso, por ejemplo:

```bash
192.168.18.205:/srv/nfs/shared /mnt/shared nfs defaults 0 0
```
Guardar y salir.
Se puede probar que la configuración es correcta montando todos los sistemas de archivos especificados en /etc/fstab con el siguiente comando y sin la necesidad de reiniciar:

```bash
sudo mount -a
```
Si no hay errores, el recurso NFS debería aparecer montado en /mnt/shared.


Agregar la configuración a /etc/exports para permitir que los nodos accedan al directorio compartido:

```bash
echo "/mnt/shared *(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
```
Si lo anterior causa problemas, considerando problemas como no poder acceder al directorio compartido desde los otros nodos, ejecutar en el manager:
```bash
echo "/srv/nfs/shared *(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
```

Reiniciar el servicio NFS:
```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

Montar el volumen NFS en los nodos

```bash
sudo apt install nfs-common
```

Cear el directorio /mnt/shared en los nodos

```bash
sudo mkdir -p /mnt/shared
```

Montar el directorio compartido en cada nodo:

```bash
sudo mount <IP_DEL_SERVIDOR_NFS>:/srv/nfs/shared /mnt/shared
```


SCRIPT DE PYTHON QUE SE ENCARGA DE FRAGMENTAR EL DATAFRAME EN TRES PARTES, (POR TANTO MÀS ADELANTE SE DISTRIBUIRA EN TRES NODOS EL PROCESAMIENTO DEL DATAFRAME-- EN CASO DE REQUERIR MÀS NODOS SE DEBE MODIFICAR EL SCRIPT EN CONSECUENCIA)



## Script para dividir el DataFrame
Este script divide un archivo CSV en tres partes para procesamiento paralelo.
```python

import pandas as pd
import numpy as np

# Cargar el DataFrame completo desde el volumen compartido
df = pd.read_csv('/data/Datos_Hidrometeorologicos.csv')

# Dividir el DataFrame en 3 partes y guardar cada una en el volumen compartido
df_split = np.array_split(df, 3)
for i, part in enumerate(df_split):
    part.to_csv(f'/data/df_part_{i}.csv', index=False)
    

```
El script anterior crea 3 dataframes a partir del dataframe Datos_Hidrometeorologicos.csv accede a este a traves del directorio /data del contenedor en el cual se monto el directorio /mnt/shared (esto se define durante la creaciòn del servicio  màs adelante).
Los dataframes creados tienen el nombre de df_part_0.csv, df_part_1.csv y df_part_2.csv y los almacena en el
directorio compartido en /mnt/shared, con el fin de descomponer el dataframe original(Grande) en piezas màs pequeñas que puedan ser procesadas individualente por  cada nodo, gracias a que tienen acceso a ellas por medio del directorio compartido, de ahì en principio la importancia de crear el directorio compartido.

A CONTINUCACIÒN SE DEFINE EL DOCKERFILE QUE CONVIERTE EN UNA IMAGEN EL SCRIPT DE PYTHON ANTERIOR
Este instala las dependencias necesarias (pandas, numpy) y copia el script split_dataframe.py dentro de la imagen
de manera que al iniciar el contenedor se ejecuta el script automaticamente


```Dockerfile

# Dockerfile

# Usar una imagen base ligera de Python
FROM python:3.8-slim

# Establecer el directorio de trabajo
WORKDIR /app

# Copiar el script Python al contenedor
COPY split_dataframe.py .

# Instalar las dependencias necesarias
RUN pip install pandas numpy

# Ejecutar el script al iniciar el contenedor
CMD ["python", "split_dataframe.py"]
```


Una vez creados los dos archivos anteriores, en el mismo directorio donde se encuentran se construye
la imagen de Docker con el siguiente comando:


### Construir imagen de Docker para fragmentar el DataFrame
```bash
sudo docker build -t split_dataframe_image .
```


//Esto crea una imagen llamada split_dataframe_image,  se puede corroborar con:

```bash
sudo docker images
```

Esto deberìa mostrar las imagenes existentes y de manera similar a:

REPOSITORY               TAG       IMAGE ID       CREATED             SIZE
split_dataframe_image    latest    e0188a0c2a5a   45 minutes ago      291MB

Creación de un servicio Docker con un Volumen Compartido para esto se  monta el volumen compartido (NFS) en el contenedor
permitiendo que el script acceda a los datos y guarde los archivos divididos en el mismo directorio: /mnt/shared

```bash
sudo docker service create --name fragdata --mount type=bind,source=/mnt/shared,target=/data split_dataframe_image
```


Cuando se crea un servicio con --mount type=bind,source=/mnt/shared,target=/data,
se esta diciendo que el contenido de /mnt/shared (en el host) se montará en /data dentro del contenedor
Esto permite que el contenedor acceda a los datos en /mnt/shared, pero en la ruta /data dentro de su propio sistema de archivos

Por defecto Docker Swarm decide automaticamente en que nodo desplegar un servicio, en un cluster sin ninguna restricciòn o preferencia
Docker Swarm elige cualquiera de los nodos disponibles para el despliegue.
Docker Swarm utiliza un algoritmo de balance de carga que considera el estado de los nodos y la disponibilidad de recursos.
si se desea que el servicio se ejecute en un nodo específico, se puede aplicar una restricción o preferencia utilizando etiquetas de nodo
o configurando manualmente el despliegue en los trabajadores o en el manager.

En el caso de que se necesite desplegar el servico  en otro nodo distinto de donde se creo la imagen, como en este caso la
imagen split_dataframe_image, que se creo en el manager, por tanto solo esta disponible para su despliegue en el manager, para que este disponible en otros nodos se puede publicar la imagen en un registro privado o publico como Docker hub, github,  incluso un registro privado en la misma red de manera que los nodos puedan descargarla.

En este caso se transferira la imagen manualmente a los nodos, usando el directorio NFS creado al inicio,
accesible para todos los nodos por medio de /mnt/shared para exportar por ejemplo, la imagen split_dataframe_image de
manera que sea accesible para todos los nodos se exporta en el directorio compartido como un archivo tar:
```bash
sudo docker save -o /mnt/shared/split_dataframe_image.tar split_dataframe_image:latest
```
Para cargar dicha imagen en los otros nodos, se debe ir a cada nodo donde se requiera la imagen este disponible y cargar la imagen desde el directorio compartido:
```bash
sudo docker load -i /mnt/shared/split_dataframe_image.tar
```
Ahora si es necesario desplegar el servicio en un nodo en especifico, se puede desde un inicio crear el servicio con una restricciòn
para que solo se ejecute en un nodo, pero para esto se le deba asignar a dicho nodo una etiqueta, es recomendable asignar un etiqueta sencilla y facil de recordar a cada nodo:
```bash
sudo docker node update --label-add destino=worker1 worker1
```
Donde, destino=ETIQUETA_ASIGNADA y worker1 el nombre del nodo, se pueden listar los nodos para conocer su nombre:
```bash
sudo docker node ls
```

Para crear el servicio con la restricción para que solo se ejecute en worker1:
```bash
sudo docker service create --name fragdata --constraint 'node.labels.destino == worker1' --mount type=bind,source=/mnt/shared,target=/data split_dataframe_image
```

Para actualizar un servicio ya existente, y se requiere aplicar ciertas restricciones, se deben eliminar las anteriores, para que no interfieran entre ellas, por ejemplo el servicio anterior se creo con la restricciòn de que se ejecute en worker1, si ahora se quiere poner la restricciòn de que se ejecute en el manager,se deben primero eliminar las restricciones actuales:

```bash
sudo docker service update --constraint-rm 'node.labels.destino == worker1' fragdata
```


Ahora se puede asignar la nueva restricciòn, para que el servicio se ejecute con manager(Con anterioridad se le asigno la etiqueta manager al nodo maestro):
```bash
docker service update --constraint-add 'node.labels.destino == manager' fragdata
```

Para confirmar la actualizaciòn se puede revisar el estado del servicio:
```bash
sudo docker service ps fragdata
```

## Script de procesamiento por fragmento
```python


import pandas as pd
import numpy as np
import os
from collections import Counter
import scipy.stats as stats

# Leer el número de partición del entorno
partition = os.getenv('PARTITION', '0')  # Defecto en 0 si no se establece

# Cargar la partición correspondiente del DataFrame
df = pd.read_csv(f'/mnt/shared/df_part_{partition}.csv')

# Variables para almacenar resultados
total_sum = 0
count = 0
max_value = float('-inf')
min_value = float('inf')
values_list = []
sum_squared_diff = 0
rolling_var_sum = 0
percentiles = [10, 25, 50, 75, 90]

# 1. Promedio y varianza
for valor in df['ValorObservado']:
    total_sum += valor
    count += 1
temperatura_promedio = total_sum / count

# 2. Cálculo del máximo y mínimo
for valor in df['ValorObservado']:
    if valor > max_value:
        max_value = valor
    if valor < min_value:
        min_value = valor

# 3. Moda
for valor in df['ValorObservado']:
    values_list.append(valor)
frecuencia = Counter(values_list)
moda = frecuencia.most_common(1)[0][0]

# 4. Cálculo de varianza
for valor in df['ValorObservado']:
    sum_squared_diff += (valor - temperatura_promedio) ** 2
varianza = sum_squared_diff / count

# 5. Desviación estándar
desviacion_estandar = np.sqrt(varianza)

# 6. Calcular percentiles
sorted_values = sorted(values_list)
percentile_values = {}
for p in percentiles:
    percentile_values[p] = np.percentile(sorted_values, p)

# 7. Calcular curtosis y asimetría
curtosis = stats.kurtosis(values_list)
asimetria = stats.skew(values_list)

# 8. Ventana móvil para varianza móvil (ventana de 500 elementos)
window_size = 500
for i in range(window_size, len(values_list)):
    window = values_list[i-window_size:i]  # Subconjunto de la ventana
    rolling_var_sum += np.var(window)

# Guardar resultados en un archivo CSV específico para esta partición
results = {
    "temperatura_promedio": temperatura_promedio,
    "max_value": max_value,
    "min_value": min_value,
    "moda": moda,
    "varianza": varianza,
    "desviacion_estandar": desviacion_estandar,
    "percentiles": percentile_values,
    "curtosis": curtosis,
    "asimetria": asimetria,
    "suma_varianza_movil": rolling_var_sum
}

# Convertir a DataFrame para guardar
results_df = pd.DataFrame([results])
results_df.to_csv(f'/mnt/shared/result_part_{partition}.csv', index=False)

```
Este a partir de la variable de entorno PARTITION a la que le define un valor en el despliegue de màs adelante, determinarà que fragmento del dataframe se procesara


DOCKERFILE QUE CONVIERTE EN UNA IMAGEN EL SCRIP ANTERIOR


## Dockerfile para procesamiento individual
```Dockerfile
# Usa una imagen base de Python
FROM python:3.8-slim


# Establecer el directorio de trabajo
WORKDIR /app

# Copiar el script de procesamiento al contenedor
COPY split_processing.py .

# Instalar las dependencias necesarias
RUN pip install pandas numpy scipy

# Comando por defecto para ejecutar el script
CMD ["python", "split_processing.py"]
```


En este caso no se desplegará la imagen anterior  como un contenedor(run) o un servicio (service create) sino al ser  en esta aplicaciòn 3 servicios en el que cada servicio se desplegara en un nodo diferente,por tanto, se usarà docker-compose para desplegar todos al tiempo màs facilmente, ya que organiza la configuraciòn de cada servicio como sus respectivas variables de entorno (PARTITION) y monta el directorio compartido.




## docker-compose.yml para procesamiento paralelo
```yaml
version: '3.8'
services:
  partition_0:
    image: split_processing_image:latest
    environment:
      - PARTITION=0
    deploy:
      placement:
        constraints: [node.labels.type == manager]
      restart_policy:
        condition: on-failure
    volumes:
      - /mnt/shared:/mnt/shared

  partition_1:
    image: split_processing_image:latest
    environment:
      - PARTITION=1
    deploy:
      placement:
        constraints: [node.labels.type == worker1]
      restart_policy:
        condition: on-failure
    volumes:
      - /mnt/shared:/mnt/shared
      
      
  partition_2:
    image: split_processing_image:latest
    environment:
      - PARTITION=2
    deploy:
      placement:
        constraints: [node.labels.type == worker2]
      restart_policy:
        condition: on-failure
    volumes:
      - /mnt/shared:/mnt/shared
      

```
Para desplegar el compose anterior como un stack(compose pero en el marco de docker swarm), se debe tener en un mismo directorio el Dockerfile, split_processing.py y docker-compose.yml.


se debe convertir el script "split_processing.py" en una imagen esto se hace(de debe tener abierto en el terminal el directorio que contiene los tres archivos anteriores, al menos el Dockerfile y el archivo .py):

```bash
sudo docker build -t split_processing_image .
```

En este caso a diferencia del primer servicio que fragmenta el dataframe, es obligatorio que la imagen recien creada con el nombre "split_processing_image" este disponible para todos los nodos pues en este caso, sì se desplegara un contenedor de esta imagen en cada uno de ellos, el proceso para hacerlo es el mismo mencionado anteriormene.
Guardar en el directorio compartido la imagen en formato .tar:

```bash
sudo docker save -o /mnt/shared/split_processing_image.tar split_processing_image:latest
```

Ahora se debe ir a cada nodo y cargar la imagen:
```bash
sudo docker load -i /mnt/shared/split_processing_image.tar
```
Despuès en el caso de aùn tener abierto en el terminal el directorio que contiene docker-compose.yml, se puede desplegar el stack:
```bash
sudo docker stack deploy -c docker-compose.yml stack_processing
```
Esto desplego un stack con el nombre "stack_processing" conformado por 3 servicios el cual cada uno esta configurado para correr en un nodo diferente.
Despuès de un tiempo, el cual depende del tamaño del dataframe, se crearan 3 archivos con el nombre result_part_0.csv, result_part_1.csv y result_part_2.csv que se almacenaran en el directorio compartido, los cuales contienen un dataframe con el resultado del procesamiento individual de cada fragmento.
Una vez se tienen estos resultados parciales lo unico que falta es un ultimo servicio que se encargue de reunir estos resultados y operarlos de manera  que se conviertan en el resultado total del dataframe original.

## Script para combinar resultados
```python

import pandas as pd
import ast  # Para convertir el string de percentiles a un diccionario

# Leer los archivos de resultados de cada partición
results = []
for i in range(3):
    df = pd.read_csv(f'/mnt/shared/result_part_{i}.csv')
    results.append(df)

# Concatenar los DataFrames para facilidad de acceso
combined_df = pd.concat(results, ignore_index=True)

# Calcular el resultado combinado
combined_result = {
    "temperatura_promedio": combined_df["temperatura_promedio"].mean(),
    "max_value": combined_df["max_value"].max(),
    "min_value": combined_df["min_value"].min(),
    "moda": combined_df["moda"].mode().iloc[0] if not combined_df["moda"].mode().empty else None,
    "varianza": combined_df["varianza"].mean(),
    "desviacion_estandar": combined_df["desviacion_estandar"].mean(),
    "curtosis": combined_df["curtosis"].mean(),
    "asimetria": combined_df["asimetria"].mean(),
    "suma_varianza_movil": combined_df["suma_varianza_movil"].sum()
}

# Promediar los percentiles
percentiles = ["{10: 0, 25: 0, 50: 0, 75: 0, 90: 0}"] * 3  # inicializa en caso de que esté vacío
if "percentiles" in combined_df.columns:
    percentiles_list = [ast.literal_eval(p) for p in combined_df["percentiles"]]
    percentiles_avg = {k: sum(p[k] for p in percentiles_list) / len(percentiles_list) for k in percentiles_list[0]}
    combined_result["percentiles"] = percentiles_avg

# Guardar el resultado combinado en un archivo CSV
combined_result_df = pd.DataFrame([combined_result])
combined_result_df.to_csv('/mnt/shared/combined_results.csv', index=False)

print("Resultado combinado guardado en combined_results.csv")

```
El script anterior crea un archivo llamdo "combined_results.csv" que contiene el resultado final del procesamiento del dataframe


EL SIGUIENTE DOCKERFILE CONVIERTE EN UNA IMAGEN EL SCRIPT ANTERIOR


## Dockerfile para combinar resultados
```Dockerfile
# Usar una imagen de Python
FROM python:3.8-slim


# Establecer el directorio de trabajo
WORKDIR /app

# Copiar el script al contenedor
COPY collect_results.py .

# Instalar pandas
RUN pip install pandas numpy

# Comando por defecto
CMD ["python", "collect_results.py"]

```

Despuès se convierte el script en imagen con:

sudo docker build -t result_dataframe_image .

Esto crea una imagen llamada "result_dataframe_image"

Finalmente se crea un servicio que despliega la imagen anterior en el nodo manger

```bash
sudo docker service create --name dataresult --constraint 'node.labels.destino == manager' --mount type=bind,source=/mnt/shared,target=/mnt/shared result_dataframe_image
```


El servicio se llama dataresult y casì inmediatamente despuès de ejecutarlo con la linea anterior creara el csv con el resultado, ignorar el hecho de que se este reinicado constantemente y mensajes como "Detected task failure" puesto que el servicio se ejecuta muy rapido, y ya que un contenedor culmina su funciòn se cierra, por tanto esta en constante ciclo de reinicio, asì que se puede cerrar el despliegue con ctr C, y se elimina "combined_results.csv" con(teniendo abierto en el terminal el directorio compartido):
```bash
sudo rm combined_results.csv
```
Se cleara uno nuevo casi al instante, mostrando que el servicio se sigue ejecutando correctamente en segundo plano.