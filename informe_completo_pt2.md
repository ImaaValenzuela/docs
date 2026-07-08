# Informe Técnico Exhaustivo — GambApp (Parte 2)

---

## Sección 4 (continuación): Responsabilidades por Desarrollador

### DEV 3 — Mapa & Red Médica

**Pantallas a cargo**: Mapa red médica, Detalle de clínica + ruta

#### 4.3.1 Geolocalización con FusedLocationProviderClient

**`ObserverLocationUseCase.kt`** — Emite la ubicación del usuario como `Flow<Location>`:
```kotlin
// Solicita actualizaciones periódicas con alta precisión
fusedLocationClient.requestLocationUpdates(request, callback, Looper.getMainLooper())
```

**¿Por qué `FusedLocationProviderClient` y no `LocationManager`?** Google Play Services fusiona GPS, WiFi y redes celulares para dar la mejor ubicación disponible con el menor consumo de batería. `LocationManager` solo usa el proveedor que se especifique explícitamente, sin la inteligencia de fusión.

**Permisos en tiempo de ejecución**: Se solicitan `ACCESS_FINE_LOCATION` y `ACCESS_COARSE_LOCATION`. La pantalla de mapa muestra un componente `LocationDeniedCard` cuando el permiso está denegado, con botón para llevar al usuario a Configuración del sistema.

#### 4.3.2 Carga de Clínicas desde Assets (JSON local)

**`GetClinicsFromAssetsUseCase.kt`** — Carga un archivo JSON con la lista de clínicas:
- Las clínicas se persisten en Room para acceso offline.
- El mapper `ClinicMapper` convierte los modelos de JSON (`ClinicDto`) a entidades de dominio (`Clinic`).

**`getNearestClinics()`** — Función de extensión que calcula las clínicas más cercanas usando la **fórmula de Haversine** para calcular distancia entre coordenadas geográficas (considerando la curvatura de la Tierra):
```kotlin
val dLat = Math.toRadians(lat2 - lat1)
val dLng = Math.toRadians(lng2 - lng1)
val a = sin(dLat/2).pow(2) + cos(lat1) * cos(lat2) * sin(dLng/2).pow(2)
val c = 2 * asin(sqrt(a))
return EARTH_RADIUS_KM * c
```

#### 4.3.3 MapTiler + Clusters + Polilínea de Ruta

**`MapScreenViewModel.kt`** gestiona:
1. **Clusters**: Agrupación visual de pines de clínicas con `GeoJSON + MTGeoJSONSource`. Cuando el zoom es bajo, las clínicas se agrupan; al acercar se expanden individualmente.
2. **Polilínea de ruta**: Se dibuja con `MTPolylineLayerHelper` usando datos GeoJSON devueltos por la API de GraphHopper.
3. **`MIN_DISTANCE_FOR_NEW_ROUTE = 20.0m`**: Evita recalcular la ruta si el usuario se movió menos de 20 metros desde la última solicitud, reduciendo el consumo de API.

**`GetRouteUseCase.kt`** — Llama a la API REST de GraphHopper mediante Retrofit2:
```
GET /route?point={lat},{lng}&point={lat},{lng}&profile=foot&locale=es
```
Retorna distancia en metros, tiempo estimado y la geometría de la ruta en formato GeoJSON codificado.

**Bottom Sheet de detalle de clínica**: Al seleccionar un pin, se muestra un `ModalBottomSheet` con nombre, dirección, teléfono y botón "Trazar ruta". Esto sigue el patrón de Material3 para detalles contextuales.

**Decisión de diseño:** Se usó MapTiler en lugar de Google Maps para evitar el cobro por uso de la API de Google Maps, que tiene un free tier limitado. MapTiler ofrece mapas vectoriales de alta calidad con un plan gratuito más generoso.

---

### DEV 4 — Dashboard & Progreso

**Pantallas a cargo**: Dashboard home, Progreso, Estadísticas, Listado de Rutinas, Logros

#### 4.4.1 DashboardViewModel — Combining múltiples Flows

El Dashboard combina datos de **4 fuentes reactivas simultáneas**:
```kotlin
combine(
    rehabRepository.getSessions(userId),    // Flow<List<Session>> — Room
    stepCounterDataSource.getStepsFlow(),   // Flow<Int> — SharedPreferences
    achievementRepository.getAchievements() // Flow<List<Achievement>> — Room
) { sessions, steps, achievements -> ... }
```

**¿Por qué `combine`?** Este operador de Kotlin Coroutines re-emite un nuevo valor cada vez que **cualquiera** de las fuentes upstream emite. Permite mantener el Dashboard sincronizado con cambios en tiempo real en pasos, sesiones y logros sin suscripción manual a cada fuente.

**Métricas derivadas computadas en el ViewModel:**
- **Calorías**: `MET × 70kg × (durationSeconds / 3600)`. MET=5 representa ejercicio de rehabilitación moderado.
- **Distancia**: `steps × 0.000762 km` (zancada promedio médica estándar de 76.2 cm).
- **Minutos activos**: `sum(durationSeconds) / 60` de todas las sesiones del día.

#### 4.4.2 StepCounterService — Servicio Persistente en Primer Plano

**`StepCounterService.kt`** — `ForegroundService` que sobrevive al cierre de la app:
- Muestra una **notificación persistente** con el conteo de pasos actual (requerimiento Android 8+ para servicios en primer plano).
- Escucha el sensor `STEP_COUNTER` de hardware, que acumula pasos desde el reinicio del dispositivo.
- Calcula los pasos del día actual: `stepsToday = currentSensorValue - baselineSensorValue`.
- Persiste en `SharedPreferences` para compartir el dato con `StepCounterDataSource`.
- Resetea el contador automáticamente a medianoche verificando la fecha del último reset.

**¿Por qué no usar `WorkManager`?** `WorkManager` es para tareas en background diferibles y periódicas. El conteo de pasos requiere escuchar el sensor **de forma continua** mientras el usuario camina. `ForegroundService` es la única opción correcta para tareas que requieren CPU/sensores en tiempo real con visibilidad del usuario.

#### 4.4.3 HealthConnectDataSource

Integración opcional con Health Connect (API de Google para datos de salud interoperables):
- Lee datos de salud de otras apps (pasos, sesiones de ejercicio) que el usuario haya compartido con Health Connect.
- Requiere permiso `READ_STEPS` declarado en el `AndroidManifest.xml`.
- Los datos se usan complementariamente al podómetro local.

#### 4.4.4 Sistema de Logros (Achievements)

**`AchievementRepositoryImpl.kt`** + Room DAO `AchievementDao`:
- Modelos persistidos: `Achievement(id, title, description, isUnlocked, unlockedAtTimestamp)`.
- `DashboardViewModel.checkAndUnlockAchievements()` evalúa condiciones en cada emisión del `combine`.
- Al desbloquear un logro, `_uiState.update { it.copy(newlyUnlockedAchievement = ...) }` dispara la animación de celebración en el Dashboard.

**Logros implementados:**
| ID | Título | Condición |
| :--- | :--- | :--- |
| `first_session` | Primer Paso | Completar al menos 1 sesión |
| `10k_steps` | Pasos Legendarios | Alcanzar 10.000 pasos en un día |
| `master_rom` | Flexibilidad Suprema | ROM promedio ≥ 120° en una sesión |

---

## Sección 5: Temas de Clase y su Implementación

| Clase | Tema | Implementación en GambApp |
| :--- | :--- | :--- |
| 01 | Integración Continua | CI local con `pr-check.sh` (ktlint → test → kover → lint). GitHub Actions para CI remoto |
| 02 | Diseños | Material3 completo, paleta custom con tokens de color separados en `Color.kt` y `Theme.kt` |
| 03 | Jetpack Compose Animations | `animateColorAsState` (SkeletonOverlay), `AnimatedVisibility` (Dashboard), Lottie (Splash) |
| 04 | Cámara | CameraX con `Preview` + `ImageAnalysis`, rotación y espejado manual |
| 04 | Testing | MockK, Turbine (testing de Flows), JUnit4, 35 archivos de tests unitarios |
| 05 | Permisos | `CAMERA`, `ACCESS_FINE_LOCATION`, `ACTIVITY_RECOGNITION`, `FOREGROUND_SERVICE` — runtime permissions con flujo de UI para denegados |
| 06 | Ubicación y Mapas | FusedLocationProviderClient, MapTiler SDK, GraphHopper API, Haversine para proximidad |
| 07 | Bluetooth y Sensores | Acelerómetro (TYPE_ACCELEROMETER), Fotómetro (TYPE_LIGHT), Podómetro (STEP_COUNTER) con `callbackFlow` |
| 07 | Interacción con apps default | `Intent(Intent.ACTION_CALL)`, `Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS)` |
| 08 | Motion Layout | Transiciones `slideInHorizontally` + `fadeIn/fadeOut` en NavHost, `AnimatedVisibility` |

---

## Sección 6: Análisis de Preguntas Frecuentes de Entrevistas Android

### Bloque A: Kotlin y Coroutines

**¿Qué es `StateFlow` y en qué se diferencia de `LiveData`?**
- `StateFlow` es un Flow de estado en Kotlin Coroutines que emite siempre el último valor. Es awareness del ciclo de vida solo cuando se usa con `collectAsStateWithLifecycle()` en Compose. `LiveData` es la alternativa de Jetpack de la era pre-Coroutines, acoplada a `LifecycleOwner`. En GambApp se usa `StateFlow` en todos los ViewModels porque es el estándar moderno de Google y funciona nativamente con `viewModelScope`.

**¿Qué es `callbackFlow` y cuándo usarlo?**
- Permite convertir APIs de callback (listeners) a Flows reactivos. Es la elección correcta cuando una librería de terceros (como SensorManager o SharedPreferences) usa el patrón Observer con callbacks Java. Se usa en `LightSensorDataSource`, `AccelerometerDataSource` y `StepCounterDataSource`.

**¿Qué hace `awaitClose`?**
- Es una función de suspensión que mantiene abierto el `callbackFlow` hasta que el collector cancele su recolección. Dentro se ejecuta el código de limpieza (ej: `sensorManager.unregisterListener(listener)`), garantizando que no queden listeners activos después de que la pantalla se destruya.

**¿Qué es `flatMapLatest`?**
- Operador que cancela el Flow interno anterior al emitir un nuevo valor upstream. Es la solución al problema de Flows anidados en `PostSessionViewModel`. Ejemplo: si cambia la sesión seleccionada, cancela la suscripción al ejercicio anterior y empieza una nueva.

**¿Diferencia entre `launch` y `async`?**
- `launch` es "fire and forget" — lanza una coroutine sin retornar un resultado. `async` retorna un `Deferred<T>` que puede awaitearse con `.await()` para obtener el valor. En GambApp se usa `launch` para operaciones de UI y escritura en BD donde no se necesita el resultado directo.

### Bloque B: Arquitectura Android (MVVM + Clean)

**¿Por qué el ViewModel no debería tener referencias a Context?**
- El ViewModel sobrevive a los cambios de configuración (rotaciones de pantalla). Si tiene una referencia a `Context` (que está ligado al ciclo de vida de la Activity), puede causar memory leaks. En `MapScreenViewModel` se usa `@ApplicationContext` (Hilt) que inyecta el contexto de la aplicación, que tiene un ciclo de vida más largo y seguro.

**¿Qué es Hilt y por qué se usa en lugar de Dagger puro?**
- Hilt es una capa de abstracción sobre Dagger que reduce el boilerplate de inyección de dependencias en Android. Genera automáticamente los componentes de Dagger para cada ámbito de Android (`@ActivityComponent`, `@ViewModelComponent`, etc.). En GambApp, `@HiltViewModel` permite que los ViewModels reciban sus dependencias sin código adicional en la Factory.

**¿Qué es `SharingStarted.WhileSubscribed(5000)` en `stateIn`?**
- Controla cuándo el Flow upstream empieza y para de recolectar. `WhileSubscribed(5000)` mantiene el Flow activo 5 segundos después de que el último subscriber se desuscriba. Esto evita reconectar si el usuario rota la pantalla (la Activity se destruye y recrea en ~500ms, pero los 5000ms dan margen). Se usa en `SplashViewModel`.

**¿Diferencia entre `Repository` y `DataSource`?**
- El `DataSource` es la fuente de datos concreta (Room DAO, SensorManager, Retrofit service). El `Repository` es la fachada que combina y abstrae múltiples `DataSources`, exponiendo una interfaz limpia al dominio. Los ViewModels y Use Cases solo conocen al Repository, nunca al DataSource directamente.

### Bloque C: Jetpack Compose

**¿Qué es `recomposition` y cómo se minimiza?**
- La recomposición es el proceso en el que Compose re-dibuja un Composable cuando su estado cambia. Para minimizarla: usar `remember`, `derivedStateOf` para estados computados, y evitar leer estados volátiles en el nivel del Composable padre. En GambApp, `collectAsStateWithLifecycle()` en lugar de `collectAsState()` evita recomposiciones cuando la app está en background.

**¿Qué hace `LaunchedEffect`?**
- Lanza una coroutine en el ciclo de vida del Composable. Se re-ejecuta cuando alguna de sus keys cambia. En `SplashScreen`, se usa `LaunchedEffect(uiState)` para navegar cuando el estado cambia de `Loading` a `Ready`, garantizando que la navegación ocurra en el hilo principal del Compositor y no en el de la coroutine del ViewModel.

**¿Qué es `hiltViewModel()` en Compose?**
- Es una función de Compose que obtiene un ViewModel inyectado con Hilt desde el `BackStackEntry` actual de Navigation. Garantiza que el ViewModel tenga el ámbito correcto de vida ligado a cada destino de navegación.

### Bloque D: Testing

**¿Qué es MockK?**
- Librería de mocking nativa de Kotlin. A diferencia de Mockito, soporta clases `final` (que en Kotlin son por defecto), objetos singleton, funciones de extensión y funciones de suspensión (`coEvery`, `coVerify`). GambApp la usa para simular Repositorios y DAOs en los tests de ViewModels.

**¿Qué es Turbine?**
- Librería para testear `Flow`s de Kotlin de forma sencilla. Permite llamar `awaitItem()` para esperar el próximo valor emitido, `awaitComplete()` para esperar la finalización, y `cancelAndConsumeRemainingEvents()` para limpiar. Se usa en `DashboardViewModelTest` y `ProgressViewModelTest`.

**¿Qué es Kover?**
- Plugin de Gradle de JetBrains para medir la cobertura de código de proyectos Kotlin. Genera reportes en formato XML/HTML. En GambApp está configurado como tarea `koverXmlReportRelease` dentro de la pipeline de PR check.

**¿Por qué se usan tests JVM y no Instrumented tests?**
- Los tests JVM (en `src/test`) corren en la JVM local sin necesidad de un emulador, son mucho más rápidos y son la opción correcta para testear lógica de negocio pura (Use Cases, ViewModels con mocks, Repositorios). Los Instrumented tests (en `src/androidTest`) son necesarios solo cuando se requiere el contexto real de Android (Room con SQLite, UI tests de Compose).

---

## Sección 7: Toma de Decisiones Técnicas Clave

| Decisión | Alternativa Descartada | Razón de la Elección |
| :--- | :--- | :--- |
| **ML Kit on-device** | Enviar frames a API cloud | Privacidad del usuario (datos biomédicos), funciona offline, sin latencia de red |
| **MapTiler** | Google Maps SDK | Costo: MapTiler tiene plan gratuito más generoso para proyectos universitarios |
| **Room** | SQLite directo / Firebase Firestore | Room provee seguridad de tipos en queries SQL, integración nativa con Flow/LiveData, migraciones estructuradas |
| **CameraX** | Camera2 API | CameraX abstrae las diferencias entre dispositivos, maneja rotación y ciclo de vida automáticamente |
| **Hilt** | Dagger puro / Koin | Hilt es el estándar oficial de Google para Android DI, reduce boilerplate vs Dagger puro y es más performante que Koin |
| **GraphHopper** | Google Directions API | Google Directions API cobra por uso. GraphHopper tiene servidor propio con plan gratuito |
| **callbackFlow para sensores** | `Channel` + coroutine | `callbackFlow` es el patrón idiomático de Kotlin para convertir callbacks. Maneja automáticamente el backpressure y la cancelación |
| **StateFlow sobre LiveData** | LiveData | StateFlow es multiplataforma, no requiere LifecycleOwner, es más composable con operadores de Flow |
