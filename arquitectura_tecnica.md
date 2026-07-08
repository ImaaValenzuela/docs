# Documentación Técnica de Arquitectura — GambApp

---

## 1. Mapa de Capas y Conexiones

```
┌─────────────────────────────────────────────────────────────┐
│  UI LAYER — Jetpack Compose                                  │
│  Screens: Login, Dashboard, RoutineList, Rehab, Map...       │
│  Cada Screen obtiene su estado de un @HiltViewModel          │
└───────────────────┬─────────────────────────────────────────┘
                    │  StateFlow / collectAsStateWithLifecycle
┌───────────────────▼─────────────────────────────────────────┐
│  VIEWMODEL LAYER — @HiltViewModel                            │
│  Expone UiState inmutable. Llama Use Cases / Repositories.   │
│  Corre en viewModelScope (Dispatchers.Main.immediate)        │
└──────────┬───────────────────────┬──────────────────────────┘
           │                       │
    Domain UseCases         Application UseCases
    (domain/usecase)        (application/usecases)
           │                       │
┌──────────▼───────────────────────▼──────────────────────────┐
│  REPOSITORY LAYER                                            │
│  domain/repository → interfaces                              │
│  data/repositories → implementaciones concretas              │
└──────────────┬──────────────────────────────────────────────┘
               │
     ┌─────────┼──────────────┬───────────────┐
     ▼         ▼              ▼               ▼
  Room DB  SharedPrefs   DataStore       Retrofit/API
  (SQLite)  (Steps)    (Settings)       (GraphHopper)
```

---

## 2. Bases de Datos y Almacenamiento

### 2.1 Room Database — `AppDatabase` (SQLite local)
**Archivo:** `data/datasources/local/database/AppDatabase.kt`
**Nombre del archivo en disco:** `app_database`
**Versión actual:** 4
**Estrategia de migración:** `fallbackToDestructiveMigration(true)` — en cambios de schema, borra y recrea la BD.

La DB se inicializa **pre-poblada** desde un asset:
```kotlin
Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
    .createFromAsset("databases/prepopulated.db")   // clínicas cargadas desde assets/
    .fallbackToDestructiveMigration(true)
    .build()
```

#### Tablas (Entities → DAOs)

| Tabla | Entity | DAO | Propósito |
| :--- | :--- | :--- | :--- |
| `users` | `UserEntity` | `UserDao` | Perfil del usuario autenticado |
| `clinics` | `AppClinicEntity` | `ClinicsDao` | Clínicas kinesiológicas (pre-poblado desde asset) |
| `sessions` | `SessionEntity` | `SessionDao` | Historial de sesiones de rehabilitación |
| `exercises` | `ExerciseEntity` | `ExerciseDao` | Catálogo de ejercicios disponibles |
| `achievements` | `AchievementEntity` | `AchievementDao` | Logros del usuario (bloqueados/desbloqueados) |

#### Queries relevantes por DAO

**`UserDao`:**
```sql
SELECT * FROM users LIMIT 1           -- getUser(): Flow<UserEntity?>
SELECT * FROM users WHERE email = ?   -- getUserByEmail(): suspend
DELETE FROM users                     -- clearUser(): suspend (logout)
```

**`SessionDao`:**
```sql
SELECT * FROM sessions WHERE userId = ? ORDER BY dateTimestamp DESC
-- getSessionsByUser(): Flow<List<SessionEntity>>
-- OnConflict: ABORT (no permite duplicados de sesión)
```

**`ClinicsDao`:**
```sql
SELECT * FROM clinics ORDER BY id DESC   -- getClinics(): Flow<List<AppClinicEntity>>
SELECT COUNT(*) FROM clinics             -- getClinicCount(): para saber si hay que poblar
```

### 2.2 DataStore — `SessionPreferences`
**Archivo:** `data/datasources/local/preferences/SessionPreferences.kt`
**Nombre del archivo:** `settings_preferences`
**Tipo:** `androidx.datastore.preferences` (Jetpack DataStore, sucesor de SharedPreferences para valores simples)

| Clave | Tipo | Propósito |
| :--- | :--- | :--- |
| `session_token` | `String` | Token de sesión Firebase Auth |
| `onboarding_completed` | `Boolean` | Si el usuario ya vio el onboarding |
| `dark_mode` | `Boolean` | Preferencia de tema del usuario |

Todos los valores se exponen como `Flow<T>` reactivos. `SplashViewModel` lee `isOnboardingCompleted` para decidir si mostrar onboarding o login.

### 2.3 SharedPreferences — Contador de Pasos
**Archivo:** `data/datasources/sensor/StepCounterDataSource.kt`
**Nombre del archivo:** `step_counter_prefs`
**Tipo:** `SharedPreferences` clásico (no DataStore, para compatibilidad con el ForegroundService)

| Clave | Tipo | Propósito |
| :--- | :--- | :--- |
| `steps_today` | `Int` | Pasos acumulados en el día actual |
| `last_sensor_value` | `Float` | Último valor acumulado del sensor STEP_COUNTER (base para delta diario) |
| `last_reset_date` | `String` | Fecha del último reset ("yyyy-MM-dd") para reseteo a medianoche |

`StepCounterService` escribe aquí. `StepCounterDataSource.getStepsFlow()` lo lee vía `OnSharedPreferenceChangeListener` convertido a `callbackFlow`.

### 2.4 MapScreen Preferences — `MapScreenPreferences`
**Archivo:** `data/datasources/local/preferences/MapScreenPreferences.kt`
Persiste el **ID de la última clínica de destino** seleccionada, para poder ofrecer "Restaurar ruta" al volver al mapa.

---

## 3. APIs Externas y Dónde se Ejecutan

### 3.1 GraphHopper API — Cálculo de Rutas a Pie
**Archivo:** `application/port/out/remote/routing/RoutingApi.kt`
**Base URL:** `Constants.GRAPH_HOPPER_BASE_URL`
**Cliente HTTP:** Retrofit2 + GsonConverterFactory (singleton en `AppModule`)

```
GET /route
  ?point={lat,lng}        ← Ubicación del usuario
  &point={lat,lng}        ← Coordenadas de la clínica destino
  &profile=foot           ← Perfil: a pie
  &locale=es              ← Respuesta en español
  &calc_points=true
  &instructions=false
  →  RouteResponse { distance: Double, time: Long, points: GeoJSON }
```

**Flujo de llamada:**
```
MapScreenViewModel.onCreateRouteClick()
  → GetRouteUseCase.getRoute(origin, destination)         ← interface
    → GetRouteInteractor.getRoute()                       ← implementación
      → RoutingRepository.getRoute()                      ← interface
        → RoutingRepositoryImpl.getRoute()                ← llama a Retrofit
          → RoutingApi.getRoute()                         ← HTTP GET
            → Respuesta parseada a RouteResponse
          → GeoJSON decodificado en MapScreenViewModel
            → MTPolylineLayerHelper dibuja la polilínea en el mapa
```

La llamada HTTP corre en `Dispatchers.IO` implícitamente a través de Retrofit's suspend support. El ViewModel recibe el resultado en `viewModelScope` (Main thread) ya como objeto parseado.

### 3.2 Firebase Auth — Autenticación
**Archivo:** `AppModule.kt` (provee `FirebaseAuth`)
**Nota:** Configurado con credenciales dummy para el entorno universitario:
```kotlin
FirebaseOptions.Builder()
    .setApiKey("dummy_api_key")
    .setApplicationId("ar.edu.unlam.mobile.scaffolding")
    .setProjectId("dummy-project")
    .build()
```
En producción, reemplazar con el `google-services.json` real. La autenticación actual simula el login a través de `LoginUseCase.loginWithMock()`.

### 3.3 ML Kit Pose Detection — On-Device (sin red)
**Archivo:** `data/datasources/device/mlkit/PoseDetectionDataSource.kt`
No realiza llamadas de red. El modelo de detección de pose se ejecuta **localmente en el dispositivo** usando aceleración de hardware (GPU/NPU si disponible). Los frames YUV_420_888 de CameraX se convierten a `InputImage` y se procesan en el `ImageAnalysis` executor thread.

---

## 4. Flujos Técnicos Detallados por Funcionalidad

### FLOW 1 — Login y Gestión de Sesión (Dev 1)

```
[1] User ingresa email + password en LoginScreen
[2] LoginScreen llama viewModel.onLogin()
[3] LoginViewModel valida localmente (Regex email, password no vacío)
    → Si hay error: actualiza _formState con errorEmail/errorPassword → UI muestra error inline
[4] Si validación OK:
    _uiState = Loading → LoginScreen muestra CircularProgressIndicator
[5] viewModelScope.launch {
      loginUseCase(email, password)         ← suspend fun
        → FirebaseAuth.signInWithEmailAndPassword()
          → Si OK: sessionPreferences.saveSessionToken(token)
                   userRepository.saveUser(user)   ← INSERT OR REPLACE en tabla 'users'
          → Si error: throws Exception con mensaje de Firebase
    }
[6] runCatching captura resultado:
    → Success: _uiState = Success
    → Error: _uiState = Error(UiText)
[7] LoginScreen observa uiState con LaunchedEffect:
    → Success → controller.navigate(Screen.Dashboard)
    → Error → Snackbar visible
[8] Al cerrar sesión (ProfileScreen → SignOutUseCase):
    → sessionPreferences.clearSession()
    → userDao.clearUser()
    → FirebaseAuth.signOut()
    → navController.navigate(Screen.Login) { popUpTo(0) { inclusive = true } }
```

**Persistencia de sesión:**
`SplashViewModel` lee `sessionPreferences.isOnboardingCompleted` → `Flow<Boolean>` → si `true`, navega directo a Login (el token se valida en Login). Si `false`, navega a Onboarding.

---

### FLOW 2 — Sesión de Rehabilitación con Cámara (Dev 2)

```
[1] RoutineListScreen lista ejercicios desde RehabRepository.getExercises()
    → ExerciseDao.getAllExercises() → Flow<List<ExerciseEntity>>
    → .map { entities → entities.map { it.toDomain() } }  ← mapper Data→Domain
    → RoutineListViewModel._uiState.exercises actualizado

[2] User selecciona ejercicio → navega a EnvironmentCheckScreen(exerciseId)
    ⚠️ BUG ACTIVO: inyecta RoutineListViewModel en vez de EnvironmentCheckViewModel
    → LightSensorDataSource no se activa en UI

[3] User presiona "Comenzar" → navega a RehabSessionScreen(exerciseId)

[4] RehabSessionScreen invoca:
    viewModel.loadExercise(exerciseId)
      → rehabRepository.getExerciseById(id)   ← ExerciseDao query por id
      → _currentExercise.value = exercise

    viewModel.startCamera(lifecycleOwner, surfaceProvider)
      → CameraSessionPort.start()
        → CameraXSessionAdapter:
           val preview = Preview.Builder().build()
           val imageAnalysis = ImageAnalysis.Builder()
               .setBackpressureStrategy(STRATEGY_KEEP_ONLY_LATEST)  ← descarta frames viejos
               .build()
           imageAnalysis.setAnalyzer(executor, poseDetectionDataSource)
           cameraProvider.bindToLifecycle(lifecycleOwner, FRONT_CAMERA, preview, imageAnalysis)

[5] Por cada frame de cámara → PoseDetectionDataSource.analyze(imageProxy):
      inputImage = InputImage.fromMediaImage(mediaImage, rotation)
      poseDetector.process(inputImage)
        .addOnSuccessListener { pose →
            _poseResult.value = PoseResult(pose, imageWidth, imageHeight, rotation)
        }
      imageProxy.close()   ← CRÍTICO: debe cerrarse siempre o CameraX se bloquea

[6] RehabSessionViewModel.init() colecta poseResult:
      poseDetectionDataSource.poseResult.collectLatest { result →
          landmarks = exercise.targetJoints.mapNotNull { getLandmarkTypeByName(it) }
          if (landmarks.size == 3) {
              angle = calculateJointAngleUseCase.execute(p0, p1, p2)
              _currentAngle.value = angle
              _precision.value = syncMotorUseCase.execute(angle, exercise)
              (completed, newWasAtTarget) = syncMotorUseCase.checkRepetition(...)
              if (completed) {
                  _repetitionCount.value += 1
                  if (repsCount >= exercise.repetitions) finishSession()
              }
          }
      }

[7] SkeletonOverlay Composable observa poseResult + precision:
      → Canvas.drawLine() para cada conexión corporal
      → animateColorAsState(IDEAL=verde, WARNING=amarillo, ERROR=rojo)

[8] Al detectar caída: AccelerometerDataSource emite SensorReading
      → isFallDetected(reading) → magnitude > 24.5 m/s²
      → _fallDetected.value = true → FallAlertDialog aparece

[9] finishSession():
      session = Session(userId, exerciseId, timestamp, duration, averageRom, reps)
      rehabRepository.saveSession(session)
        → sessionDao.insertSession(session.toEntity())  ← INSERT ABORT (no duplica)
      _isSessionFinished.value = true → UI navega a PostSession

[10] PostSessionViewModel.loadLastSession():
      rehabRepository.getSessions("user_imanol").collectLatest { sessions →
          lastSession = sessions.first()   ← la más reciente por ORDER BY dateTimestamp DESC
          rehabRepository.getExerciseById(lastSession.exerciseId).collectLatest { exercise →
              _exercise.value = exercise
          }
      }

[11] onCleared():
      cameraSession.stop()   ← unbind de CameraX, libera cámara
```

---

### FLOW 3 — Mapa de Clínicas y Ruta (Dev 3)

```
[1] MapScreen se compone → MapScreenViewModel.init():
    a) getLastDestinationClinicIdUseCase()
       → MapPrefsRepositoryImpl → MapScreenPreferences (SharedPreferences)
       → _uiState.lastSavedClinicId = id guardado

    b) observerLocationUseCase.getLocation()   ← Flow<Location>
       → FusedLocationProviderClient.requestLocationUpdates()
       → Emite Location cada vez que el GPS actualiza
       → _uiState.location = location
       → Si mapa ya inicializado: mapController.jumpTo(LngLat(lng, lat), zoom=15)

    c) getClinicsStoredUseCase()
       → ClinicsRepositoryPort.getClinics()   ← Flow<List<AppClinicEntity>>
       → Si count == 0: PopulateClinicsDbUseCase() carga desde assets/clinics.json
         → parsea JSON → mapper DTO→Entity → ClinicsDao.insertAll()
       → getNearestClinics(location, clinics, count=15)
         → Haversine por cada clínica → ordena por distancia → toma top 15
       → _uiState.clinics + clinicsNear actualizados
       → setupClusters() → MTGeoJSONSource con coordenadas → renderiza pines en mapa

[2] User selecciona clínica (tap en pin o en lista):
    onClinicSelectedChange(clinic)
    → _uiState.selectedClinic = clinic → BottomSheet se abre

[3] User presiona "Trazar ruta":
    onCreateRouteClick(hexColor)
    → Verifica distancia desde última solicitud > MIN_DISTANCE_FOR_NEW_ROUTE (20m)
      (evita llamadas repetidas si el usuario no se movió)
    → viewModelScope.launch {
         getRouteUseCase.getRoute(origin=userLocation, dest=clinicLocation)
           → GetRouteInteractor → RoutingRepositoryImpl.getRoute()
             → Retrofit: GET /route?point=...&point=...&profile=foot
             → RouteResponse { distance, time, points: GeoJSON }
         saveLastDestinationClinicIdUseCase(clinic.id)
           → MapPrefsRepositoryImpl → MapScreenPreferences.save()
         // Dibuja polilínea:
         MTGeoJSONSource.setData(geoJsonString)
         MTPolylineLayerHelper con color hexColor, ancho adaptativo por zoom
       }
    → _uiState.routeDistance + routeTime actualizados → UI muestra ETA

[4] User presiona "Llamar" en el BottomSheet:
    onCallTriggered()
    → Intent(ACTION_DIAL) { data = "tel:${clinic.phone}" }
    → context.startActivity(intent)   ← abre app de teléfono del SO

[5] Permiso de ubicación denegado:
    onGoToConfigClick()
    → Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS)
    → context.startActivity(intent)   ← abre Ajustes > Apps > GambApp > Permisos
```

---

### FLOW 4 — Dashboard, Pasos y Logros (Dev 4)

```
[1] DashboardScreen se crea → LaunchedEffect(Unit):
    → Verifica permisos ACTIVITY_RECOGNITION + POST_NOTIFICATIONS
    → Si permisos OK:
        Intent(context, StepCounterService::class.java)
        context.startForegroundService(serviceIntent)   ← Android 8+

[2] StepCounterService.onCreate():
    → registerListener(this, STEP_COUNTER, SENSOR_DELAY_NORMAL)
    → startForeground(NOTIFICATION_ID, buildNotification(steps))
      buildNotification():
        → PendingIntent(MainActivity) para tap en notificación
        → NotificationCompat.Builder con setOngoing(true) → no se puede descartar

[3] StepCounterService.onSensorChanged(event):
    baseValue = prefs.getLastSensorValue()
    if (baseValue == -1f) baseValue = event.values[0]   ← primer uso del día
    stepsToday = (event.values[0] - baseValue).toInt()
    // Reset a medianoche:
    if (LocalDate.now().toString() != prefs.getLastResetDate()) {
        baseValue = event.values[0]
        prefs.saveLastResetDate(today)
    }
    prefs.saveStepsToday(stepsToday)     ← escribe en step_counter_prefs
    updateNotification(stepsToday)

[4] StepCounterDataSource.getStepsFlow():
    callbackFlow {
        trySend(prefs.getInt("steps_today", 0))   ← emite valor actual inmediatamente
        prefs.registerOnSharedPreferenceChangeListener { sharedPrefs, key →
            if (key == "steps_today") trySend(sharedPrefs.getInt(key, 0))
        }
        awaitClose { prefs.unregisterOnSharedPreferenceChangeListener(listener) }
    }
    → Cada vez que el Service escribe → el Flow emite → DashboardViewModel recibe

[5] DashboardViewModel.loadDashboardData():
    prepareMockDataIfNeeded()   ← inserta usuario y sesiones mock si BD vacía
    userRepository.getUser().collect { user →
        dataJob?.cancel()
        dataJob = viewModelScope.launch {
            combine(
                rehabRepository.getSessions(userId),      ← Flow Room
                stepCounterDataSource.getStepsFlow(),     ← Flow SharedPrefs
                achievementRepository.getAchievements()   ← Flow Room
            ) { sessions, steps, achievements →
                // Derivaciones:
                caloriesKcal = (5.0 * 70.0 * totalSeconds / 3600).toInt()
                distanceKm = steps * 0.000762f
                activeMinutes = (totalSeconds / 60).toInt()

                // Logros:
                seedAchievementsIfNeeded(achievements)
                checkAndUnlockAchievements(steps, sessions, achievements)

                DashboardUiState(...)
            }.collect { state → _uiState.value = state }
        }
    }

[6] checkAndUnlockAchievements():
    Para cada logro no desbloqueado:
    → Evalúa condición (ej: steps >= 10000)
    → Si se cumple: achievementRepository.unlockAchievement(id)
         → AchievementDao.update(isUnlocked=true, timestamp=now)
    → triggerNewUnlock() → _uiState.newlyUnlockedAchievement = achievement
    → UI muestra popup animado de celebración
    → User dismisses → dismissUnlockPopup() → newlyUnlockedAchievement = null
```

---

## 5. Grafo de Inyección de Dependencias (Hilt — SingletonComponent)

Todas las dependencias se proveen como `@Singleton` en `AppModule`. El ciclo de vida es el de la `Application` (mientras la app vive).

```
Application
  └── AppModule (@Singleton)
       ├── AppDatabase (Room singleton)
       │    ├── UserDao
       │    ├── ClinicsDao
       │    ├── SessionDao
       │    ├── ExerciseDao
       │    └── AchievementDao
       │         └── AchievementRepository → AchievementRepositoryImpl
       ├── Retrofit (singleton) → RoutingApi → RoutingRepository → GetRouteUseCase
       ├── FirebaseAuth (singleton)
       ├── SessionPreferences (DataStore)
       ├── StepCounterDataSource (SharedPreferences)
       ├── LightSensorDataSource (SensorManager)
       ├── AccelerometerDataSource (SensorManager)
       ├── LocationServicePort → FusedLocationProviderClient
       ├── CameraSessionPort → CameraXSessionAdapter
       ├── UserRepository → UserRepositoryImpl → UserDao
       ├── RehabRepository → RehabRepositoryImpl → SessionDao + ExerciseDao
       └── ClinicsRepositoryPort → ClinicsRepositoryImpl → ClinicsDao
```

Los `@HiltViewModel` reciben sus dependencias automáticamente cuando Compose invoca `hiltViewModel()`. Cada ViewModel tiene scope `ViewModelComponent` (vive mientras el NavBackStackEntry correspondiente está en el back stack).
