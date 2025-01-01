
Android Developers


Acceder
Desarrollo

Android Developers

Se usó la API de Cloud Translation para traducir esta página.
Switch to English
Android Developers
Desarrollo
Guías
¿Te resultó útil?

Cómo usar las corrutinas de Kotlin con componentes optimizados para ciclos de vida 

bookmark_border


Las corrutinas de Kotlin proporcionan una API que te permite escribir código asíncrono. Con ellas, puedes definir un CoroutineScope, lo que te ayuda a administrar cuándo deben ejecutarse las corrutinas. Cada operación asíncrona se ejecuta dentro de un alcance particular.

Los componentes optimizados para ciclos de vida proporcionan compatibilidad de primer nivel con las corrutinas para alcances lógicos de tu app, junto con una capa de interoperabilidad con LiveData. En este tema, se explica cómo usar corrutinas de manera eficaz con componentes optimizados para ciclos de vida.

Agrega dependencias de KTX
Los alcances integrados de las corrutinas que se describen en este tema se encuentran en las extensiones de KTX de cada componente correspondiente. Asegúrate de agregar las dependencias apropiadas cuando uses estos alcances.

Para ViewModelScope, usa androidx.lifecycle:lifecycle-viewmodel-ktx:2.4.0 o una versión posterior.
Para LifecycleScope, usa androidx.lifecycle:lifecycle-runtime-ktx:2.4.0 o una versión posterior.
Para liveData, usa androidx.lifecycle:lifecycle-livedata-ktx:2.4.0 o una versión posterior.
Ámbitos de corrutinas optimizados para ciclos de vida
Los componentes optimizados para ciclos de vida definen los siguientes alcances integrados que puedes usar en tu app.

ViewModelScope
Se define un ViewModelScope para cada objeto ViewModel de tu app. Si se borra ViewModel, se cancela automáticamente cualquier corrutina iniciada en este alcance. Las corrutinas son útiles cuando tienes trabajos que se deben hacer solo si ViewModel está activo. Por ejemplo, si estás procesando datos para un diseño, debes definir el alcance del trabajo en el ViewModel, de modo que, si se borra el ViewModel, se cancele automáticamente el trabajo para no consumir recursos.

Puedes acceder al CoroutineScope de un ViewModel mediante la propiedad viewModelScope del ViewModel, como se muestra en el siguiente ejemplo:


class MyViewModel: ViewModel() {
    init {
        viewModelScope.launch {
            // Coroutine that will be canceled when the ViewModel is cleared.
        }
    }
}
LifecycleScope
Se define un LifecycleScope para cada objeto Lifecycle. Se cancelan todas las corrutinas iniciadas en este alcance cuando se destruye el Lifecycle. Puedes acceder al CoroutineScope del Lifecycle mediante las propiedades lifecycle.coroutineScope o lifecycleOwner.lifecycleScope.

En el siguiente ejemplo, se muestra cómo usar lifecycleOwner.lifecycleScope para crear texto procesado previamente de forma asíncrona:


class MyFragment: Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        viewLifecycleOwner.lifecycleScope.launch {
            val params = TextViewCompat.getTextMetricsParams(textView)
            val precomputedText = withContext(Dispatchers.Default) {
                PrecomputedTextCompat.create(longTextContent, params)
            }
            TextViewCompat.setPrecomputedText(textView, precomputedText)
        }
    }
}
Corrutinas optimizadas para ciclos de vida reiniciables
Aunque lifecycleScope proporciona una forma adecuada de cancelar automáticamente operaciones de larga duración cuando Lifecycle es DESTROYED, es posible que haya otros casos en los que quieras iniciar la ejecución de un bloque de código cuando Lifecycle esté en un estado determinado y cancelarla cuando esté en otro estado. Por ejemplo, es posible que quieras recopilar un flujo cuando Lifecycle sea STARTED y cancelar la recopilación cuando sea STOPPED. Este enfoque procesa las emisiones de flujo solo cuando la IU es visible en la pantalla, lo que ahorra recursos y puede evitar posibles fallas en la app.

En estos casos, Lifecycle y LifecycleOwner proporcionan la API de suspensión repeatOnLifecycle que hace exactamente eso. El siguiente ejemplo incluye un bloque de código que se ejecuta cada vez que el Lifecycle asociado está al menos en el estado STARTED y se cancela cuando el estado de Lifecycle es STOPPED:


class MyFragment : Fragment() {

    val viewModel: MyViewModel by viewModel()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // Create a new coroutine in the lifecycleScope
        viewLifecycleOwner.lifecycleScope.launch {
            // repeatOnLifecycle launches the block in a new coroutine every time the
            // lifecycle is in the STARTED state (or above) and cancels it when it's STOPPED.
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                // Trigger the flow and start listening for values.
                // This happens when lifecycle is STARTED and stops
                // collecting when the lifecycle is STOPPED
                viewModel.someDataFlow.collect {
                    // Process item
                }
            }
        }
    }
}
Recopilación de flujos optimizados para ciclos de vida
Si solo necesitas realizar una recopilación optimizada para ciclos de vida en un solo flujo, puedes usar el método Flow.flowWithLifecycle() para simplificar tu código:


viewLifecycleOwner.lifecycleScope.launch {
    exampleProvider.exampleFlow()
        .flowWithLifecycle(viewLifecycleOwner.lifecycle, Lifecycle.State.STARTED)
        .collect {
            // Process the value.
        }
}
Sin embargo, si necesitas realizar una recopilación optimizada para ciclos de vida en varios flujos en paralelo, debes recopilar cada flujo en diferentes corrutinas. En ese caso, es más eficiente usar repeatOnLifecycle():


viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        // Because collect is a suspend function, if you want to
        // collect multiple flows in parallel, you need to do so in
        // different coroutines.
        launch {
            flow1.collect { /* Process the value. */ }
        }

        launch {
            flow2.collect { /* Process the value. */ }
        }
    }
}
Cómo suspender corrutinas optimizadas para ciclos de vida
Aunque CoroutineScope proporciona una forma adecuada de cancelar automáticamente operaciones de larga duración, es posible que haya otros casos en los que quieras suspender la ejecución de un bloque de código, a menos que el Lifecycle esté en un estado determinado. Por ejemplo, para ejecutar un FragmentTransaction, debes esperar hasta que el Lifecycle esté al menos en el estado STARTED. En estos casos, Lifecycle proporciona métodos adicionales: lifecycle.whenCreated, lifecycle.whenStarted y lifecycle.whenResumed. Se suspenderá cualquier ejecución de corrutina dentro de estos bloques si el Lifecycle no está al menos en el estado mínimo deseado.

El siguiente ejemplo incluye un bloque de código que se ejecuta solamente cuando el Lifecycle asociado está al menos en el estado STARTED:


class MyFragment: Fragment {
    init { // Notice that we can safely launch in the constructor of the Fragment.
        lifecycleScope.launch {
            whenStarted {
                // The block inside will run only when Lifecycle is at least STARTED.
                // It will start executing when fragment is started and
                // can call other suspend methods.
                loadingView.visibility = View.VISIBLE
                val canAccess = withContext(Dispatchers.IO) {
                    checkUserAccess()
                }

                // When checkUserAccess returns, the next line is automatically
                // suspended if the Lifecycle is not *at least* STARTED.
                // We could safely run fragment transactions because we know the
                // code won't run unless the lifecycle is at least STARTED.
                loadingView.visibility = View.GONE
                if (canAccess == false) {
                    findNavController().popBackStack()
                } else {
                    showContent()
                }
            }

            // This line runs only after the whenStarted block above has completed.

        }
    }
}
Si el Lifecycle se destruye mientras una corrutina está activa mediante uno de los métodos when, se cancelará automáticamente la corrutina. En el siguiente ejemplo, el bloque finally se ejecuta una vez que el estado de Lifecycle es DESTROYED:


class MyFragment: Fragment {
    init {
        lifecycleScope.launchWhenStarted {
            try {
                // Call some suspend functions.
            } finally {
                // This line might execute after Lifecycle is DESTROYED.
                if (lifecycle.state >= STARTED) {
                    // Here, since we've checked, it is safe to run any
                    // Fragment transactions.
                }
            }
        }
    }
}
Nota: Aunque estos métodos son convenientes cuando se trabaja con Lifecycle, debes usarlas solo cuando la información sea válida en permiso de Lifecycle (por ejemplo, texto procesado previamente). Ten en cuenta que cuando se reinicia la actividad, no se reinicia la corrutina.
Advertencia: Es preferible recopilar flujos con la API de repeatOnLifecycle en lugar de recopilando dentro de las APIs de launchWhenX. Dado que estas últimas APIs suspenden el corrutina en lugar de cancelarla cuando Lifecycle es STOPPED, los flujos upstream se mantienen activos en segundo plano, lo que podría emitir nuevos elementos y desperdiciar recursos.
Usa corrutinas con LiveData
Cuando usas LiveData, es posible que debas calcular valores de forma asíncrona. Por ejemplo, te recomendamos que recuperes las preferencias de un usuario y las entregues a tu IU. En estos casos, puedes usar la función del compilador de liveData para llamar a una función suspend, que muestra el resultado como un objeto LiveData.

En el siguiente ejemplo, loadUser() es una función de suspensión declarada en otro lugar. Usa la función del compilador de liveData para llamar a loadUser() de forma asíncrona y, luego, usa emit() para emitir el resultado:


val user: LiveData<User> = liveData {
    val data = database.loadUser() // loadUser is a suspend function.
    emit(data)
}
El bloque de compilación liveData funciona como un tipo primitivo de simultaneidad estructurada entre las corrutinas y LiveData. El bloque de código comienza a ejecutarse cuando se activa LiveData, y se cancela automáticamente después de un tiempo de espera configurable cuando LiveData se vuelve inactivo. Si se cancela antes de completarse, se reiniciará cuando se vuelve a activar LiveData. Si se completó correctamente en una ejecución anterior, no se reiniciará. Ten en cuenta que solo se reiniciará si se cancela automáticamente. Si se cancela el bloque por cualquier otro motivo (por ejemplo, si se muestra una CancellationException), no se reiniciará.

También puedes emitir varios valores desde el bloque. Cada llamada a emit() suspende la ejecución del bloque hasta que se establezca el valor LiveData en el subproceso principal.


val user: LiveData<Result> = liveData {
    emit(Result.loading())
    try {
        emit(Result.success(fetchUser()))
    } catch(ioException: Exception) {
        emit(Result.error(ioException))
    }
}
También puedes combinar liveData con Transformations, como se muestra en el siguiente ejemplo:


class MyViewModel: ViewModel() {
    private val userId: LiveData<String> = MutableLiveData()
    val user = userId.switchMap { id ->
        liveData(context = viewModelScope.coroutineContext + Dispatchers.IO) {
            emit(database.loadUserById(id))
        }
    }
}
Puedes emitir varios valores desde un LiveData llamando a la función emitSource() cuando quieras emitir un nuevo valor. Ten en cuenta que cada llamada a emit() o emitSource() quita la fuente que se había agregado previamente.


class UserDao: Dao {
    @Query("SELECT * FROM User WHERE id = :id")
    fun getUser(id: String): LiveData<User>
}

class MyRepository {
    fun getUser(id: String) = liveData<User> {
        val disposable = emitSource(
            userDao.getUser(id).map {
                Result.loading(it)
            }
        )
        try {
            val user = webservice.fetchUser(id)
            // Stop the previous emission to avoid dispatching the updated user
            // as `loading`.
            disposable.dispose()
            // Update the database.
            userDao.insert(user)
            // Re-establish the emission with success type.
            emitSource(
                userDao.getUser(id).map {
                    Result.success(it)
                }
            )
        } catch(exception: IOException) {
            // Any call to `emit` disposes the previous one automatically so we don't
            // need to dispose it here as we didn't get an updated value.
            emitSource(
                userDao.getUser(id).map {
                    Result.error(exception, it)
                }
            )
        }
    }
}
Para obtener más información relacionada con las corrutinas, consulta los siguientes vínculos:

Cómo mejorar el rendimiento de la app con las corrutinas de Kotlin
Descripción general de las corrutinas
Cómo ejecutar subprocesos en CoroutineWorker
Recursos adicionales
Para obtener más información sobre el uso de corrutinas con componentes optimizados para ciclos de vida, consulta los siguientes recursos adicionales.

Ejemplos

GitHub

Architecture
These samples showcase different architectural approaches to developing Android apps. In its different branches you'll find the same app (a TODO app) implemented with small differences. In this branch you'll find: User Interface built with Jetpack


GitHub

Jetcaster sample 🎙️
Jetcaster is a sample podcast app, built with Jetpack Compose. The goal of the sample is to showcase building with Compose across multiple form factors (mobile, TV, and Wear) and full featured architecture. To try out this sample app, use the latest


GitHub

Now in Android App
Learn how this app was designed and built in the design case study, architecture learning journey and modularization learning journey. This is the repository for the Now in Android app. It is a work in progress 🚧. Now in Android is a fully functional


Blogs
Corrutinas en Android: Patrones de aplicación
Corrutinas fáciles en Android: viewModelScope
Prueba de dos emisiones consecutivas de LiveData en corrutinas
Recomendaciones para ti

Descripción general de LiveData
Usa LiveData y administra datos de manera optimizada para ciclos de vida.

Última actualización: 21 dic 2024
Cómo manejar ciclos de vida con componentes optimizados para ciclos de vida
Usa las nuevas clases Lifecycle para administrar los ciclos de vida de actividades y fragmentos.

Última actualización: 21 dic 2024
Cómo cargar y mostrar datos paginados
Discover the latest app development tools, platform updates, training, and documentation for developers across every Android
