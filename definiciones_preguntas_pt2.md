# Banco de Definiciones y Preguntas — GambApp (Parte 2)
## Room · Sensores · CameraX · ML Kit · Navigation · Retrofit · Testing

---

## BLOQUE 6: Room Database

### ¿Qué es Room?
Capa de abstracción sobre SQLite que genera código SQL en tiempo de compilación, verificando queries con type-safety. Si un query tiene error de sintaxis, **el proyecto no compila**.

### `@Entity`
Anota una `data class` como tabla de Room. Cada propiedad es una columna.

**En GambApp:**
```kotlin
@Entity(tableName = "sessions")
data class SessionEntity(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val userId: String,
    val exerciseId: String,
    val dateTimestamp: Long,
    val durationSeconds: Long,
    val averageRom: Float,
    val successfulReps: Int,
)
```

### `@Dao` (Data Access Object)
Interface donde se definen las operaciones sobre la base de datos. Room genera la implementación concreta en tiempo de compilación.

```kotlin
@Dao
interface SessionDao {
    @Query("SELECT * FROM sessions WHERE userId = :userId ORDER BY dateTimestamp DESC")
    fun getSessionsByUser(userId: String): Flow<List<SessionEntity>>  // reactivo

    @Insert(onConflict = OnConflictStrategy.ABORT)
    suspend fun insertSession(session: SessionEntity)  // suspend = corre en IO

    @Query("DELETE FROM sessions WHERE userId = :userId")
    suspend fun deleteSessionsByUser(userId: String)
}
```

### `OnConflictStrategy`
Define qué hacer si se intenta insertar una fila con clave primaria duplicada:
- `REPLACE`: reemplaza la fila existente. Usado en `UserDao.insertUser`.
- `ABORT`: cancela la inserción y lanza excepción. Usado en `SessionDao.insertSession` (no permite sesiones duplicadas).
- `IGNORE`: ignora el nuevo dato si ya existe.

### `Flow` en Room
Cuando un DAO retorna `Flow<T>`, Room observa la tabla y **re-emite automáticamente** cuando los datos cambian. No se necesita polling manual.

```kotlin
// Cada vez que se inserta una sesión, este Flow emite la lista actualizada:
sessionDao.getSessionsByUser(userId): Flow<List<SessionEntity>>
```

### `@Database`
Anota la clase principal de Room. Lista todas las entidades y la versión del schema.

**En GambApp — `AppDatabase` versión 4:**
```kotlin
@Database(
    entities = [UserEntity::class, AppClinicEntity::class, ExerciseEntity::class,
                SessionEntity::class, AchievementEntity::class],
    version = 4,
    exportSchema = false,
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun sessionDao(): SessionDao
    // ... demás DAOs
}
```

### `createFromAsset()`
Pre-popula la base de datos desde un archivo `.db` en `assets/`. Las clínicas kinesiológicas ya vienen cargadas desde `assets/databases/prepopulated.db` sin necesidad de una llamada de red inicial.

### `fallbackToDestructiveMigration(true)`
Si el schema de Room cambia (nueva versión) y no hay migración definida, **borra y recrea** la BD. Útil en desarrollo. En producción se deben definir `Migration` explícitas para no perder datos de usuarios.

### DataStore vs SharedPreferences
| | DataStore | SharedPreferences |
|---|---|---|
| API | Coroutines (suspend, Flow) | Síncrona (con listeners) |
| Tipo-seguro | Sí (con Serialization) | No |
| Thread-safe | Sí | Con restricciones |
| En GambApp | `SessionPreferences` (onboarding, token) | `StepCounterDataSource` (pasos) |

**¿Por qué el podómetro usa SharedPreferences y no DataStore?** El `StepCounterService` (ForegroundService) necesita leer/escribir de forma síncrona y registrar un `OnSharedPreferenceChangeListener` que se convierte a `callbackFlow`. Con DataStore se requeriría más complejidad de setup en el service.

---

## BLOQUE 7: Sensores Android

### `SensorManager`
Servicio del sistema para acceder a los sensores físicos del dispositivo.

```kotlin
val sensorManager = context.getSystemService(Context.SENSOR_SERVICE) as SensorManager
val sensor = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
```

### `SensorEventListener`
Interface con dos callbacks:
- `onSensorChanged(event: SensorEvent)` — emite nuevas lecturas del sensor.
- `onAccuracyChanged(sensor, accuracy)` — notifica cambios de precisión.

**Registro y limpieza:**
```kotlin
sensorManager.registerListener(listener, sensor, SensorManager.SENSOR_DELAY_GAME)
// En awaitClose:
sensorManager.unregisterListener(listener)
```

### Tipos de Sensores usados

| Sensor | Constante | Propósito en GambApp |
| :--- | :--- | :--- |
| Acelerómetro | `TYPE_ACCELEROMETER` | Detección de caídas (magnitud vectorial > 24.5 m/s²) |
| Fotómetro | `TYPE_LIGHT` | Nivel de iluminación (lux) para validar entorno |
| Podómetro hardware | `STEP_COUNTER` | Cuenta pasos acumulados desde el reinicio del dispositivo |

### `SENSOR_DELAY_*`
Define la frecuencia de muestreo del sensor:
- `SENSOR_DELAY_NORMAL` — ~5 Hz. Para datos lentos (orientación).
- `SENSOR_DELAY_UI` — ~16 Hz. Para actualizar UI. Usado en `LightSensorDataSource`.
- `SENSOR_DELAY_GAME` — ~50 Hz. Para juegos/detección rápida. Usado en `AccelerometerDataSource` para capturar picos de caída.

### Cálculo de caída con Acelerómetro
```kotlin
// Los 3 ejes del acelerómetro:
val x = event.values[0]   // horizontal izquierda/derecha
val y = event.values[1]   // vertical arriba/abajo
val z = event.values[2]   // profundidad frente/atrás

val magnitude = sqrt(x*x + y*y + z*z)   // vector resultante en m/s²

// En reposo, magnitud ≈ 9.8 m/s² (gravedad terrestre)
// En caída libre, magnitud ≈ 0 m/s²
// En impacto violento, magnitud >> 9.8 m/s²
val isFall = magnitude > 24.5f   // 2.5G — threshold empírico
```

### `STEP_COUNTER` vs `STEP_DETECTOR`
- `STEP_COUNTER`: acumula el total de pasos **desde el último reinicio del dispositivo**. Es un hardware counter. No se resetea al reiniciar la app. Requiere diferencia: `stepsToday = currentValue - baseValue`.
- `STEP_DETECTOR`: emite un evento por cada paso detectado. Requiere conteo manual.

**En GambApp** se usa `STEP_COUNTER` (más preciso y eficiente en batería) y se calcula el delta diario.

---

## BLOQUE 8: CameraX y ML Kit

### CameraX
Librería Jetpack para cámara que abstrae las diferencias entre dispositivos y maneja automáticamente el ciclo de vida. Construida sobre Camera2.

**Casos de uso (Use Cases) de CameraX:**
- `Preview` — muestra el visor en pantalla.
- `ImageAnalysis` — analiza frames en tiempo real.
- `ImageCapture` — toma fotos.
- `VideoCapture` — graba video.

**En GambApp se usan `Preview` + `ImageAnalysis` simultáneamente.**

### `ImageAnalysis.Analyzer`
Interface con un método `analyze(imageProxy: ImageProxy)`. **Obligatorio** llamar `imageProxy.close()` al terminar, o CameraX deja de enviar frames.

```kotlin
class PoseDetectionDataSource : ImageAnalysis.Analyzer {
    override fun analyze(imageProxy: ImageProxy) {
        val mediaImage = imageProxy.image ?: run {
            imageProxy.close()  // siempre cerrar, incluso si no hay imagen
            return
        }
        val inputImage = InputImage.fromMediaImage(mediaImage, imageProxy.imageInfo.rotationDegrees)
        poseDetector.process(inputImage)
            .addOnSuccessListener { pose -> _poseResult.value = PoseResult(...) }
            .addOnCompleteListener { imageProxy.close() }  // cierra al finalizar
    }
}
```

### `STRATEGY_KEEP_ONLY_LATEST`
Estrategia de backpressure de `ImageAnalysis`. Si el analyzer no puede procesar frames a la velocidad de la cámara (ej: ML Kit tarda ~100ms y la cámara produce frames a 30fps), **descarta los frames intermedios** y solo procesa el más reciente. Evita que se acumule una cola de frames en memoria.

### ML Kit Pose Detection
Modelo de IA de Google para detectar 33 puntos articulares del cuerpo humano en una imagen. Corre **localmente on-device** (no requiere internet). Retorna un objeto `Pose` con `allPoseLandmarks: List<PoseLandmark>`.

**33 landmarks incluyen:** nariz, ojos, orejas, hombros, codos, muñecas, caderas, rodillas, tobillos, dedos de manos y pies.

### Compensación de espejo para cámara frontal
ML Kit retorna coordenadas relativas a la imagen capturada. La cámara frontal captura la imagen espejada. Para que el esqueleto se vea correctamente:
```kotlin
// En SkeletonOverlay.kt:
val mirroredX = logicalWidth - landmark.position.x
val canvasX = mirroredX * scale + offsetX
```
Sin esto, el esqueleto aparece invertido horizontalmente (el brazo derecho del paciente se muestra a la izquierda en pantalla).

### `InputImage.fromMediaImage(mediaImage, rotation)`
Crea un `InputImage` para ML Kit a partir de un frame de CameraX. El parámetro `rotation` es en grados (0, 90, 180, 270) y representa la rotación del dispositivo respecto al sensor de cámara. ML Kit la usa para entregar coordenadas en el espacio "upright" (vertical).

---

## BLOQUE 9: Navigation Compose

### NavHost y NavController
- `NavController`: controla el back stack y la navegación. Creado con `rememberNavController()`.
- `NavHost`: define el grafo de navegación — qué composables corresponden a cada ruta.

```kotlin
NavHost(navController = controller, startDestination = Screen.Dashboard.route) {
    composable(Screen.Dashboard.route) { DashboardScreen(...) }
    composable(Screen.Login.route) { LoginScreen(...) }
    composable(
        route = Screen.RehabSession.route,   // "rehab_session/{exerciseId}"
        arguments = listOf(navArgument("exerciseId") { type = NavType.StringType })
    ) { backStackEntry ->
        val id = backStackEntry.arguments?.getString("exerciseId") ?: ""
        RehabSessionScreen(exerciseId = id, ...)
    }
}
```

### Back Stack
Pila de destinos de navegación. Al navegar a una pantalla, se apila. `popBackStack()` vuelve al anterior.

**`popUpTo` + `inclusive = true`** — Elimina destinos del back stack al navegar. Se usa en Login/Register para que al autenticarse no se pueda volver a la pantalla de login con el botón atrás:
```kotlin
controller.navigate(Screen.Dashboard.route) {
    popUpTo(Screen.Login.route) { inclusive = true }  // elimina Login del stack
}
```

### `launchSingleTop = true`
Si el destino ya está en el top del back stack, no crea una nueva instancia. Evita duplicados al presionar varias veces el mismo ítem del BottomBar.

---

## BLOQUE 10: Retrofit y APIs REST

### Retrofit
Cliente HTTP para Android/Java que convierte interfaces Kotlin en llamadas HTTP. La interfaz define el contrato de la API, Retrofit genera la implementación.

**En GambApp — `RoutingApi`:**
```kotlin
interface RoutingApi {
    @GET("route")
    suspend fun getRoute(
        @Query("point") origin: String,
        @Query("point") destination: String,
        @Query("profile") profile: String = "foot",
    ): RouteResponse
}
```
```kotlin
// Creación en AppModule:
val retrofit = Retrofit.Builder()
    .baseUrl("https://graphhopper.com/api/1/")
    .addConverterFactory(GsonConverterFactory.create())
    .build()
val api: RoutingApi = retrofit.create(RoutingApi::class.java)
```

### GsonConverterFactory
Convierte el JSON de respuesta de la API en objetos Kotlin/Java automáticamente usando reflexión. Las propiedades del `data class` deben coincidir con las claves del JSON (o usar `@SerializedName`).

### FusedLocationProviderClient
API de Google Play Services que fusiona GPS, WiFi y red celular para proveer la mejor ubicación con el menor consumo de batería. Reemplaza el `LocationManager` clásico de Android.

```kotlin
fusedLocationClient.requestLocationUpdates(
    LocationRequest.create().apply {
        interval = 5000          // actualización cada 5 segundos
        fastestInterval = 2000   // mínimo 2 segundos entre actualizaciones
        priority = Priority.PRIORITY_HIGH_ACCURACY
    },
    locationCallback,
    Looper.getMainLooper()
)
```

---

## BLOQUE 11: Testing

### MockK
Librería de mocking para Kotlin. A diferencia de Mockito, soporta clases `final` (default en Kotlin), funciones de extensión y suspend functions.

```kotlin
val rehabRepository = mockk<RehabRepository>()

// Simular una suspend function:
coEvery { rehabRepository.saveSession(any()) } just Runs

// Simular un Flow:
every { rehabRepository.getSessions(any()) } returns flowOf(listOf(mockSession))
```

### Turbine
Librería para testear `Flow`s de Kotlin:
```kotlin
viewModel.uiState.test {
    val initial = awaitItem()   // espera la primera emisión
    assertEquals(true, initial.isLoading)

    val loaded = awaitItem()    // espera la siguiente emisión
    assertEquals(false, loaded.isLoading)
    assertEquals(3, loaded.sessions.size)

    cancelAndConsumeRemainingEvents()
}
```

### `TestCoroutineDispatcher` / `UnconfinedTestDispatcher`
Dispatcher de testing que ejecuta coroutines de forma inmediata y síncrona, evitando esperas en tests.

```kotlin
@get:Rule
val mainDispatcherRule = MainDispatcherRule()  // reemplaza Dispatchers.Main en tests
```

### `coEvery` vs `every`
- `every { ... }` — para funciones normales (no suspend).
- `coEvery { ... }` — para `suspend fun`. Lanza la suspensión en el contexto de test.

---

## BANCO DE PREGUNTAS — PARTE 2

### Room y Persistencia

**P: ¿Por qué usamos `Flow` en los DAOs en lugar de `suspend fun` que retorne una List?**
R: `Flow` es reactivo. Cuando se insertan, modifican o eliminan datos, Room emite automáticamente la lista actualizada a todos los collectors. Con `suspend fun` habría que hacer polling manual. El Dashboard de GambApp se actualiza en tiempo real porque `getSessions()` retorna `Flow<List<Session>>`.

**P: ¿Qué significa `OnConflictStrategy.REPLACE` vs `ABORT`?**
R: `REPLACE` borra la fila existente con el mismo ID y la reinserta con los nuevos datos (útil para `UserDao`, ya que el usuario puede actualizar su perfil). `ABORT` lanza una excepción si ya existe una fila con el mismo ID (útil para `SessionDao`, donde cada sesión es única e irrepetible).

**P: ¿Qué riesgo tiene `fallbackToDestructiveMigration` en producción?**
R: Borra toda la base de datos del usuario cuando hay un cambio de schema. Si un usuario tenía sesiones de rehabilitación guardadas y se publica una versión nueva con una tabla modificada, **pierde todo su historial**. En producción se deben escribir `Migration` explícitas con `addMigrations()`.

### Sensores

**P: ¿Por qué el acelerómetro necesita `SENSOR_DELAY_GAME` y no `SENSOR_DELAY_NORMAL`?**
R: Una caída dura milisegundos. Con `SENSOR_DELAY_NORMAL` (~200ms entre muestras), el pico de aceleración podría ocurrir entre dos muestras y nunca detectarse. Con `SENSOR_DELAY_GAME` (~20ms entre muestras) se captura el pico antes de que desaparezca.

**P: ¿Por qué `STEP_COUNTER` requiere calcular un delta?**
R: `STEP_COUNTER` es un hardware counter que **acumula pasos desde el último reinicio del dispositivo**, nunca se resetea por sí solo. Si el dispositivo lleva 50.000 pasos acumulados y el usuario caminó 3.000 hoy, hay que almacenar el valor base al inicio del día y calcular `stepsHoy = 53000 - 50000 = 3000`.

### CameraX y ML Kit

**P: ¿Por qué `imageProxy.close()` es crítico?**
R: `ImageAnalysis` tiene un buffer de 1 frame. Si el frame no se cierra después de procesar, el pipeline de CameraX queda bloqueado esperando que se libere el buffer. La app deja de recibir nuevos frames y la cámara se "congela".

**P: ¿ML Kit envía imágenes a internet?**
R: No. La detección de pose de ML Kit es completamente on-device. El modelo de IA está embebido en el APK. Esto garantiza privacidad de los datos biomédicos del paciente y funcionamiento offline.

**P: ¿Por qué se invierte la coordenada X en `SkeletonOverlay`?**
R: La cámara frontal captura la imagen espejada (como un espejo real). ML Kit retorna las coordenadas del frame original (no espejado). Al invertir `X = imageWidth - landmark.x`, el esqueleto dibujado coincide con lo que el usuario ve en pantalla (su brazo derecho aparece a la derecha).

### Navigation

**P: ¿Cuál es la diferencia entre `popUpTo` con y sin `inclusive = true`?**
R: `popUpTo("login") { inclusive = false }` elimina todo **hasta** Login (Login queda en el stack). `inclusive = true` también elimina Login. En GambApp, al hacer login exitoso se usa `inclusive = true` para que el botón atrás desde el Dashboard no lleve de vuelta a Login.

**P: ¿Por qué `startDestination` del NavHost está mal en el proyecto actual?**
R: Está fijado en `Screen.Dashboard.route` en lugar de `Screen.Splash.route`. Esto salta la `SplashScreen` y su ViewModel, que verifica si el onboarding fue completado y decide a dónde navegar. El resultado es que el usuario siempre entra al Dashboard sin autenticarse.

### Retrofit y Network

**P: ¿Qué hace `@Query` en Retrofit?**
R: Agrega parámetros a la URL como query string. `@Query("point") "lat,lng"` genera `?point=lat,lng`. Con múltiples `@Query` del mismo nombre (como en GraphHopper que recibe dos puntos) se pueden repetir: `?point=orig&point=dest`.

**P: ¿Cómo maneja Retrofit los errores HTTP?**
R: Si el servidor retorna un código >= 400, Retrofit lanza `HttpException`. Si hay falla de red (sin internet), lanza `IOException`. En GambApp se capturan con `runCatching` o `.catch { }` en el Flow pipeline.

**P: ¿Por qué Firebase tiene credenciales dummy en el proyecto?**
R: Es un proyecto universitario sin un backend Firebase real asignado. El `LoginUseCase.loginWithMock()` simula la autenticación localmente para poder demostrar el flujo sin depender de un servidor. En producción se reemplaza con el `google-services.json` del proyecto Firebase real.

### Testing

**P: ¿Por qué se usan tests JVM y no instrumented tests (con emulador)?**
R: Los tests JVM son 10-20x más rápidos porque corren directamente en la JVM local sin necesitar un emulador o dispositivo. Son suficientes para testear ViewModels, Use Cases y Repositorios con mocks. Los instrumented tests se reservan para UI tests de Compose o tests que requieren SQLite real.

**P: ¿Qué testea `CalculateJointAngleUseCaseTest`?**
R: Que el cálculo geométrico de ángulos entre 3 puntos sea correcto. Casos cubiertos: ángulo de 90° (L perfecta), 180° (línea recta), 0° (puntos superpuestos), y valores intermedios. Es un test purísimo — sin mocks, sin Android SDK, solo matemática.

**P: ¿Qué es la cobertura de código y por qué debe superar el 80%?**
R: La cobertura (code coverage) mide el porcentaje de líneas/branches de código que son ejecutadas por al menos un test. La consigna exige >80% para garantizar que la mayoría de la lógica crítica esté validada. En GambApp se mide con Kover y se reporta en formato XML para integración con CI.
