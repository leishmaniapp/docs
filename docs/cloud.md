# 🌎 Cloud
Leishmaniapp es una arquitectura de almacenamiento de información diagnóstica y procesamiento de imágenes, _LeishmaniappCloudServicesV2_ es el nombre asignado al conjunto de servicios en nube capaces de proveer esta funcionalidad

## Vista General de la Arquitectura
_LeishmaniappCloudServicesV2_ es una arquitectura basada en [microservicios](https://aws.amazon.com/es/microservices/), cada uno de estos servicios se encarga de una funcionalidad específica, opera de manera independiente y se comunica con los demás elementos de la arquitectura a través de una API [gRPC](https://grpc.io/) y un esquema de datos predefinido. (Visite [_Protobuf Schema_](schema.md) para más información acerca de la API.)

![Diagrama de infrastructura de nube](assets/cloud/cloud.png)

## Componentes
La arquitectura _LeishmaniappCloudServicesV2_ atiende a un cliente (usualmente, pero no limitado a, [_aplicación de Leishmaniapp_](application.md)) a través de la API de gRPC utilizando el protocolo HTTP/2 (sin encriptación). Las peticiones deberán ser atendidas por un [Gateway](#service_gateway) a través de un único puerto TCP y redirigidas al microservicio capaz de atender la solicitud, el gateway también provee un servicio de _health check_ con el cual los clientes pueden validar la conexión con el servidor.

Cada uno de los microservicios atiende un único servicio definido en [protobuf_schema.](schema.md), estos son: [Autenticación](#auth_service), [Almacenamiento de Diagnósticos](#diagnoses_service), [Almacenamiento de muestras](#samples_service) y [Análisis](#analysis_service). Este último mantiene un stream de datos, a través de una conexión bidireccional full-duplex entre el cliente y el servidor, con la cual múltiples peticiones de análisis pueden ser enviadas y sus resultados entregados de manera asíncrona, esto difiere con los demás servicios que son de tipo _unicast_.

Las peticiones de análisis son enviadas a un [servicio de mensajería](#servicio-de-mensajeria) que entregará la petición a un servicio de análisis específico para cada enfermedad a diagnosticar, estos servicios pueden ser implementados en cualquier lenguaje de programación y se incorporan en la arquitectura a través de un [servicio de encapsulamiento](#model_wrapper), para ello los binarios encargados del análisis deben estar en conformidad con el formato [ALEF](models.md#alef-adapter-layer-exec-format) (Visite [_Modelos de Detección_](models.md) para más información). Los resultados de análisis son enviados al servicio de mensajería quien entregará los resultados al servicio de análisis, se guardará la información de la muestra a través del servicio de almacenamiento de muestras y se enviarán los resultados al cliente de manera asíncrona.

### Microservicios
> 💡 Los repositorios de cada uno de los servicios hacen uso de  _submódulos de git_, no olvide inicializarlos con `git submodule update --init --recursive` después de clonar el repositorio.

> 💡 Para la creación de las imagenes de contenedores utilize el comando `docker build -t <nombre> -f deployment/Containerfile .`, donde `<nombre>` es reemplazado por el nombre que se le asignará a la imagen.

Cada uno de los microservicios anteriormente mencionados son descritos a detalle en esta sección.

#### service_gateway
> 📚 Link del repositorio: [https://github.com/leishmaniapp/service_gateway](https://github.com/leishmaniapp/service_gateway).

> 🐋 Utilice _leishmaniapp.cloud/service_gateway_ como nombre de la imagen de contenedores

Es un _gateway_ para gRPC personalizado con el cual un cliente puede acceder a todos los servicios ofrecidos por _LeishmaniappCloudServicesV2_ a través de un único punto de acceso.

Este servicio tiene como responsabilidad redirigir las peticiones del cliente al _host_ correspondiente capaz de suplir el requerimiento, la lista de servicios ofrecidos y la información del _host_ asociado se debe de especificar en la sección de **'resources'** en el archivo de configuración **config.yaml**.

Estos servicios también pueden ser protegidos de accesos no autorizados mediante el atributo booleano **'protected'** asociado a cada uno de los items de **'resources'** en el archivo de configuración, el _gateway_ verifica la autenticidad del _token_ de acceso del usuario haciendo uso del servicio _leishmaniapp.cloud.auth.AuthService_ cuyo _host_ debe de ser especificado en la sección **'authentication'** del archivo de configuración.

##### Configuración

A continuación un ejemplo del archivo de configuración **config.yaml**, este archivo es utilizado en la [infrastructura con Kubernetes](#infrastructura-kubernetes)

```yaml
# Server configuration
server:
  # Service port
  port: 9871

# Authentication service
authentication:
  timeout_sec: 5
  host: grpc-auth.leishmaniapp.svc.cluster.local
  port: 9875

# List of services provided by the service
resources:
  - service: leishmaniapp.cloud.auth.AuthService
    host: grpc-auth.leishmaniapp.svc.cluster.local
    port: 9875
    # Does not require the user to be authenticated
    protected: false

  - service: leishmaniapp.cloud.diagnoses.DiagnosesService
    host: grpc-diagnoses.leishmaniapp.svc.cluster.local
    port: 9873
    # User must provide auth token
    protected: true
    
  - service: leishmaniapp.cloud.analysis.AnalysisService
    host: grpc-analysis.leishmaniapp.svc.cluster.local
    port: 9872
    protected: true
```


#### auth_service

> 📚 Link del repositorio: [https://github.com/leishmaniapp/auth_service](https://github.com/leishmaniapp/auth_service).

> 🐋 Utilice _leishmaniapp.cloud/auth_service_ como nombre de la imagen de contenedores


Este servicio valida la identidad de un cliente (especialista) a través de un _token de autenticación_ [JWT](https://jwt.io/), la lista de especialistas registrados se obtiene de una base de datos (requiere una conexión a [mongodb](https://www.mongodb.com/)) y no existe un servicio para la creación de usuarios.

El servicio tiene las siguiente propiedades

* El identificador del usuario es su _email_
* La contraseña es encriptada con el algoritmo _bcrypt_
* Solo un _token_ de autenticación es válido en dado momento, este token puede ser invalidado y reemplazado por uno nuevo
* Para validar la autenticación se debe validar el _token_ y verificar que coincida con el que se encuentra registrado en la base de datos

Ejemplo de un registro en la base de datos:

```json
{
  "email": "leishmaniapp@javeriana.edu.co",
  "name": "Default User",
  "passwordHash": "$2a$12$p21C4UmRQEZqX/KBHISuk.dmvIsiiywYl9Cj49BXZFgtL/81H9KyW",
  "diseases": ["mock.spots", "malaria.romanowsky", "leishmaniasis.giemsa"],
  "token": null
}
```

##### Configuración

Para configurar la base de datos y demás propiedades del servicio, se debe de proveer el siguiente archivo __.env__
```bash
# Run in debug mode (optional, default=false)
DEBUG=0

# Where the gRPC service will be accesible from (optional, default=9875)
SERVER_PORT=9875

# Database connection string
# Remove double quotes for container .env
DATABASE_URL="mongodb+srv://[username:password@]host[/[defaultauthdb][?options]]"

# Database operations timeout
DATABASE_TIMEOUT=10

# Database source name
DATABASE_NAME=authDB

# Database table or collection
DATABASE_TABLE=users

# Web security HMAC secret
# Remove double quotes for container .env
SECRET_KEY="your-secret-key"
```

#### diagnoses_service
> 📚 Link del repositorio: [https://github.com/leishmaniapp/diagnoses_service](https://github.com/leishmaniapp/diagnoses_service).

> 🐋 Utilice _leishmaniapp.cloud/diagnoses_service_ como nombre de la imagen de contenedores

Este servicio almacena los diagnósticos finalizados por el especialista, estos diagnósticos incluyen la información del paciente y la cantidad de elementos de interés encontrados así como cualquier otra observación relevante para la historia clínica del paciente.

##### Configuración

La información debe de ser consolidada en una base de datos _mongodb_ y los parámetros de conexión deben de ser declarados a través de variables de entorno o un archivo _.env_.

```bash
# Run in debug mode (Optional, Default=0)
DEBUG=0

# Where the gRPC service will be accesible from (Optional, Default=9873)
SERVER_PORT=9873

# Database connection string
DATABASE_URL=mongodb+srv://[username:password@]host[/[defaultauthdb][?options]]

# Database source name
DATABASE_NAME=diagnosesDB

# Database table or collection
DATABASE_TABLE=diagnoses

# Database operations timeout
DATABASE_TIMEOUT=10
```

#### samples_service
> 📚 Link del repositorio: [https://github.com/leishmaniapp/samples_service](https://github.com/leishmaniapp/samples_service).

> 🐋 Utilice _leishmaniapp.cloud/samples_service_ como nombre de la imagen de contenedores

Este servicio se encarga de almacenar los metadatos correspondientes a las muestras de análisis. Actúa como un repositorio central donde se registran y mantienen los resultados de todas las muestras procesadas, que pueden ser utilizadas para diagnósticos posteriores o entrenamiento de nuevos modelos de diagnóstico.

##### Configuración

Este servicio requiere de una conexión a una base de datos _mongodb_, así como un directorio en donde almacenar las imágenes diagnósticas. Esta configuración debe de ser declarada a través de variables de entorno o un archivo _.env_

```bash
# Run in debug mode (Optional, Default=0)
DEBUG=0

# Where the gRPC service will be accesible from (Optional, Default=9874)
SERVER_PORT=9874

# Database connection string
# Remove double quotes for container .env
DATABASE_URL=mongodb+srv://[username:password@]host[/[defaultauthdb][?options]]

# Database operations timeout
DATABASE_TIMEOUT=10

# Database source name
DATABASE_NAME=samplesDB

# Database table or collection
DATABASE_TABLE=samples

# Where to store images
# Remove double quotes for container .env
STORAGE_PATH="./storage"
```


#### analysis_service
> 📚 Link del repositorio: [https://github.com/leishmaniapp/analysis_service](https://github.com/leishmaniapp/analysis_service).

> 🐋 Utilice _leishmaniapp.cloud/analysis_service_ como nombre de la imagen de contenedores

Este servicio recibe las imágenes y otros datos de muestras y los envía a través de un [servicio de mensajería](#servicio-de-mensajeria) al [modelo de análisis](models.md) correspondiente, los metadatos y resultados de las imágenes son enviados al [servicio de muestras](#samples_service) para su almacenamiento y los resultados de análisis son entregados al cliente de manera asíncrona a través de una conexión bidireccional.

##### Headers HTTP/2
Este servicio requiere del _pseudo-header HTTP/2_ `:from` el cual contiene el _email_ del especialista, esto con el fin de identificar y validar las peticiones sin necesidad de inspeccionar el contenido del mensaje y así poder asociarlos a la cola correspondiente.

El [gateway](#service_gateway) es el servicio responsable de validar que el _header From_ coincida con el campo _email_ del token de autenticación del _header Authorization_

##### Órden de operación

Una vez el cliente inicia una conexión con el servicio de análisis:

1. Se validará la conexión a través del _pseudo-header_ HTTP/2 `:from`, el cual debe de contener el _email_ del especialista
2. Se verificará con el _servicio de muestras_ si hay resultados pendientes de entregar y se los enviará
3. Se creará una cola de resultados específica para el cliente. (Véase [resultados en el servicio de mensajería](#resultados))
4. Se suscribirá a la cola específica del cliente y a la cola _global_.

Una vez el cliente envía una solicitud de análisis:

1. Se verifica la petición
2. Se almacenará mediante el _servicio de muestras_ la imagen y sus metadatos
3. Se enviará la petición al _servicio de mensajería_ para que sea analizado por algún _modelo_.

Una vez el modelo retorna los resultados de análisis:

* Si se reciben a través de la cola del cliente y la conexión con el cliente está activa, se marca el análisis como entregado, se envían los resultados a través del canal de comunicación y se actualizan los metadatos a través del _servicio de muestras_.
* Si se reciben a través de la cola del cliente pero la conexión ha sido finalizada, se rechaza la entrega de resultados para que sean entregados a la cola global.
* Si se reciben a través de la cola _global_, se marca el análisis como pendiente de entrega y se actualizan los metadatos a través del _servicio de muestras_. 

Una vez el cliente abandona la conexión
* Se elimina la cola del cliente

##### Configuración
Este servicio requiere las credenciales de acceso al _broker_ para el _servicio de mensajería_ y la dirección de _host_ del _servicio de muestras_ para el almacenamiento de metadatos.

Esta configuración debe de ser especificada a través de variables de entorno o un archivo _.env_, a continuación un ejemplo de configuración mediante un archivo _.env_

```bash
# Run in debug mode (Optional, Default=0)
DEBUG=0

# Where the gRPC service will be accesible from (Optional, Default=9872)
SERVER_PORT=9872

# Amount of maximum buffered requests (Optional, Default=10)
BUFFER_SIZE=10

# Client queue TTL (Optional, Default=60)
QUEUE_TTL_SEC=60

# Analysis request timeout
REQUEST_TIMEOUT_SEC=1200

# Remote service timeout
SERVICE_TIMEOUT_SEC=10

# AMQP exchange URL for the analysis requests and results
MESSAGE_BROKER=amqp://user:pass@localhost:5672/

# URL to the samples service RPC service
SAMPLES_SERVICE=localhost:9874
```

#### model_wrapper
> 📚 Link del repositorio: [https://github.com/leishmaniapp/model_wrapper](https://github.com/leishmaniapp/model_wrapper).

> 🐋 Utilice _leishmaniapp.cloud/model_wrapper_ como nombre de la imagen de contenedores

Permite la integración entre los [modelos de análisis](models.md) y _LeishmaniappCloudServicesV2_ sin que estos tengan que implementar código relacionado a la recepción y entrega de mensajes mediante el [servicio de mensajería](#servicio-de-mensajeria).

##### Funcionamiento

Los modelos de análisis deben de ser ejecutables en conformidad con el formato [ALEF](models.md#alef-adapter-layer-exec-format), la invocación de estos ejecutables se realiza a través de la _llamada al sistema [exec](https://en.wikipedia.org/wiki/Exec_(system_call))_ o similiar (`std::process::Command::new` en Rust). Los resultados de análisis son recolectados de la _salida estándar (**stdout**)_ si el resultado de ejecución es 0, el output del _error estándar (**stderr**)_ es ignorado durante la ejecución

##### Configuración
El archivo de configuración **config.yaml** determinará el _id_ de la enfermedad a diagnosticar, la versión del modelo y el _path_ de los _modelos de análisis_ que serán ejecutados, ya sea en serie o paralelo.

A continuación un archivo de configuración extraído de la enfermedad [_leishmaniasis.giemesa_](https://github.com/leishmaniapp/leishmaniasis-giemsa-disease/tree/main). En esta configuración, dos modelos de análisis (_macrophages_ y _parasites_) se ejecutan en paralelo con la misma imagen de entrada.

```yaml
disease: leishmaniasis.giemsa
version: "2.0.0"
exec:
  - path: "/usr/local/bin/python"
    args: ["./macrophages/src/alef.py"]

  - path: "/usr/local/bin/python"
    args: ["./parasites/src/alef.py"]
```

A continuación un ejemplo genérico explorando todas las capacidades del archivo de configuración

```yaml
# Which disease is this model going to identify, use the disease ID
disease: "mock.chain"

# Model version
version: "1.0.0"

# List of executables that need to be run (must comply to the AdapterLayerExecFormat)
# They can be executed in parallel by defining them as members of the array
# The input file for this top-level executables is going to be the unaltered
exec:
  # Absolute path to the executable
  - path: "path/to/binary/model_a"

    # The output of this model is going to be included in the response
    # (Optional), default value is true
    include: true

    # Only mock.chain:element.a1 and mock.chain:element.a3 are going to be appended to the response
    # Use 'null' to disable the filter and append all the elements
    # (Optional), default value is null
    filter: ["element.a1", "element.a3"]

    # Arguments to pass to the path (this ones go before --alef-in and --alef-out)
    args: ["--sensitivity=0.8", "-d", "logs.txt"]

    # Specify a list of models that will run after the parent finished
    # The input file for this models will be the output file of the parent and thus the parent must accept --alef-out parameter
    # (Optional), default value is null
    next:
      - path: "path/to/binary/model_b"
        include: true
        # All the outputs from this model will be appended to the output
        filter: null
        args: []
        next:
          - path: "path/to/binary/model_c"
            include: false
            filter: null
            args: []
            next: null # No more models are going to be called after this one

      - path: "path/to/binary/model_d"
        include: false
        # Filter is ignored when include is false
        filter: []
        args: []
        next: null

  # This model will be executed at the same time as model_a because they are siblings in the array
  - path: "/usr/bin/python3"
    args: ["path/to/model_e", "--debug"]
    include: true
    filter: null
    next: null
```
###### Esquema para el archivo de configuración **config.yaml**

```yaml
disease: string
version: string
exec: ModelExecutor[]
```

ModelExecutor type

| field   | type                          | description                                                                                                                                       |
| ------- | ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| path    | string                        | Absolute path to the executable                                                                                                                   |
| include | bool                          | Will the results of this model be appended to the respnse?                                                                                        |
| filter  | string[] _or_ null            | Which elements returned by the model will be appended to the response                                                                             |
| args    | string[]                      | Additional arguments to pass to the model                                                                                                         |
| next    | **ModelExecutor**[] _or_ null | List with future analysis models to call when the current one finishes, the output of these analysis models will be the output of the current one |

### Servicio de Mensajería
El servicio de mensajería hace uso del protocolo [AMQP 0-9-1](https://www.rabbitmq.com/tutorials/amqp-concepts) (específicamente [RabbitMQ](https://www.rabbitmq.com/) es utilizado como broker) para hacer entrega de las peticiones y resultados de análisis.

#### Peticiones
Las peticiones de análisis se realizan desde el [servicio de análisis](#analysis_service) a través de un _exchange_ nombrado `analysis:requests` de tipo **topic**, este exchange debe de tener una cola para cada una de las enfermedades soportadas y el nombre de estas colas corresponde al **[identificador único](diseases.md) de la enfermedad**, por lo general las colas deberían de ser creadas por el modelo al iniciar y eliminadas al finalizar.

Las peticiones de análisis deben llevar como tópico el _identificador único_ de la enfermedad a diagnosticar, y los modelos de análisis capaces de diagnosticar esa enfermedad deben de suscribirse para recibir y procesar solicitudes.

![MQ Request Diagram](assets/cloud/mq_request.png)

#### Resultados
Una vez establecida una conexión entre el cliente y el servicio de análisis, este último debe de crear una cola cuyo nombre corresponde al **email del especialista** en un _exchange_ de nombre `analysis:response` y de tipo **direct**, esta cola es _transiente_ y _efímera_ por lo cuál sólo tiene un suscriptor y es eliminada una vez este libera la conexión. El servicio de análisis debe de liberar la conexión una vez el cliente es deconectado.

Los resultados de análisis se envían desde alguno de los modelos hacia la cola cuyo nombre es equivalente al **email del especialista**, de esta manera los resultados pueden ser entregados de manera inmediata.

El modelo debe de recibir una confirmación de recepción por parte del servicio de análisis que atiende al cliente destino de los resultados, el servicio de análisis puede marcar la entrega como fallida en caso de que el cliente haya abandonado la conexión. Si el mensaje no pudo ser entregado, el modelo debe de reenviar los resultados a la cola de nombre `global`, esta cola actúa como una _cola de mensajes muertos_ y entregará los resultados a cualquiera de los servicios de análisis disponibles para su almacenamiento (Todos los servicios de análisis deben de estar suscritos a la cola `global`).

![MQ Response Diagram](assets/cloud/mq_response.png)

## Infrastructura (Kubernetes)

Esta es la plantilla base para generar una configuración declarativa de _Kubernetes_ para _LeishmaniappCloudServicesV2_.
![Diagrama de despliegue en Kubernetes](assets/cloud/kubernetes.png)

### Instrucciones de Despliegue

1. Crear los volúmenes requeridos
    1. **Volumen de la Base de Datos:** Contiene la base de datos **mongodb**, que incluye todas las colecciones de _autenticación_, _muestras_ y _diagnósticos_. Este volumen debe ser persistido y protegido, ya que contiene los datos principales de la aplicación.
    2. **Volumen del Repositorio de Muestras:** Contiene todos los archivos de imágenes sin procesar de muestras subidas a través de _leishmaniapp.cloud.analysis.AnalysisService_. Debe ser persistido y tener una gran capacidad de almacenamiento. Este volumen debe tratarse como un **data lake** para el entrenamiento de modelos, pero las copias de seguridad no son tan cruciales.

2. Crear los _namespaces_

   Todos los componentes están aislados dentro de [_namespaces de Kubernetes_](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/). Los namespaces son los siguientes:
    1. **mongodb:** Contiene el [_MongoDB Community Kubernetes Operator_](https://github.com/mongodb/mongodb-kubernetes-operator) y las bases de datos _mongodb_ desplegadas (una primaria y dos secundarias con afinidad de pod) por defecto.
    2. **rabbitmq:** Contiene el [_RabbitMQ Kubernetes Cluster Operator_](https://github.com/rabbitmq/cluster-operator) y la cola de mensajes _rabbitmq_ desplegada.
    3. **leishmaniapp:** Contiene todos los servicios de _Leishmaniapp_ (Gateway, AuthService, SamplesService, DiagnosesService y AnalysisService), sus secretos correspondientes, configuraciones y [_ClusterIP services_](https://kubernetes.io/es/docs/concepts/services-networking/service/).
    4. **models:** Contiene todos los servicios de análisis desplegados compatibles con el MQ.

   Para crear los _namespaces_ requeridos, use el siguiente comando:
   ```shell
   kubectl apply -R -f k8s/00-namespace
   ```

3. Instalar los operadores de Kubernetes **MongoDB y RabbitMQ**

   Estos suelen instalarse usando [**helm**](https://helm.sh/). Este repositorio utiliza la configuración declarativa de **helm** a través de [_helmfile_](https://github.com/roboll/helmfile). Instale la utilidad _helmfile_ siguiendo la documentación y luego utilice los siguientes comandos para desplegar los _charts_ de helm requeridos en el clúster de Kubernetes:
    ```shell
    helmfile init
    helmfile apply
    ```

4. Desplegar y configurar la base de datos **mongodb**

    1. Despliegue _mongodb_ en el clúster usando el siguiente comando:
       ```shell
       kubectl apply -R -f k8s/10-mongodb
       ```
    2. Use las credenciales proporcionadas en el archivo `01-credentials.yaml` para acceder a la base de datos a través del servicio `mongodb-nodeport`. Una vez conectado, cree las bases de datos y colecciones requeridas (authDB/specialists, samplesDB/samples, diagnosesDB/diagnoses) y agregue las restricciones necesarias.
    3. Obtenga la _cadena de conexión_ necesaria para que los microservicios accedan a la base de datos a través de los secretos proporcionados por el operador de _mongodb_. Liste estos secretos con:
       ```shell
       kubectl -n mongodb get secrets
       ```
    4. Use las credenciales para establecer los secretos requeridos en cada uno de los servicios.\
       Ejemplo: Obtenga las credenciales de _AuthService_ con el siguiente comando:
       ```shell
       kubectl -n mongodb get secret mongodb-database-admin-authserviceuser -o json | jq -r ".data"
       ```
       Con las credenciales predeterminadas, la salida del comando debería ser (valores codificados en base64):
```json
  {
    "connectionString.standard": "bW9uZ29kYjovL2F1dGhTZXJ2aWNlVXNlcjphdXRoU2VydmljZVBhc3N3b3JkMUBtb25nb2RiLWRhdGFiYXNlLTAubW9uZ29kYi1kYXRhYmFzZS1zdmMubW9uZ29kYi5zdmMuY2x1c3Rlci5sb2NhbDoyNzAxNyxtb25nb2RiLWRhdGFiYXNlLTEubW9uZ29kYi1kYXRhYmFzZS1zdmMubW9uZ29kYi5zdmMuY2x1c3Rlci5sb2NhbDoyNzAxNy9hZG1pbj9yZXBsaWNhU2V0PW1vbmdvZGItZGF0YWJhc2Umc3NsPWZhbHNl",
    "connectionString.standardSrv": "bW9uZ29kYitzcnY6Ly9hdXRoU2VydmljZVVzZXI6YXV0aFNlcnZpY2VQYXNzd29yZDFAbW9uZ29kYi1kYXRhYmFzZS1zdmMubW9uZ29kYi5zdmMuY2x1c3Rlci5sb2NhbC9hZG1pbj9yZXBsaWNhU2V0PW1vbmdvZGItZGF0YWJhc2Umc3NsPWZhbHNl",
    "password": "YXV0aFNlcnZpY2VQYXNzd29yZDE=",
    "username": "YXV0aFNlcnZpY2VVc2Vy"
  }
```
       Copie el valor de _connectionString.standard_ y péguelo en la clave _database-str_ dentro del archivo `30-services/20-auth-service/10_secret.yaml`. Lo mismo se requiere para los servicios de _Diagnoses_ y _Samples_.

5. Desplegar y configurar el servicio MQ **rabbitmq**
    1. Despliegue _RabbitMQ_ en el clúster usando el siguiente comando:
        ```shell
        kubectl apply -R -f k8s/20-rabbitmq
        ```
    2. Obtenga las credenciales predeterminadas usando el secreto `rabbitmq-cluster-default-user`. Se sugiere el siguiente comando (_kubectl_ y _jq_ requeridos):
       ```shell
       kubectl -n rabbitmq get secret rabbitmq-cluster-default-user -o json | jq -r ".data | with_entries(.value |= @base64d)"
       ```
    3. Use el servicio `management` NodePort para acceder a la _Consola de Gestión de RabbitMQ_ y crear los usuarios *analysis* y *model* con sus contraseñas, y configurar los permisos de lectura+escritura del _vhost_ '/'.

6. Desplegar los servicios de Leishmaniapp
    1. Configure los _secrets_ y las _configuraciones_ correspondientes de acuerdo con las credenciales configuradas para los servicios de **mongodb** y **rabbitmq**.
    2. Despliegue los servicios usando el siguiente comando:
       ```shell
       kubectl apply -R -f k8s/30-services
       ```

7. Desplegar los _modelos de análisis_

    1. Configure el _secret_ correspondiente según las credenciales configuradas para el servicio **rabbitmq**.
    2. Despliegue los servicios usando el siguiente comando:
       ```shell
       kubectl apply -R -f k8s/40-models
       ```