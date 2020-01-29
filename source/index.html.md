---
title: API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - shell
  - python
  - conf

toc_footers:
  - <a href='mailto:dl_Uatsbia'>Equipo de UATs y mantenimiento</a>

includes:
  - errors

search: true
---

# Introducción

Bienvenido al catálogo de microservicios Kubernetes del equipo de UATs y mantenimiento.




# MS Log


El microservicio Log permite a las aplicaciones implementar fácilmente la funcionalidad de logging a stdout, schema log de base de datos, blob storage... [[Enlace a Gitlab](http://http://172.30.1.126/)].


## Endpoints

> Puedes apagar el microservicio en cualquier momento enviando una petición POST al endpoint /shutdown o comprobar si está desplegado enviando una petición GET a /ready.

```shell
curl -X POST http://127.0.0.1:9009/shutdown
```

```python
from requests import post

# Encuesta para no operar hasta que el microservicio esté listo.
api_ready_flag = False
while not api_ready_flag:
    try:
        response = get('http://127.0.0.1:9009/ready')

        if response.status_code == 200:
            api_ready_flag = True

    except exceptions.RequestException as e:
        print('No se puede alcanzar')
        sleep(5)

# Petición para apagar el microservicio       
response = post('http://127.0.0.1:9009/shutdown')

# Comprobación.
if response == 200:
    print("Microservicio Log finalizado\n")
```

`POST /log`


`GET /ready`


`GET /health`


`POST /shutdown`

## Registrar mensaje de log

```shell
curl -H "Content-Type: application/json" \
  -d '{\"message\":\"Hello World!\"}' \
  -X POST \
  http://127.0.0.1:9009/log
```

```python
from requests import post, get, exceptions

# Envío de la petición de escritura en log.
response = post('http://127.0.0.1:9009/log', 
                json={'message': '[INFO] Prueba de log 001'}, 
                headers={"Content-Type": "application/json"})

# Comprobación.
if response == 200:
    print("El mensaje ha sido recibido por el servicio de Log\n")
```

Para crear un nuevo log en el microservicio, enviamos una petición POST el endpoint /log.

Atributo | Tipo | Descripción
--------- | ------- | -----------
message | string | Texto del mensaje a incluir en el log.

<aside class="notice">
En la petición debes utilizar el puerto al que has configurado el servicio para que escuche.
</aside>

## Configuración del servicio

```conf
# Contenido de archivo: log-ms-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: log-config
  namespace: > "Añadir nombre" <
data:
  TERMINATION_PATH_LOG: "/usr/src/app/log"
  AZURE_TENANT_ID_LOG: "> Añadir id de la subscripción <"
  AZURE_STORAGE_ACCOUNT_LOG: "> Añadir nombre de la Storage Account de Log <"
  AZURE_STORAGE_CONTAINER_LOG: "> Añadir nombre del Container de Log <"
  AZURE_LOG_FILE_LOG: "> Añadir nombre del Blob de Log <"
  API_HOST_LOG: "0.0.0.0"
  API_PORT_LOG: "> Puerto en el que escucha <"
  TO_BLOB_LOG: "True"
  TO_STDOUT_LOG: "True"
  TO_FILE_LOG: "True"

# Contenido de archivo: log-ms-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: log-secrets
  namespace: > "Añadir nombre" <
type: Opaque
stringData:
  AZURE_CLIENT_ID_LOG: "> Añadir id del service principal con acceso al Storage Account <"
  AZURE_CLIENT_SECRET_LOG: "> Añadir secreto del service principal <"

# Contenido de archivo: log-ms-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-ms
  namespace: > "Añadir nombre" <
spec:
  containers:
  - name: log-ms-ctr
    image: local/uat-log-ms:0.1
    imagePullPolicy: IfNotPresent
    terminationMessagePath: "/usr/src/app/log"

    envFrom:
    - configMapRef:
        name: log-config
    - secretRef:
        name: log-secrets
    livenessProbe:
      httpGet:
        path: /health
        port: 9009
      initialDelaySeconds: 300
      periodSeconds: 5
    readinessProbe:
      httpGet:
        path: /ready
        port: 9009
      initialDelaySeconds: 10
```

El microservicio de log está pensado para ser desplegado en de 1 a n contenedores dentro de un pod kubernetes (job, cronjob y derivados...). Para ello, debe ir acompañado de un ConfigMap que le proporcione los parámetros de configuración como variables de entorno y un Secret, para suministrarle los credenciales de acceso a base de datos y/o storage account en su caso.

### ConfigMap

Parámetro | Defecto | Descripción
--------- | ------- | -----------
API_HOST_LOG | "0.0.0.0" | Dirección IP en la que escucha el microservicio.
API_PORT_LOG | "9009" | Puerto en el que escucha el microservicio.
AZURE_TENANT_ID_LOG | "" | Tenant id del storage account en el que se almacenarán los mensajes.
AZURE_STORAGE_ACCOUNT_LOG | "" | Nombre de la storage account.
AZURE_STORAGE_CONTAINER_LOG | "" | Container dentro de la storage account donde se guardarán los mensajes como blob.
AZURE_LOG_FILE_LOG | "" | Nombre que se le quiera dar al blob de mensajes.
namespace | "" | Namespace de Kubernetes en el que se desplegarán los recursos del microservicio.
SQL_DB_HOST | "" | tes
SQL_DB_DATABASE | "" | tes
TO_BLOB_LOG | "False" | Con valor a True activa en el servicio la opción de guardado de mensajes en blob.
TO_STDOUT_LOG | "False" | Con valor a True activa en el servicio la opción de imprimir mensajes.
TO_FILE_LOG | "False" | Con valor a True activa en el servicio la opción de guardar mensajes en archivo del directorio local del servicio.
TERMINATION_PATH_LOG | "" | Ruta para el archivo de terminación para Kubernetes (debug del servicio).


### Secret

Parámetro | Defecto | Descripción
--------- | ------- | -----------
namespace | "" | Namespace de Kubernetes en el que se desplegarán los recursos del microservicio.
AZURE_CLIENT_ID_LOG | "" | Texto.
AZURE_CLIENT_SECRET_LOG | "" | Texto.
SQL_DB_USERNAME | "" | test
SQL_DB_PASSWORD | "" | tes


## Permisos necesarios

Dependiendo de la funcionalidad del servicio que queramos utilizar, necesitaremos ciertos permisos proporcionados por Arquitectura/CSC. En caso de que queramos que los mensajes se guarden en el schema log de base de datos, necesitaremos un usuario y contraseña para escribir en esas tablas. En el caso de querer guardar los mensajes en un blob, necesitaremos un service principal que pueda escribir en ese storage account.

Funcionalidad | Permiso | Descripción
--------- | --------- | -----------
Escritura en Blob | Storage Account Contributor | Permisos de escritura/lectura en storage account.
Escritura en base de datos | READ/WRITE | Permisos de escritura/lectura en storage account.


# MS Match


```conf
# Contenido de archivo: log-ms-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: log-config
  namespace: > "Añadir nombre" <
data:
  TERMINATION_PATH_LOG: "/usr/src/app/log"
  AZURE_TENANT_ID_LOG: "> Añadir id de la subscripción <"
  AZURE_STORAGE_ACCOUNT_LOG: "> Añadir nombre de la Storage Account de Log <"
  AZURE_STORAGE_CONTAINER_LOG: "> Añadir nombre del Container de Log <"
  AZURE_LOG_FILE_LOG: "> Añadir nombre del Blob de Log <"
  API_HOST_LOG: "0.0.0.0"
  API_PORT_LOG: "> Puerto en el que escucha <"
  TO_BLOB_LOG: "True"
  TO_STDOUT_LOG: "True"
  TO_FILE_LOG: "True"

# Contenido de archivo: log-ms-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: log-secrets
  namespace: > "Añadir nombre" <
type: Opaque
stringData:
  AZURE_CLIENT_ID_LOG: "> Añadir id del service principal con acceso al Storage Account <"
  AZURE_CLIENT_SECRET_LOG: "> Añadir secreto del service principal <"

# Contenido de archivo: log-ms-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-ms
  namespace: > "Añadir nombre" <
spec:
  containers:
  - name: log-ms-ctr
    image: local/uat-log-ms:0.1
    imagePullPolicy: IfNotPresent
    terminationMessagePath: "/usr/src/app/log"

    envFrom:
    - configMapRef:
        name: log-config
    - secretRef:
        name: log-secrets
    livenessProbe:
      httpGet:
        path: /health
        port: 9009
      initialDelaySeconds: 300
      periodSeconds: 5
    readinessProbe:
      httpGet:
        path: /ready
        port: 9009
      initialDelaySeconds: 10
```

> `m`.

El microservicio Log permite a las aplicaciones implementar fácilmente la funcionalidad de logging a stdout, base de datos, blob storage... [[Enlace a Gitlab](http://http://172.30.1.126/)].

Parámetros de ConfigMap log-config:

Parámetro | Defecto | Descripción
--------- | ------- | -----------
namespace | "" | Texto.
API_HOST_LOG | "0.0.0.0" | Texto.
API_PORT_LOG | "9009" | Texto.
TO_BLOB_LOG | "True" | Texto.
TO_STDOUT_LOG | "True" | Texto.
TO_FILE_LOG | "True" | Texto.
TERMINATION_PATH_LOG | "True" | Texto.
AZURE_TENANT_ID_LOG | "True" | Texto.
AZURE_STORAGE_ACCOUNT_LOG | "True" | Texto.
AZURE_STORAGE_CONTAINER_LOG | "True" | Texto.
AZURE_LOG_FILE_LOG | "True" | Texto.

Parámetros de Secret log-secrets:

Parámetro | Defecto | Descripción
--------- | ------- | -----------
namespace | "" | Texto.
AZURE_CLIENT_ID_LOG | "0.0.0.0" | Texto.
AZURE_CLIENT_SECRET_LOG | "9009" | Texto.

<aside class="notice">
You must replace <code>meowmeowmeow</code> with your personal API key.
</aside>

<aside class="success">
En la petición debes utilizar el puerto al que has configurado el servicio para que escuche.
</aside>


# MS Copy


```conf
# Contenido de archivo: log-ms-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: log-config
  namespace: > "Añadir nombre" <
data:
  TERMINATION_PATH_LOG: "/usr/src/app/log"
  AZURE_TENANT_ID_LOG: "> Añadir id de la subscripción <"
  AZURE_STORAGE_ACCOUNT_LOG: "> Añadir nombre de la Storage Account de Log <"
  AZURE_STORAGE_CONTAINER_LOG: "> Añadir nombre del Container de Log <"
  AZURE_LOG_FILE_LOG: "> Añadir nombre del Blob de Log <"
  API_HOST_LOG: "0.0.0.0"
  API_PORT_LOG: "> Puerto en el que escucha <"
  TO_BLOB_LOG: "True"
  TO_STDOUT_LOG: "True"
  TO_FILE_LOG: "True"

# Contenido de archivo: log-ms-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: log-secrets
  namespace: > "Añadir nombre" <
type: Opaque
stringData:
  AZURE_CLIENT_ID_LOG: "> Añadir id del service principal con acceso al Storage Account <"
  AZURE_CLIENT_SECRET_LOG: "> Añadir secreto del service principal <"

# Contenido de archivo: log-ms-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-ms
  namespace: > "Añadir nombre" <
spec:
  containers:
  - name: log-ms-ctr
    image: local/uat-log-ms:0.1
    imagePullPolicy: IfNotPresent
    terminationMessagePath: "/usr/src/app/log"

    envFrom:
    - configMapRef:
        name: log-config
    - secretRef:
        name: log-secrets
    livenessProbe:
      httpGet:
        path: /health
        port: 9009
      initialDelaySeconds: 300
      periodSeconds: 5
    readinessProbe:
      httpGet:
        path: /ready
        port: 9009
      initialDelaySeconds: 10
```

> `m`.

El microservicio Log permite a las aplicaciones implementar fácilmente la funcionalidad de logging a stdout, base de datos, blob storage... [[Enlace a Gitlab](http://http://172.30.1.126/)].

Parámetros de ConfigMap log-config:

Parámetro | Defecto | Descripción
--------- | ------- | -----------
namespace | "" | Texto.
API_HOST_BLOB | "0.0.0.0" | Texto.
API_PORT_BLOB | "9009" | Texto.
TERMINATION_PATH_BLOB | "True" | (Opcional).
AZURE_TENANT_ID_BLOB | "True" | Texto.
AZURE_STORAGE_ACCOUNT_BLOB | "True" | Texto.

Parámetros de Secret log-secrets:

Parámetro | Defecto | Descripción
--------- | ------- | -----------
namespace | "" | Texto.
AZURE_CLIENT_ID_BLOB | "" | Texto.
AZURE_CLIENT_SECRET_BLOB | "" | Texto.

<aside class="notice">
You must replace <code>meowmeowmeow</code> with your personal API key.
</aside>


> POST /copy

Body de la petición:

Parámetro | Defecto | Descripción
--------- | ------- | -----------
from_blob | "" | Texto.
from_local | "" | Texto.
to_blob | "" | Texto.
to_local | "" | (Opcional).

Si se especifican varios from a la vez, ¿cuál se prioriza?




# MS PBIRefresh



> Para utilizar este servicio, incluye este código en tu proyecto:


```conf
# Contenido de archivo: log-ms-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: log-config
  namespace: > "Añadir nombre" <
data:
  TERMINATION_PATH_LOG: "/usr/src/app/log"
  AZURE_TENANT_ID_LOG: "> Añadir id de la subscripción <"
  AZURE_STORAGE_ACCOUNT_LOG: "> Añadir nombre de la Storage Account de Log <"
  AZURE_STORAGE_CONTAINER_LOG: "> Añadir nombre del Container de Log <"
  AZURE_LOG_FILE_LOG: "> Añadir nombre del Blob de Log <"
  API_HOST_LOG: "0.0.0.0"
  API_PORT_LOG: "> Puerto en el que escucha <"
  TO_BLOB_LOG: "True"
  TO_STDOUT_LOG: "True"
  TO_FILE_LOG: "True"

# Contenido de archivo: log-ms-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: log-secrets
  namespace: > "Añadir nombre" <
type: Opaque
stringData:
  AZURE_CLIENT_ID_LOG: "> Añadir id del service principal con acceso al Storage Account <"
  AZURE_CLIENT_SECRET_LOG: "> Añadir secreto del service principal <"

# Contenido de archivo: log-ms-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-ms
  namespace: > "Añadir nombre" <
spec:
  containers:
  - name: log-ms-ctr
    image: local/uat-log-ms:0.1
    imagePullPolicy: IfNotPresent
    terminationMessagePath: "/usr/src/app/log"

    envFrom:
    - configMapRef:
        name: log-config
    - secretRef:
        name: log-secrets
    livenessProbe:
      httpGet:
        path: /health
        port: 9009
      initialDelaySeconds: 300
      periodSeconds: 5
    readinessProbe:
      httpGet:
        path: /ready
        port: 9009
      initialDelaySeconds: 10
```

> `m`.

El microservicio Log permite a las aplicaciones implementar fácilmente la funcionalidad de logging a stdout, base de datos, blob storage... [[Enlace a Gitlab](http://http://172.30.1.126/)].

Parámetros de ConfigMap log-config:

Parámetro | Defecto | Descripción
--------- | ------- | -----------
namespace | "" | Texto.
API_HOST_LOG | "0.0.0.0" | Texto.
API_PORT_LOG | "9009" | Texto.
TO_BLOB_LOG | "True" | Texto.
TO_STDOUT_LOG | "True" | Texto.
TO_FILE_LOG | "True" | Texto.
TERMINATION_PATH_LOG | "True" | Texto.
AZURE_TENANT_ID_LOG | "True" | Texto.
AZURE_STORAGE_ACCOUNT_LOG | "True" | Texto.
AZURE_STORAGE_CONTAINER_LOG | "True" | Texto.
AZURE_LOG_FILE_LOG | "True" | Texto.

Parámetros de Secret log-secrets:

Parámetro | Defecto | Descripción
--------- | ------- | -----------
namespace | "" | Texto.
AZURE_CLIENT_ID_LOG | "0.0.0.0" | Texto.
AZURE_CLIENT_SECRET_LOG | "9009" | Texto.

<aside class="notice">
You must replace <code>meowmeowmeow</code> with your personal API key.
</aside>

## Refrescar informe

```shell
# With shell, you can just pass the correct header with each request
curl "http://127.0.0.1:puerto/ready"
  -H "Content-Type: application/json"
```

```python
from requests import post, get, exceptions

# Comprobación del estado del microservicio.
api_ready_flag = False
while not api_ready_flag:
    try:
        response = get('http://127.0.0.1:9009/ready')

        if response.status_code == 200:
            api_ready_flag = True

    except exceptions.RequestException as e:
        print('No se puede alcanzar')
        sleep(5)

# Envío de la petición de escritura en log.
response = post('http://127.0.0.1:9009/log', 
                json={'message': '[INFO] Prueba de log 001'}, 
                headers={"Content-Type": "application/json"})

# Comprobación.
if response == 200:
    print("El mensaje ha sido recibido por el servicio de Log\n")
```


Para refrescar un informe del servicio online de PowerBI, necesitamos conocer el nombre del informe desplegado y el nombre del workspace en el que se ha desplegado.


### Solicitud HTTP

`POST http://127.0.0.1:PUERTO/refresh`

### Cuerpo de la solicitud HTTP

Parámetro | Tipo | Descripción
--------- | ------- | -----------
workspace | String | Nombre del workspace.
informe | String | Nombre del informe.


<aside class="success">
Recuerda — en la petición debes utilizar el puerto al que has configurado el servicio para que escuche.
</aside>

## Permisos Arquitectura necesarios


Descripción.

### Service principal

Para ejecutar la pieza, necesitamos suministrarle los credenciales de un service principal con los siguientes permisos.

Permiso | Descripción
--------- | -----------
Power BI Service.Group.Read | Permisos de lectura en los grupos a los que pertenece.
Power BI Service.Group.Read.All | Permisos de escritura/lectura en storage account.
Power BI Service.Workspace.Read.All | Permisos de escritura/lectura en storage account.
Power BI Service.Report.Read.All | Permisos de escritura/lectura en storage account.
Power BI Service.Dataset.Read.All | Permisos de escritura/lectura en storage account.


# MS SQL


El microservicio Log permite a las aplicaciones implementar fácilmente la funcionalidad de logging a stdout, schema log de base de datos, blob storage... [[Enlace a Gitlab](http://http://172.30.1.126/)].


## Endpoints

> Puedes apagar el microservicio en cualquier momento enviando una petición POST al endpoint /shutdown o comprobar si está desplegado enviando una petición GET a /ready.

```shell
curl -X POST http://127.0.0.1:9009/shutdown
```

```python
from requests import post

# Encuesta para no operar hasta que el microservicio esté listo.
api_ready_flag = False
while not api_ready_flag:
    try:
        response = get('http://127.0.0.1:9009/ready')

        if response.status_code == 200:
            api_ready_flag = True

    except exceptions.RequestException as e:
        print('No se puede alcanzar')
        sleep(5)

# Petición para apagar el microservicio       
response = post('http://127.0.0.1:9009/shutdown')

# Comprobación.
if response == 200:
    print("Microservicio Log finalizado\n")
```

`POST /log`


`GET /ready`


`GET /health`


`POST /shutdown`

## Registrar mensaje de log

```shell
curl -H "Content-Type: application/json" \
  -d '{\"message\":\"Hello World!\"}' \
  -X POST \
  http://127.0.0.1:9009/log
```

```python
from requests import post, get, exceptions

# Envío de la petición de escritura en log.
response = post('http://127.0.0.1:9009/log', 
                json={'message': '[INFO] Prueba de log 001'}, 
                headers={"Content-Type": "application/json"})

# Comprobación.
if response == 200:
    print("El mensaje ha sido recibido por el servicio de Log\n")
```

Para crear un nuevo log en el microservicio, enviamos una petición POST el endpoint /log.

Atributo | Tipo | Descripción
--------- | ------- | -----------
message | string | Texto del mensaje a incluir en el log.

<aside class="notice">
En la petición debes utilizar el puerto al que has configurado el servicio para que escuche.
</aside>

## Configuración del servicio

```conf
# Contenido de archivo: log-ms-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: log-config
  namespace: > "Añadir nombre" <
data:
  TERMINATION_PATH_LOG: "/usr/src/app/log"
  AZURE_TENANT_ID_LOG: "> Añadir id de la subscripción <"
  AZURE_STORAGE_ACCOUNT_LOG: "> Añadir nombre de la Storage Account de Log <"
  AZURE_STORAGE_CONTAINER_LOG: "> Añadir nombre del Container de Log <"
  AZURE_LOG_FILE_LOG: "> Añadir nombre del Blob de Log <"
  API_HOST_LOG: "0.0.0.0"
  API_PORT_LOG: "> Puerto en el que escucha <"
  TO_BLOB_LOG: "True"
  TO_STDOUT_LOG: "True"
  TO_FILE_LOG: "True"

# Contenido de archivo: log-ms-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: log-secrets
  namespace: > "Añadir nombre" <
type: Opaque
stringData:
  AZURE_CLIENT_ID_LOG: "> Añadir id del service principal con acceso al Storage Account <"
  AZURE_CLIENT_SECRET_LOG: "> Añadir secreto del service principal <"

# Contenido de archivo: log-ms-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-ms
  namespace: > "Añadir nombre" <
spec:
  containers:
  - name: log-ms-ctr
    image: local/uat-log-ms:0.1
    imagePullPolicy: IfNotPresent
    terminationMessagePath: "/usr/src/app/log"

    envFrom:
    - configMapRef:
        name: log-config
    - secretRef:
        name: log-secrets
    livenessProbe:
      httpGet:
        path: /health
        port: 9009
      initialDelaySeconds: 300
      periodSeconds: 5
    readinessProbe:
      httpGet:
        path: /ready
        port: 9009
      initialDelaySeconds: 10
```

El microservicio de log está pensado para ser desplegado en de 1 a n contenedores dentro de un pod kubernetes (job, cronjob y derivados...). Para ello, debe ir acompañado de un ConfigMap que le proporcione los parámetros de configuración como variables de entorno y un Secret, para suministrarle los credenciales de acceso a base de datos y/o storage account en su caso.

### ConfigMap

Parámetro | Defecto | Descripción
--------- | ------- | -----------
API_HOST_LOG | "0.0.0.0" | Dirección IP en la que escucha el microservicio.
API_PORT_LOG | "9009" | Puerto en el que escucha el microservicio.
AZURE_TENANT_ID_LOG | "" | Tenant id del storage account en el que se almacenarán los mensajes.
AZURE_STORAGE_ACCOUNT_LOG | "" | Nombre de la storage account.
AZURE_STORAGE_CONTAINER_LOG | "" | Container dentro de la storage account donde se guardarán los mensajes como blob.
AZURE_LOG_FILE_LOG | "" | Nombre que se le quiera dar al blob de mensajes.
namespace | "" | Namespace de Kubernetes en el que se desplegarán los recursos del microservicio.
SQL_DB_HOST | "" | tes
SQL_DB_DATABASE | "" | tes
TO_BLOB_LOG | "False" | Con valor a True activa en el servicio la opción de guardado de mensajes en blob.
TO_STDOUT_LOG | "False" | Con valor a True activa en el servicio la opción de imprimir mensajes.
TO_FILE_LOG | "False" | Con valor a True activa en el servicio la opción de guardar mensajes en archivo del directorio local del servicio.
TERMINATION_PATH_LOG | "" | Ruta para el archivo de terminación para Kubernetes (debug del servicio).


### Secret

Parámetro | Defecto | Descripción
--------- | ------- | -----------
namespace | "" | Namespace de Kubernetes en el que se desplegarán los recursos del microservicio.
AZURE_CLIENT_ID_LOG | "" | Texto.
AZURE_CLIENT_SECRET_LOG | "" | Texto.
SQL_DB_USERNAME | "" | test
SQL_DB_PASSWORD | "" | tes


## Permisos necesarios

Dependiendo de la funcionalidad del servicio que queramos utilizar, necesitaremos ciertos permisos proporcionados por Arquitectura/CSC. En caso de que queramos que los mensajes se guarden en el schema log de base de datos, necesitaremos un usuario y contraseña para escribir en esas tablas. En el caso de querer guardar los mensajes en un blob, necesitaremos un service principal que pueda escribir en ese storage account.

Funcionalidad | Permiso | Descripción
--------- | --------- | -----------
Escritura en Blob | Storage Account Contributor | Permisos de escritura/lectura en storage account.
Escritura en base de datos | READ/WRITE | Permisos de escritura/lectura en storage account.
