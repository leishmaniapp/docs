# 🚀 LAM

**LAM _(Local Analysis Model)_** es una interfaz utilizada para definir modelos de análisis locales que pueden ser cargados dinámicamente en un entorno compatible con Leishmaniapp.

> 💡 Las herramientas necesarias para implementar modelos _LAM_ en la _JVM_ y en _Android_ se encuentran en el repositorio (https://github.com/leishmaniapp/analysis-jvm)[https://github.com/leishmaniapp/analysis-jvm]

> 🚀 Interfaz LAM para entornos con _JVM_ o _Android_ disponible en [JitPack](https://jitpack.io/#leishmaniapp/analysis-jvm)

Los modelos locales en Leishmaniapp son utilizados para realizar análisis cuando no hay un servicio de análisis disponible o no existe una conexión con el servidor, siempre que haya conexión hay que priorizar el servicio web sobre los modelos locales _LAM_.

Visite [modelos de detección](models.md) para obtener información específica acerca de los modelos LAM disponibles para cada enfermedad

## Android
Los modelos _LAM_ en Android son implementados a través de aplicaciones independientes que exponen [_bound services_](https://developer.android.com/develop/background-work/services/bound-services) que luego son enlazados por la aplicación de _Leishmaniapp_. Este método aprovecha la comunicación entre procesos _IPC_ que el sistema operativo provee al mismo tiempo que mantiene la independencia entre los módulos y la aplicación.

El sistema de plugins _LAM_ mediante _bound services_ permite que una aplicación Android cargue dinámicamente funcionalidades adicionales sin necesidad de reiniciar, reinstalar o actualizar la aplicación de Leishmaniapp, cada modelo _LAM_ está representado por un servicio el cuál interactúa con la aplicación mediante las interfaces expuestas por la liberaría `analysis-jvm` a través de mecanismos de _IPC_, esto aísla el módulo de la aplicación permitiendo así escalabilidad a demanda del usuario y libertad de actualizaciones independientes

### Creación de un LAM
Para crear modelos _LAM_ que puedan ser utilizados por la aplicación de Leishmaniapp en _Android_:

1. Cree una nueva aplicación sin ninguna actividad cuyo nombre de paquete sea `com.leishmaniapp.lam.<disease_id>`
2. Incluya las dependencias de `analysis-jvm` en su proyecto (recuerde configurar los repositorios de JitPack, [visite la documentación](https://jitpack.io/))

    ```groovy
    dependencies {
    	// analysis-jvm Core library
    	implementation("com.github.leishmaniapp.analysis-jvm:core:<version>")
    	// Android libraries for LAM
    	implementation("com.github.leishmaniapp.analysis-jvm:android:<version>")
    }
    ```

3. Cree una clase encargada del manejo de peticiones de análisis, esta clase debe heredar de `com.leishmaniapp.analysis.lam.ILocalAnalysisModel`

    ```java
    package com.leishmaniapp.analysis.lam;

    public interface ILocalAnalysisModel {
        String getModel();
        void load() throws LamException;
        Map<String, List<BoxCoordinates>> analyze(File content) throws LamException;
    }
    ```

    **getModel()** debe de retornar el _identificador_ del modelo, **load()** debe de ser llamado al cargar el modelo _LAM_ y es aquí donde se debe hacer inicialización de cualquier recurso que pueda ser requerido durante el análisis y **analyze()** es el método encargado de realizar el análisis de la muestra.

4. Cree un _bound service_ que maneje las peticiones utilizando las utilidades del paquete `analysis-jvm:android`

    Las peticiones de análisis son del tipo `com.leishmaniapp.analysis.lam.LamAnalysisRequest` y pueden ser recuperadas del _bundle_ del mensaje a través del método `LamAnalysisRequest::fromBundle`, los resultados por otra parte deben de ser entregados mediante la estructura `com.leishmaniapp.analysis.lam.LamAnalysisResponse` y una vez creada esta se puede transformar en un _bundle_ mediante el método `LamAnalysisresponse::toBundle`

5. Exporte el servicio, agrege la acción `com.leishmaniapp.lam.ACTION_ANALYZE` y agrege permiso `com.leishmaniapp.lam.BIND_PERMISSION` (Para que los permisos tengan efecto, ambos Leishmaniapp y el _LAM_ deben de firmarse con la misma firma de desarrollador)

    ```xml
    <service
	    android:name="com.leishmaniapp.lam.foobar.MyBoundService"
	    android:exported="true"
	    android:permission="com.leishmaniapp.lam.BIND_PERMISSION">

	    <intent-filter>
	    	<action android:name="com.leishmaniapp.lam.ACTION_ANALYZE" />
	    </intent-filter>
    </service>
    ```

### Ejemplo

Los siguientes snippets de código fueron obtenidos del modelo _LAM_ para la enfermedad de prueba _mock.spots_ ([Visite el repositorio](https://github.com/leishmaniapp/mock-spots-lam-android)).

##### Bound Service
Este es el servicio al cual la aplicación de Leishmaniapp se enlazará para enviar las peticiones de análisis, recibe un dato de tipo _LamAnalysisRequest_ a través de un _bundle_ cuya key es _**LAM_REQUEST_BUNDLE**_ (puede obtener este dato a través de la extensión `LamAnalysisRequest::fromBundle(Bundle, ClassLoader) -> LamAnalysisRequest?`) y retorna los resultados a través de un _bundle_ con un _AnalysisResultsParcel_ (una especializacion de _AnalysisResults_ que implementa _Parcelable_) y llave _**LAM_RESPONSE_BUNDLE**_ (puede obtener este bundle a través de la extensión `LamAnalysisRequest.toBundle() -> Bundle`)

> 💡 Si su proyecto hace uso de _Jetpack Hilt_, puede utilizar este mismo código para su modelo _LAM_ pues no posee lógica particular a la enfermedad de _mock.spots_

```kotlin
package com.leishmaniapp.lam.mock.spots.service

import android.app.Service
import android.content.Intent
import android.os.Handler
import android.os.IBinder
import android.os.Looper
import android.os.Message
import android.os.Messenger
import android.util.Log
import com.leishmaniapp.analysis.core.AnalysisResultsParcel
import com.leishmaniapp.analysis.core.AnalysisStatus
import com.leishmaniapp.analysis.core.BoxCoordinates
import com.leishmaniapp.analysis.core.BoxCoordinatesParcel
import com.leishmaniapp.analysis.lam.ILocalAnalysisModel
import com.leishmaniapp.analysis.lam.LamAnalysisRequest
import com.leishmaniapp.analysis.lam.LamAnalysisResponse
import com.leishmaniapp.analysis.lam.fromBundle
import com.leishmaniapp.analysis.lam.toBundle
import com.leishmaniapp.lam.mock.spots.BuildConfig
import dagger.hilt.android.AndroidEntryPoint
import java.io.File
import java.io.FileOutputStream
import java.io.IOException
import java.time.LocalDateTime
import java.time.format.DateTimeFormatter
import javax.inject.Inject

/**
 * Analysis service for usage within Leishmaniapp
 *
 */
@AndroidEntryPoint
class IPCAnalysisService : Service() {

    companion object {
        /**
         * TAG containing the class name and the model_id
         */
        val TAG: String = IPCAnalysisService::class.simpleName!!
    }

    /**
     * LAM model implementation (injected)
     */
    @Inject
    lateinit var lam: ILocalAnalysisModel

    /**
     * Messenger between the caller and the service
     */
    private lateinit var messenger: Messenger

    /**
     * Handle incoming messages (analysis requests)
     */
    private val messageHandler = Handler(Looper.getMainLooper()) { msg ->

        // Read the bundle
        Log.i(TAG, "A new processing request arrived")
        val request = LamAnalysisRequest.fromBundle(msg.data, this@IPCAnalysisService.classLoader)
            ?: // Drop request as it is invalid
            return@Handler false

        // Initialize the LAM
        Log.d(TAG, "Request for diagnosis (${request.diagnosis}#${request.sample})")
        lam.load()

        try {

            // Copy file
            val cacheFile = File.createTempFile(
                LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE),
                request.uri.path.let { p -> p!!.substring(p.lastIndexOf(".")) },
                applicationContext.cacheDir,
            )

            // Copy from origin to cacheFile
            applicationContext.contentResolver.openInputStream(request.uri)?.use { ins ->
                FileOutputStream(cacheFile).use { outs ->
                    ins.copyTo(outs)
                }
            }

            // Fetch the analysis
            val results = lam.analyze(cacheFile)

            // Delete the file
            cacheFile.delete()

            // Create the results bundle
            Log.i(TAG, "Results are being constructed")
            val response = LamAnalysisResponse(
                diagnosis = request.diagnosis,
                sample = request.sample,
                results = AnalysisResultsParcel(
                    model = lam.model,
                    status = AnalysisStatus.OK,
                    version = BuildConfig.VERSION_NAME,
                    // Append the results as parcelable
                    results = results.map { (k, v) ->
                        k to v.map { c: BoxCoordinates ->
                            BoxCoordinatesParcel(
                                c.x,
                                c.y,
                                c.w,
                                c.h,
                            )
                        }
                    }.toMap(),
                )
            )

            // Reply
            Log.i(TAG, "Replied to client with analysis results")
            msg.replyTo.send(Message.obtain(msg).apply {
                data = response.toBundle()
            })

        } catch (e: Exception) {

            // No reply
            if (msg.replyTo == null) {
                Log.e(TAG, "Sender does not contain (replyTo)")
                return@Handler false
            }

            //  Send a failed request to the client
            Log.e(TAG, "Error while processing request", e)
            msg.replyTo.send(Message().apply {
                data.putParcelable(
                    LamAnalysisResponse::class.qualifiedName!!, LamAnalysisResponse(
                        diagnosis = request.diagnosis,
                        sample = request.sample,
                        results = AnalysisResultsParcel(
                            model = lam.model,
                            version = BuildConfig.VERSION_NAME,
                            status = when (e) {
                                is IOException -> AnalysisStatus.INVALID_FILE
                                else -> AnalysisStatus.UNPROCESSABLE_CONTENT
                            },
                            results = null,
                        )
                    )
                )
            })
        }


        // Set to true if no more messages are desired
        false
    }

    /**
     * Bind this service with a [Messenger]
     */
    override fun onBind(intent: Intent?): IBinder? {
        Log.i(TAG, "External bind request for action (${intent?.action})")
        messenger = Messenger(messageHandler)

        Log.i(TAG, "Successfully bound (${BuildConfig.APPLICATION_ID}:${BuildConfig.VERSION_NAME})")
        return messenger.binder
    }
}
```

##### Implementación del modelo _LAM_
A continuación un ejemplo de una implementación concreta de _ILocalAnalysisModel_ para ejecutar un modelo implementado en _Python_ a través del SDK de [Chaquopy](https://chaquo.com/chaquopy/)

```kotlin
package com.leishmaniapp.lam.mock.spots.python

import android.content.Context
import android.util.Log
import com.chaquo.python.PyObject
import com.chaquo.python.Python
import com.chaquo.python.android.AndroidPlatform
import com.leishmaniapp.analysis.core.BoxCoordinates
import com.leishmaniapp.analysis.lam.ILocalAnalysisModel
import com.leishmaniapp.analysis.lam.exception.LamException
import com.leishmaniapp.analysis.lam.exception.LamNativeModuleLoadException
import com.leishmaniapp.analysis.lam.exception.LamNotLoadedException
import com.leishmaniapp.lam.mock.spots.R
import dagger.hilt.android.qualifiers.ApplicationContext
import java.io.File
import javax.inject.Inject

/**
 * Python implementation of [ILocalAnalysisModel] using Chaquopy
 */
class PythonLamImpl @Inject constructor(

    /**
     * Application context for grabbing resources and Python initialization
     */
    @ApplicationContext
    private val context: Context,

    ) : ILocalAnalysisModel {

    companion object {

        /**
         * TAG for using with [Log]
         */
        val TAG: String = PythonLamImpl::class.simpleName!!

        /**
         * Python module name derived from the filename
         */
        const val MODULE_NAME: String = "model"

        /**
         * Module function to handle analysis
         */
        const val ENTRY_POINT_ATTR: String = "analyze"

        /**
         * Model tolerance
         */
        const val TOLERANCE: Int = 10
    }

    /**
     * Get the disease ID from resources
     */
    override fun getModel(): String =
        context.getString(R.string.model_id)

    /**
     * model.py module
     */
    private lateinit var pyModule: PyObject

    /**
     * Start the Python environment and load the module
     */
    @Throws(LamException::class)
    override fun load() {

        // Load the python environment
        try {
            if (!Python.isStarted()) {
                Python.start(AndroidPlatform(context))
                Log.i(TAG, "Python interpreter successfully started")
            }
        } catch (e: Exception) {
            // Return native module fail
            Log.e(TAG, "Failed to start Python interpreter", e)
            throw LamNativeModuleLoadException(
                model,
                e
            )
        }

        // Load the python module
        pyModule = try {
            Python.getInstance().getModule(MODULE_NAME)
        } catch (e: Exception) {
            // Return native module fail
            Log.e(TAG, "Failed to load python module ($MODULE_NAME)", e)
            throw LamNativeModuleLoadException(
                model,
                e
            )
        }

        Log.i(TAG, "Python module ($MODULE_NAME) successfully loaded")
    }

    @Throws(LamException::class)
    override fun analyze(content: File): Map<String, List<BoxCoordinates>> {

        // Check if module was initialized
        if (!::pyModule.isInitialized) {
            Log.e(TAG, "Python module not initialized")
            throw LamNotLoadedException(model)
        }

        // Call the python function
        Log.d(
            TAG,
            "Calling Python symbol ($MODULE_NAME::$ENTRY_POINT_ATTR) with (${content.path}) as input data"
        )
        return try {
            // Python input:
            //      filepath: Input filename (str)
            //      tolerance: Model tolerance (int)
            // Python return type:
            // dict[str, list[dict[str, int]]]: A dictionary with elements as key and a list of box coordinates as value
            pyModule.callAttr(ENTRY_POINT_ATTR, content.path, TOLERANCE)
                // Parse return values
                // Cast dict[str, list[dict[str, int]]] into Map<String, List<BoxCoordinates>>
                .asMap().map { (key, value) ->
                    key.toString() to value.asList().map { pyObject ->
                        pyObject.asMap().let {
                            it.map { (k, v) -> k.toString() to v.toInt() }.toMap().let { m ->
                                BoxCoordinates(
                                    m["x"]!!,
                                    m["y"]!!,
                                    m["w"]!!,
                                    m["h"]!!
                                )
                            }
                        }
                    }
                }.toMap()
        } catch (e: Exception) {
            Log.e(TAG, "Failed to callAttr on ($MODULE_NAME::$ENTRY_POINT_ATTR)", e)
            throw LamException(model, "Python callAttr failed", e)
        }
    }
}
```

##### AndroidManifest.xml

* El nombre del paquete es (**com.leishmaniapp.lam.**_mock.spots_), nótese que para que el modelo _LAM_ funcione adecuadamente con la aplicación de Leishmaniapp el nombre del paquete debe de comenzar por el prefijo `com.leishmaniapp.lam.`
* No hay actividades, este modelo es únicamente accesible a través del _bound service_
* El servicio _.service.IPCAnalysisService_ está exportado (`android:exported="true"`), requiere el permiso `com.leishmaniapp.lam.BIND_PERMISSION` y tiene un _intent-filter_ con la acción `com.leishmaniapp.lam.ACTION_ANALYZE`

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.leishmaniapp.lam.mock.spots">

    <application
        android:name=".android.MainApplication"
        android:description="@string/app_description"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        tools:targetApi="31">

        <service
            android:name=".service.IPCAnalysisService"
            android:description="@string/app_description"
            android:exported="true"
            android:permission="com.leishmaniapp.lam.BIND_PERMISSION">
            <intent-filter>
                <action android:name="com.leishmaniapp.lam.ACTION_ANALYZE" />
            </intent-filter>
        </service>

    </application>

</manifest>
```

### LAM Python
Es usual que algunos de estos modelos de análisis estén escritos con Python, a continuación se mostrará cómo actualizar los modelos en Python utilizando como ejemplo a la enfermedad `leishmaniasis.giemsa`. (Visite el repositorio para obtener el código fuente: [github.com/leishmaniapp/leishmaniasis-giemsa-lam-android](https://github.com/leishmaniapp/leishmaniasis-giemsa-lam-android))

La implementación de este modelo está escrita enteramente en _Python_a, esto es posible gracias al _SDK_ de _[Chaquopy](https://chaquo.com/chaquopy/)_ el cual permite la ejecución del intérprete de _Python_ en el teléfono, gracias a esto es posible modificar el modelo sin necesidad de modificar el código navito en _Kotlin_ o cualquiera de las configuraciones de _Python_. Debido a las limitaciones del _SDK_ se recomiendo utilizar la versión de **Python 3.8**

A continuación de explica cómo actualizar los modelos:

#### Actualización de Librerías
Dentro del directorio `app` puede encontrar el archivo `requirements.txt`, este archivo de texto plano contiene los nombres de las librerías y las versiones que serán instaladas mediante el gestor de paquetes _pip_, agrege, elimine o actualice las dependencias de acuerdo a la syntaxis oficial (Visite la [documentación oficial de PiP acerca de los archivos requirements.txt](https://pip.pypa.io/en/stable/reference/requirements-file-format/))

Este es un ejemplo de un archivo `requirements.txt`

```properties
cvzone==1.6.1
numpy==1.19.5
opencv-python==4.5.1.48
scikit-learn==1.1.3
scikit-image==0.18.3
```cvzone==1.6.1
numpy==1.19.5
opencv-python==4.5.1.48
scikit-learn==1.1.3
scikit-image==0.18.3
```

#### Actualización del código fuente de los modelos
El código fuente de los modelos se encuentra en el directorio `app/src/main/python`, encuentre el archivo correspondiente al modelo que desea modificar y copie/pegue el código fuente en _Python_ que se desea ejecutar. Recuerde que los modelos deben de cumplir con el formato _[ALEF](models.md#alef-adapter-layer-exec-format)_

Si cambia el nombre del archivo deberá de especificar el nuevo nombre del módulo en el archivo `app/src/main/kotlin/com/leishmaniapp/lam/leishmaniasis/giemsa/python/PythonLamImpl.kt` en la variable `PARASITES_MODULE_NAME`

```kotlin
package com.leishmaniapp.lam.leishmaniasis.giemsa.python
...
class PythonLamImpl @Inject constructor(...) : ILocalAnalysisModel {
    ...
    companion object {
        ...
        // !!! Modifique esta variable
        // Coloque el nombre del archivo sin la extensión .py
        const val PARASITES_MODULE_NAME: String = "parasites_v2"
        ...
    }
...
}
```

##### Función de análisis (Punto de entrada)
El punto de entrada para el análisis debe de ser una función con un único parámetro de entrada de tipo `str`, este parámetro corresponde a la ruta relativa o absoluta hacia el archivo de imagen que se utilizará para el análisis, los resultados deben de ser del tipo `dict[str, list[dict[str, int]]]` y corresponden a las salidas definidas por _ALEF_.

A continuación un ejemplo sencillo del funcionamiento de la función de análisis

```python
import cv2

def analyze(filepath):
    # type: (str) -> dict[str, list[dict[str, int]]]

    # Leer la imagen
    image = cv2.imread(filepath)

    ...

    # Retorne resultados como los de este ejemplo
    return {
        "parasite": [
            { 'x': 10, 'y': 20, 'w': 5, 'h': 5 },
            { 'x': 15, 'y': 10, 'w': 6, 'h': 3 }
        ]
    }
```

El nombre por defecto de esta función debe de ser `analyze`, sin embargo; es posible cambiar este nombre modificando el valor de la variable `ENTRY_POINT_ATTR` en el archivo `app/src/main/kotlin/com/leishmaniapp/lam/leishmaniasis/giemsa/python/PythonLamImpl.kt`.

```kotlin
package com.leishmaniapp.lam.leishmaniasis.giemsa.python
...
class PythonLamImpl @Inject constructor(...) : ILocalAnalysisModel {
    ...
    companion object {
        ...
        // !!! Modifique esta variable
        // Coloque el nombre de la función
        const val ENTRY_POINT_ATTR: String = "analyze"
        ...
    }
...
}
```

##### Archivos adicionales
Cualquier otro archivo adicional que se requiera durante la ejecución del modelo (ej. archivos de _TensorFlow .tflite_ o archivos _pickle .pkl_) debe se colocarse en el mismo directorio que el código fuente en Python (`app/src/main/python`). Y debe de accederse usando una ruta relativa al archivo mediante la variable mágina `__file__`

Ejemplo: suponga que desea utilizar un archivo `my_file.pkl`, la ruta final del archivo debe de ser `app/src/main/python/my_file.pkl` y puede obtener la ruta hacia este archivo durante la ejecución del modelo con el siguiente código:

```python
import os
import pathlib

my_file_path = os.path.join(
    pathlib.Path(__file__).parent.absolute(),
    "my_file.pkl",
)
```

Ahora puede leer el archivo utilizando la variable `my_file_path` en _Python_. Recuerde que esto no aplica únicamente a los modelos _LAM_ de Leishmaniapp, sino que es considerado _mejores prácticas_ en la programación _Python_ en general.