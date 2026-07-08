# Banco de Definiciones y Preguntas — GambApp (Parte 1)
## Kotlin · Coroutines · Compose · MVVM

---

## BLOQUE 1: Kotlin Fundamentals

### `data class`
Clase cuyo propósito es únicamente contener datos. Kotlin genera automáticamente `equals()`, `hashCode()`, `toString()` y `copy()`.

**En GambApp:**
```kotlin
data class Session(
    val userId: String,
    val exerciseId: String,
    val dateTimestamp: Long,
    val durationSeconds: Long,
    val averageRom: Float,
    val successfulReps: Int,
)
```
`copy()` permite crear una versión modificada sin mutar el original:
```kotlin
val updatedState = _uiState.value.copy(isLoading = false, error = null)
```

### `sealed interface` / `sealed class`
Define una jerarquía cerrada de tipos. El compilador sabe todos los subtipos posibles, lo que permite usar `when` exhaustivo sin `else`.

**En GambApp — `LoginUiState`:**
```kotlin
sealed interface LoginUiState {
    object Idle : LoginUiState
    object Loading : LoginUiState
    object Success : LoginUiState
    data class Error(val message: UiText) : LoginUiState
}

// Uso en UI (when exhaustivo):
when (val state = uiState) {
    is LoginUiState.Idle -> { /* nada */ }
    is LoginUiState.Loading -> CircularProgressIndicator()
    is LoginUiState.Success -> onNavigateToDashboard()
    is LoginUiState.Error -> Snackbar(state.message)
}
```

### `object` vs `data object`
- `object`: singleton. No implementa `equals()`/`toString()` correctamente.
- `data object` (Kotlin 1.9+): singleton con `equals()`, `hashCode()` y `toString()` generados. Recomendado para destinos de navegación y estados sellados sin parámetros.

**En GambApp — `Navigation.kt`:**
```kotlin
// Actual (debería ser data object):
object Dashboard : Screen("dashboard")
// Correcto:
data object Dashboard : Screen("dashboard")
```

### `companion object`
Bloque singleton dentro de una clase. Equivale a los miembros `static` de Java.

**En GambApp — `AccelerometerDataSource`:**
```kotlin
companion object {
    const val FALL_THRESHOLD_MS2: Float = 24.5f // 2.5G
}
// Acceso: AccelerometerDataSource.FALL_THRESHOLD_MS2
```

### Extension functions
Añaden métodos a clases existentes sin heredar. No modifican la clase original.

**En GambApp — mappers:**
```kotlin
fun SessionEntity.toDomain(): Session = Session(
    userId = this.userId,
    exerciseId = this.exerciseId,
    ...
)
// Uso: entity.toDomain()
```

### `runCatching`
Envuelve un bloque en un `Result<T>`. Si lanza excepción → `Result.Failure`. Si ok → `Result.Success`.

**En GambApp — `LoginViewModel`:**
```kotlin
_uiState.value = runCatching {
    loginUseCase(email, password)
    LoginUiState.Success
}.getOrElse { e ->
    LoginUiState.Error(UiText.DynamicString(e.message ?: "Error"))
}
```
Ventaja: evita try/catch anidado, es más expresivo y funcional.

### `?.`, `?:`, `!!`
- `?.` (safe call): llama el método solo si el objeto no es null.
- `?:` (Elvis): valor por defecto si es null.
- `!!` (non-null assertion): fuerza no-null, lanza `NullPointerException` si es null. Evitar en producción.

```kotlin
val userId = user?.id ?: "user_imanol"   // en DashboardViewModel
val phone = clinic?.phone?.takeIf { it.isNotEmpty() } ?: return  // en MapScreenViewModel
```

### `inline fun` y `reified`
- `inline`: el compilador copia el cuerpo de la función en el call site (evita overhead de lambdas).
- `reified`: permite acceder al tipo genérico en tiempo de ejecución dentro de funciones `inline`.

**Uso en Compose:**
```kotlin
inline fun <reified T : ViewModel> hiltViewModel(): T
```
Sin `reified`, no se puede hacer `T::class.java` en genéricos.

---

## BLOQUE 2: Kotlin Coroutines

### ¿Qué es una Coroutine?
Unidad de computación suspendible. No bloquea el hilo — cuando suspende, libera el hilo para que haga otro trabajo. Cuando la operación termina, se reanuda (posiblemente en otro hilo).

```kotlin
viewModelScope.launch {        // lanza coroutine en Main thread
    val data = repository.getData()  // suspend fun → no bloquea UI thread
    _uiState.value = data      // vuelve al Main thread automáticamente
}
```

### `suspend fun`
Función que puede suspenderse (pausarse) sin bloquear el hilo. Solo puede llamarse desde otra función `suspend` o desde una coroutine.

**En GambApp:**
```kotlin
override suspend fun saveSession(session: Session) {   // RehabRepository
    sessionDao.insertSession(session.toEntity())       // Room auto-maneja Dispatchers.IO
}
```

### `Dispatchers`
Determina en qué hilo/pool de hilos corre una coroutine:
- `Dispatchers.Main` — UI thread (Android Main Thread). Solo operaciones de UI.
- `Dispatchers.IO` — Pool para I/O (BD, red, archivos). Hasta 64 hilos.
- `Dispatchers.Default` — Cómputo CPU intensivo (sorting, parsing).

**En GambApp:** Room y Retrofit manejan internamente `Dispatchers.IO`. Los ViewModels operan en `viewModelScope` que usa `Dispatchers.Main.immediate`.

### `viewModelScope`
`CoroutineScope` ligado al ciclo de vida del ViewModel. Se cancela automáticamente cuando el ViewModel se destruye (cuando el usuario sale de la pantalla y el back stack entry desaparece). Evita memory leaks.

```kotlin
// DashboardViewModel:
viewModelScope.launch {
    // Si el usuario cierra la pantalla, esta coroutine se cancela automáticamente
    rehabRepository.getSessions(userId).collect { ... }
}
```

### `Flow<T>`
Stream reactivo de valores que emite 0 a N elementos y luego completa o falla. Es "cold" — no emite hasta que alguien colecta. Se controla con el ciclo de vida del collector.

```kotlin
// RehabRepositoryImpl — emite lista actualizada cada vez que Room cambia:
override fun getSessions(userId: String): Flow<List<Session>> =
    sessionDao.getSessionsByUser(userId)
        .map { entities -> entities.map { it.toDomain() } }
```

### `StateFlow<T>`
`Flow` especial que:
1. Siempre tiene un valor (no puede estar vacío).
2. Emite el último valor inmediatamente al nuevo collector (replay = 1).
3. No emite si el valor nuevo es igual al anterior (`distinctUntilChanged` implícito).

**En GambApp:**
```kotlin
private val _uiState = MutableStateFlow(DashboardUiState())   // mutable (interno)
val uiState: StateFlow<DashboardUiState> = _uiState.asStateFlow()  // read-only (expuesto)
```

### `stateIn()`
Convierte un `Flow` frío en un `StateFlow` caliente con ciclo de vida controlado.

**En GambApp — `SplashViewModel`:**
```kotlin
val uiState: StateFlow<SplashUiState> =
    sessionPreferences.isOnboardingCompleted
        .map { completed -> SplashUiState.Ready(onboardingCompleted = completed) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),  // activo 5s tras desuscripción
            initialValue = SplashUiState.Loading,
        )
```
`WhileSubscribed(5000)`: mantiene el Flow activo 5 segundos después de que el último subscriber cancele. Cubre rotaciones de pantalla (~500ms) y navegación rápida.

### `combine()`
Operador que combina los últimos valores de múltiples Flows. Re-emite cuando **cualquiera** de los upstream emite.

**En GambApp — `DashboardViewModel`:**
```kotlin
combine(
    rehabRepository.getSessions(userId),    // Flow 1: DB Room
    stepCounterDataSource.getStepsFlow(),   // Flow 2: SharedPreferences
    achievementRepository.getAchievements() // Flow 3: DB Room
) { sessions, steps, achievements ->
    DashboardUiState(
        currentSteps = steps,
        maxRom = sessions.maxOfOrNull { it.averageRom } ?: 0f,
        unlockedAchievementsCount = achievements.count { it.isUnlocked }
    )
}.collect { state -> _uiState.value = state }
```

### `callbackFlow`
Builder para crear un `Flow` que publica valores desde callbacks externos (listeners). `awaitClose` es la función de limpieza que se ejecuta cuando el Flow se cancela.

**En GambApp — `LightSensorDataSource`:**
```kotlin
fun getLuxFlow(): Flow<Float> = callbackFlow {
    val listener = object : SensorEventListener {
        override fun onSensorChanged(event: SensorEvent?) {
            event?.values?.get(0)?.let { trySend(it) }  // emite lux value
        }
        override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
    }
    sensorManager.registerListener(listener, lightSensor, SENSOR_DELAY_UI)
    awaitClose { sensorManager.unregisterListener(listener) }  // limpieza al cancelar
}
```

### `collectLatest`
Colecta un Flow, pero si llega un nuevo valor antes de que el bloque anterior termine, **cancela el anterior y empieza con el nuevo**. Ideal para búsquedas o pantallas que reaccionan al último estado.

**En GambApp — `PostSessionViewModel`:**
```kotlin
rehabRepository.getSessions(userId).collectLatest { sessions ->
    val session = sessions.firstOrNull()
    _lastSession.value = session
    // Si sessions cambia antes de terminar, cancela y reinicia
}
```

### `flatMapLatest`
Transforma cada valor emitido en un nuevo Flow, y cancela el Flow interno anterior cuando llega un nuevo upstream. La alternativa declarativa a `collectLatest` anidado.

```kotlin
// Refactorización ideal de PostSessionViewModel:
val exercise: StateFlow<Exercise?> = lastSession
    .flatMapLatest { session ->
        session?.let { rehabRepository.getExerciseById(it.exerciseId) } ?: flowOf(null)
    }.stateIn(viewModelScope, SharingStarted.Lazily, null)
```

### `SupervisorJob`
Job donde el fallo de un hijo **no** cancela a los demás hijos ni al padre. Contrasta con `Job` regular donde un fallo se propaga.

**En GambApp — `AppModule`:**
```kotlin
@Provides @Singleton @ApplicationScope
fun providesApplicationScope(): CoroutineScope = CoroutineScope(SupervisorJob())
```
Se usa para el scope de la aplicación porque un error en una tarea background no debe tumbar toda la app.

---

## BLOQUE 3: Jetpack Compose

### Composable function
Función anotada con `@Composable`. Describe una parte de la UI de forma declarativa. Compose la re-ejecuta (recompone) cuando el estado observado cambia.

```kotlin
@Composable
fun SkeletonOverlay(poseResult: PoseResult?, precision: JointPrecision) {
    if (poseResult == null) return
    val color by animateColorAsState(targetValue = precisionColor(precision))
    Canvas(modifier = Modifier.fillMaxSize()) { /* dibuja esqueleto */ }
}
```

### Recomposición (Recomposition)
Proceso donde Compose re-ejecuta las funciones `@Composable` cuyo estado cambió para actualizar la UI. Solo recompone las partes que realmente cambiaron, no toda la pantalla.

**Importante:** Las funciones `@Composable` deben ser puras (sin efectos secundarios) para que la recomposición sea predecible.

### `remember`
Almacena un valor a través de recomposiciones. El valor se recalcula solo si las `keys` cambian.

```kotlin
val snackBarHostState = remember { SnackbarHostState() }
// Sin remember: se crearía un nuevo SnackbarHostState en cada recomposición
```

### `LaunchedEffect(key)`
Lanza una coroutine cuando el Composable entra en composición. Se re-ejecuta si `key` cambia. Se cancela cuando el Composable sale de composición.

**En GambApp — `SplashScreen`:**
```kotlin
LaunchedEffect(uiState) {                          // key = uiState
    when (val state = uiState) {
        is SplashUiState.Ready -> {
            if (state.onboardingCompleted) onNavigateToLogin()
            else onNavigateToOnboarding()
        }
        else -> Unit
    }
}
```
Cuando `uiState` cambia de `Loading` a `Ready`, el effect se re-ejecuta y navega.

### `collectAsStateWithLifecycle()`
Colecta un `Flow`/`StateFlow` y lo convierte en `State<T>` de Compose, respetando el ciclo de vida de la pantalla. **Pausa la colección cuando la app va a background** (a diferencia de `collectAsState()` que sigue colectando).

```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

### `hiltViewModel()`
Función de Compose que obtiene un `ViewModel` inyectado con Hilt. Está ligado al `BackStackEntry` actual de Navigation, por lo que cada pantalla tiene su propio ViewModel con su propio scope.

### `Modifier`
Cadena de instrucciones que modifica cómo se dibuja y se comporta un Composable. Son inmutables y se encadenan.

```kotlin
Modifier
    .fillMaxSize()
    .background(MaterialTheme.colorScheme.background)
    .padding(24.dp)
    .clickable { onNavigate() }
```

### `MaterialTheme.colorScheme`
Acceso al esquema de colores del tema actual (light/dark). Reemplaza el uso de colores hardcodeados.

```kotlin
// ❌ Mal — no soporta dark mode:
color = ElectricIndigo

// ✅ Bien:
color = MaterialTheme.colorScheme.primary
```

### `animateColorAsState`
Produce un `State<Color>` que transiciona suavemente entre valores cuando el target cambia. Sin animación manual de frames.

**En GambApp — `SkeletonOverlay`:**
```kotlin
val colorFeedback by animateColorAsState(
    targetValue = when(precision) {
        JointPrecision.IDEAL -> EmeraldIdeal
        JointPrecision.WARNING -> AmberWarning
        JointPrecision.ERROR -> CoralDanger
    },
    label = "colorFeedback"
)
```

---

## BLOQUE 4: MVVM y Clean Architecture

### MVVM — Model View ViewModel
Patrón de presentación donde:
- **Model**: lógica de negocio y datos (Use Cases, Repositories, DataSources).
- **View**: UI (Composables). Solo observa estado y emite eventos.
- **ViewModel**: puente. Expone estado inmutable (`StateFlow<UiState>`). Procesa eventos de la View y coordina el Model.

**Flujo en GambApp:**
```
LoginScreen (View)
    → viewModel.onLogin()          (evento)
LoginViewModel (ViewModel)
    → loginUseCase(email, pass)    (usa el Model)
    → _uiState.value = Success     (actualiza estado)
LoginScreen
    → observa uiState → navega     (reacciona al estado)
```

### UiState
Clase inmutable (generalmente `data class` o `sealed interface`) que representa todo el estado visible de una pantalla en un momento dado. El ViewModel es la única fuente de verdad.

**En GambApp:**
```kotlin
data class DashboardUiState(
    val userName: String = "Imanol",
    val currentSteps: Int = 0,
    val maxRom: Float = 0f,
    val isLoading: Boolean = true,
    val error: String? = null,
)
```

### Repository Pattern
Abstrae el origen de los datos. El ViewModel no sabe si los datos vienen de Room, una API o memoria. Implementa una interfaz definida en la capa de dominio.

**En GambApp:**
```kotlin
// Interface en domain/repository:
interface RehabRepository {
    fun getSessions(userId: String): Flow<List<Session>>
    suspend fun saveSession(session: Session)
    fun getExerciseById(id: String): Flow<Exercise?>
}

// Implementación en data/repositories:
class RehabRepositoryImpl(
    private val sessionDao: SessionDao,
    private val exerciseDao: ExerciseDao,
) : RehabRepository {
    override fun getSessions(userId: String) =
        sessionDao.getSessionsByUser(userId).map { it.map { e -> e.toDomain() } }
}
```

### Use Case (Caso de Uso)
Encapsula una operación de negocio específica. Coordina repositorios y aplica reglas de negocio. Hace al sistema testeable sin la UI.

**En GambApp — `CalculateJointAngleUseCase`:**
```kotlin
class CalculateJointAngleUseCase @Inject constructor() {
    fun execute(firstPointX: Float?, firstPointY: Float?, ...): Float {
        val radians = atan2(lastY - midY, lastX - midX) - atan2(firstY - midY, firstX - midX)
        var angle = abs(radians * 180.0 / Math.PI).toFloat()
        if (angle > 180f) angle = 360f - angle
        return angle
    }
}
```

### Mapper (toDomain / toEntity)
Funciones de conversión entre capas. Evitan que los modelos de una capa contaminen a otra:
- `Entity` (capa Data) ↔ `Domain Model` (capa Domain)
- `DTO` (respuesta de API) ↔ `Domain Model`

**En GambApp:**
```kotlin
fun SessionEntity.toDomain(): Session = Session(
    userId = userId, exerciseId = exerciseId, dateTimestamp = dateTimestamp, ...
)
fun Session.toEntity(): SessionEntity = SessionEntity(
    userId = userId, exerciseId = exerciseId, dateTimestamp = dateTimestamp, ...
)
```

---

## BLOQUE 5: Hilt (Dependency Injection)

### ¿Qué es la Inyección de Dependencias?
Patrón donde los objetos reciben sus dependencias desde el exterior en lugar de crearlas internamente. Facilita el testing (se pueden inyectar mocks) y reduce el acoplamiento.

```kotlin
// Sin DI — difícil de testear:
class LoginViewModel() {
    private val loginUseCase = LoginUseCase(FirebaseAuth.getInstance(), ...)
}

// Con DI — fácil de testear con mocks:
class LoginViewModel @Inject constructor(
    private val loginUseCase: LoginUseCase,
) : ViewModel()
```

### `@Module` y `@Provides`
- `@Module`: clase que indica a Hilt cómo crear dependencias que no pueden ser creadas directamente (librerías de terceros, interfaces, etc.).
- `@Provides`: función que retorna la instancia a proveer.

**En GambApp — `AppModule`:**
```kotlin
@Provides @Singleton
fun providesRetrofit(): Retrofit =
    Retrofit.Builder()
        .baseUrl(Constants.GRAPH_HOPPER_BASE_URL)
        .addConverterFactory(GsonConverterFactory.create())
        .build()
```

### `@Singleton`
La misma instancia es reutilizada durante toda la vida de la aplicación. Si no se anotara, Hilt crearía una nueva instancia cada vez que se solicite.

### `@HiltViewModel`
Permite a Hilt inyectar dependencias en un `ViewModel`. La Factory generada automáticamente por Hilt pasa las dependencias al constructor anotado con `@Inject`.

### `@ApplicationContext` vs `Context`
- `@ApplicationContext`: Hilt inyecta el `Context` de la `Application` (no de la Activity). Tiene mayor duración y no genera memory leaks en Singletons.

```kotlin
class MapScreenViewModel @Inject constructor(
    @ApplicationContext private val context: Context,  // seguro en Singleton
    ...
)
```

---

## BANCO DE PREGUNTAS — PARTE 1

### Kotlin

**P: ¿Cuál es la diferencia entre `val` y `var`?**
R: `val` es inmutable (read-only, como `final` en Java). `var` es mutable. En GambApp, todos los `UiState` se exponen como `val` (StateFlow read-only) aunque internamente el `MutableStateFlow` es `private val _uiState`.

**P: ¿Por qué usamos `copy()` en vez de mutar directamente el UiState?**
R: El `UiState` es una `data class` inmutable. `copy()` crea una nueva instancia con los campos modificados, preservando el resto. Esto garantiza que `StateFlow` detecte el cambio (`equals()` dará `false`) y notifique a los collectors. Si mutáramos el objeto directamente, el `StateFlow` no emitiría nada porque la referencia sería la misma.

**P: ¿Qué pasa si usamos `!!` en producción y el valor es null?**
R: Lanza `NullPointerException` y crashea la app. En GambApp se evita con safe calls (`?.`) y el operador Elvis (`?:`). Si usamos `!!` y el valor es null, el usuario ve un crash. Preferimos manejar el null explícitamente.

**P: ¿Qué es un `sealed interface` y por qué es mejor que un `enum` para los estados de UI?**
R: `sealed interface` permite que cada estado tenga distintos campos (ej: `Error` tiene `message`, `Success` no tiene nada). Un `enum` no puede tener campos distintos por variante. Además, el compilador verifica exhaustividad en `when`.

**P: ¿Qué hace `takeIf`?**
R: Retorna el receptor si el predicado es `true`, o `null` si es `false`.
```kotlin
val phone = clinic?.phone?.takeIf { it.isNotEmpty() } ?: return
// Si phone es "" → takeIf retorna null → ?: return → sale de la función
```

### Coroutines

**P: ¿Por qué `callbackFlow` y no simplemente un `Channel`?**
R: `callbackFlow` maneja automáticamente el ciclo de vida del canal, incluye `awaitClose` para la limpieza, y tiene backpressure incorporado. Un `Channel` manual requiere más boilerplate para garantizar la cancelación correcta.

**P: ¿Qué pasa si no llamamos `imageProxy.close()` en CameraX?**
R: CameraX deja de enviar nuevos frames. El `ImageAnalysis` tiene un buffer de 1 frame (`STRATEGY_KEEP_ONLY_LATEST`), y si el frame actual no se cierra, el buffer queda bloqueado. La cámara se "congela" en la última imagen.

**P: ¿Cuál es la diferencia entre `collect` y `collectLatest`?**
R: `collect` procesa cada elemento hasta terminar antes de recibir el siguiente. `collectLatest` cancela el procesamiento anterior si llega un nuevo elemento. Se usa `collectLatest` cuando solo importa el valor más reciente (búsquedas en tiempo real, estado de UI).

**P: ¿Por qué `combine` y no `zip` para el Dashboard?**
R: `zip` espera que **todos** los Flows emitan antes de combinar (empareja 1:1). `combine` usa el **último** valor de cada Flow y re-emite cuando cualquiera cambia. Para un dashboard que actualiza pasos cada segundo, `zip` bloquearía si la BD no emite al mismo ritmo que el sensor.

**P: ¿Qué es `Dispatchers.IO` y cuándo se usa?**
R: Pool de hilos optimizado para operaciones de entrada/salida (disco, red). Room y Retrofit lo usan internamente con sus `suspend fun`. Si hacemos I/O manual (leer un archivo), usamos `withContext(Dispatchers.IO) { ... }` para no bloquear el Main thread.
