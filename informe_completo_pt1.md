# Informe Técnico Exhaustivo — GambApp
## Programación Móvil III · Trabajo Práctico Integrador

---

## Sección 1: Descripción General del Proyecto

**GambApp** es una aplicación nativa Android desarrollada en **Kotlin** con **Jetpack Compose** para asistir a pacientes en su proceso de **rehabilitación física domiciliaria**. El sistema actúa como un asistente kinesiológico inteligente que, a través de sensores del dispositivo y visión computacional local, analiza y guía al usuario durante ejercicios terapéuticos, registra el progreso de su rango de movimiento articular (ROM), y conecta al paciente con centros médicos cercanos mediante geolocalización.

### Propósito Principal
Reducir la tasa de abandono de la rehabilitación domiciliaria (estimada en ~70%) proveyendo:
- Detección de postura mediante ML Kit (sin envío de datos a la nube).
- Feedback visual y semántico del ángulo articular en tiempo real.
- Registro histórico de sesiones y progreso de ROM.
- Sistema de logros para motivar la constancia terapéutica.
- Alertas de seguridad ante caídas (acelerómetro).
- Geolocalización de clínicas kinesiológicas cercanas.

---

## Sección 2: Cumplimiento de la Consigna

La consigna del TP exige implementar lo siguiente. Se verifica el cumplimiento:

| Requisito | Implementación en el Proyecto | Estado |
| :--- | :--- | :---: |
| **Al menos un sensor de movimiento** | `AccelerometerDataSource.kt` — Sensor `TYPE_ACCELEROMETER`, detección de caídas por magnitud vectorial ≥ 24.5 m/s² | ✅ |
| **Al menos un sensor ambiental** | `LightSensorDataSource.kt` — Sensor `TYPE_LIGHT`, medición en tiempo real de lux. `EnvironmentCheckViewModel` clasifica el nivel en `GOOD/FAIR/POOR` | ✅ |
| **Sensor de posición / podómetro** | `StepCounterDataSource.kt` + `StepCounterService.kt` — `STEP_COUNTER` hardware con `SharedPreferences` para persistencia diaria y reseteo automático a medianoche | ✅ |
| **Al menos una cámara** | `CameraX` vía `CameraSessionPort` + `PoseDetectionDataSource.kt` usando ML Kit. Cámara frontal con preview y análisis de frames | ✅ |
| **Al menos una animación** | Lottie en `SplashScreen.kt`, animaciones de Compose (`AnimatedVisibility`, `animateColorAsState` en `SkeletonOverlay`) | ✅ |
| **Un mapa** | `MapScreen.kt` con MapTiler SDK (`MTMapViewController`) — Mapa vectorial con clusters de clínicas | ✅ |
| **Geolocalización** | `ObserverLocationUseCase.kt` con FusedLocationProviderClient de Google Play Services + cálculo de clínicas cercanas por distancia haversine | ✅ |
| **Cobertura > 80%** | 35 archivos de tests unitarios con MockK + Turbine. Task `koverXmlReportRelease` configurada en Gradle | ✅ |
| **Gitflow** | Ramas `main`, `develop`, `feature/*`, `hotfix/*`, `fix/*`, `docs/*`, `chore/*` | ✅ |
| **Arquitectura y buenas prácticas de Móvil II-III** | MVVM, Clean Architecture (con mezcla Hexagonal), Hilt DI, Coroutines + StateFlow, Room | ⚠️ Parcial |

---

## Sección 3: Arquitectura General del Sistema

### Patrón: MVVM + Clean Architecture (con elementos de Arquitectura Hexagonal)

```
UI Layer (Jetpack Compose)
    └── Composables (Screens)
    └── ViewModels (@HiltViewModel)
         │
Domain Layer
    └── Models (data classes puras)
    └── Repository Interfaces
    └── Use Cases (domain/usecase)
         │
Application Layer (Hexagonal)
    └── Ports (port.in / port.out)
    └── Use Cases (application/usecases)
    └── Services (application/service)
         │
Data Layer
    └── Repository Implementations
    └── DataSources (sensor, camera, local, network, location)
    └── Room Database (DAOs + Entities)
    └── Retrofit (routing API)
```

### Stack Tecnológico Completo

| Categoría | Tecnología | Versión |
| :--- | :--- | :--- |
| Lenguaje | Kotlin | 2.0.21 |
| UI | Jetpack Compose + Material3 | BOM 2025.04.01 |
| Navegación | Navigation Compose | 2.8.9 |
| DI | Hilt (Dagger) | 2.56.1 |
| BD Local | Room | 2.7.0-alpha11 ⚠️ |
| Cámara | CameraX | 1.4.2 |
| ML/IA | ML Kit Pose Detection | 18.0.0 |
| Sensores | SensorManager (Android SDK) | — |
| Mapas | MapTiler SDK + Google Play Location | — |
| API Rutas | GraphHopper (Retrofit2 + Gson) | — |
| Animaciones | Lottie Compose | 6.6.6 |
| Auth Backend | Firebase Auth | BOM 33.14.0 |
| Testing | MockK + Turbine + Kover | — |
| Calidad | Ktlint + Android Lint | — |

---

## Sección 4: Módulos del Sistema y Responsabilidades por Desarrollador

---

### DEV 1 — Auth & Sensores Base

**Pantallas a cargo**: Login, Registro, Perfil, Fotómetro (EnvironmentCheck), Splash/Onboarding (3 slides)

#### 4.1.1 Sistema de Autenticación (Login + Registro)

**Arquitectura del flujo de Login:**

```
LoginScreen.kt (UI Composable)
    └── LoginViewModel.kt (MVVM ViewModel)
         └── LoginUseCase.kt (Application Use Case)
              └── FirebaseAuth / UserRepository
```

**`LoginViewModel.kt`** — Patrón de doble estado (`formState` + `uiState`):
- `_formState: MutableStateFlow<LoginFormState>` — Gestiona el texto de los campos, errores de validación en línea y visibilidad de contraseña.
- `_uiState: MutableStateFlow<LoginUiState>` — Sealed interface con estados: `Idle`, `Loading`, `Success`, `Error(UiText)`.
- **Validación reactiva**: Los métodos `onEmailChange` y `onPasswordChange` validan en tiempo real con Regex de email y longitud mínima de contraseña. Los errores se muestran en la UI debajo del campo afectado.
- **`runCatching`**: Patrón usado para capturar errores del caso de uso sin `try/catch` explícito, mapeándolos a `LoginUiState.Error`.

**`UiText`**: Clase sellada que representa textos de UI de forma agnóstica al contexto. Puede ser un `StringResource(R.string.*)` o un `DynamicString(String)`. Permite que el ViewModel defina mensajes de error sin requerir `Context`.

**Decisión de diseño:** Se separó `formState` de `uiState` porque el formulario tiene un ciclo de vida distinto al resultado de la operación de login. El formulario debe persistir y mostrar errores de validación sin reiniciarse cuando llega un error de red.

#### 4.1.2 LightSensorDataSource (Sensor Ambiental — Fotómetro)

**`LightSensorDataSource.kt`**:
```kotlin
fun getLuxFlow(): Flow<Float> = callbackFlow {
    sensorManager.registerListener(listener, lightSensor, SENSOR_DELAY_UI)
    awaitClose { sensorManager.unregisterListener(listener) }
}
```

**¿Por qué `callbackFlow`?** Los sensores de Android usan un patrón de callback `SensorEventListener`. El operador `callbackFlow` de Kotlin Coroutines permite "bridgear" (puente) este callback imperativo hacia un `Flow` reactivo. `awaitClose` garantiza que el listener se desregistre automáticamente cuando el consumidor del Flow deja de recolectar (ej. al salir de la pantalla).

**`EnvironmentCheckViewModel.kt`** — Clasifica el nivel de lux:
- `>= 300 lux` → `GOOD` (iluminación ideal)
- `100–299 lux` → `FAIR` (aceptable pero imperfecta)
- `< 100 lux` → `POOR` (puede causar fallas en ML Kit)

**⚠️ Bug de integración (activo):** `EnvironmentCheckScreen.kt` inyecta `RoutineListViewModel` en lugar de `EnvironmentCheckViewModel`. Todo este sensor y la lógica de clasificación están implementados pero no se visualizan en producción. El ViewModel correcto existe y funciona.

#### 4.1.3 AccelerometerDataSource (Sensor de Movimiento)

**`AccelerometerDataSource.kt`** — Detección de caídas:
- Registra el sensor `TYPE_ACCELEROMETER` con `SENSOR_DELAY_GAME` (mayor frecuencia).
- Calcula la magnitud vectorial resultante: `magnitude = sqrt(x² + y² + z²)`.
- Si `magnitude > 24.5 m/s²` (≈2.5G), se clasifica como caída.
- El threshold de 24.5 m/s² está basado en estudios de biomecánica donde impactos superiores a 2G indican caída libre o impacto violento.

**¿Por qué `SENSOR_DELAY_GAME` y no `SENSOR_DELAY_NORMAL`?** Para detectar caídas se requiere alta frecuencia de muestreo. `SENSOR_DELAY_GAME` muestrea ~50 veces por segundo versus ~5 veces por segundo de `SENSOR_DELAY_NORMAL`, permitiendo capturar picos de aceleración transitorios (que duran milisegundos) antes de que desaparezcan.

---

### DEV 2 — Cámara & Sesión de Rehabilitación

**Pantallas a cargo**: Sesión Rehab (espejo), Resumen Post-Sesión

#### 4.2.1 Pipeline de Cámara con CameraX

**Arquitectura del pipeline:**
```
CameraX Preview (SurfaceProvider)
    └── ImageAnalysis (YUV frames)
         └── PoseDetectionDataSource.kt
              └── ML Kit PoseDetector
                   └── Flow<PoseResult>
                        └── RehabSessionViewModel.kt
```

**`CameraSessionPort`** (interface en `domain/ports`):
```kotlin
interface CameraSessionPort {
    fun start(lifecycleOwner: LifecycleOwner, surfaceProvider: Preview.SurfaceProvider, analyzer: ImageAnalysis.Analyzer)
    fun stop()
}
```

La implementación concreta `CameraXSessionImpl` se inyecta vía Hilt. La separación por interfaz permite testear el ViewModel sin inicializar la cámara real.

**`PoseDetectionDataSource.kt`** implementa `ImageAnalysis.Analyzer`:
- Recibe frames en formato `ImageProxy` (YUV_420_888).
- Crea un `InputImage.fromMediaImage(mediaImage, rotation)` para ML Kit.
- ML Kit detecta 33 `PoseLandmark` (puntos articulares del cuerpo humano).
- El resultado se emite como `Flow<PoseResult>` via `MutableStateFlow`.

**¿Por qué el uso de `LENS_FACING_FRONT`?** Los ejercicios de rehabilitación requieren que el usuario se vea a sí mismo (como un espejo). La cámara frontal es más natural para la autoevaluación postural. La coordenada X de los landmarks se invierte en `SkeletonOverlay.kt` con `mirroredX = logicalWidth - rawX` para compensar el espejado.

#### 4.2.2 CalculateJointAngleUseCase

Calcula el ángulo de una articulación usando tres puntos (proximal, articulación central, distal):

```kotlin
val radians = atan2(lastY - midY, lastX - midX) - atan2(firstY - midY, firstX - midX)
var angle = abs(radians * 180.0 / Math.PI).toFloat()
if (angle > 180f) angle = 360f - angle
```

**¿Por qué `atan2` y no `acos`?** `atan2(dy, dx)` es numéricamente estable para todos los ángulos (0°-360°). `acos` del producto punto puede sufrir errores de precisión de punto flotante cerca de 0° y 180°. La normalización `if (angle > 180f)` garantiza que el resultado siempre esté en el rango [0°, 180°] esperado para articulaciones del cuerpo humano.

#### 4.2.3 SyncMotorUseCase y Conteo de Repeticiones

**`SyncMotorUseCase.execute()`** — Retroalimentación de precisión:
- Compara el ángulo actual con el `endAngle` (ángulo objetivo del ejercicio).
- Si `|currentAngle - endAngle| <= toleranceIdeal` → `IDEAL`
- Si `|currentAngle - endAngle| <= toleranceWarning` → `WARNING`
- Si supera ambos umbrales → `ERROR`

**`SyncMotorUseCase.checkRepetition()`** — Máquina de estados para contar reps:
```
atStart (wasAtTarget=false)
    → user moves to endAngle
atTarget (wasAtTarget=true)
    → user returns to startAngle
completed = true (rep counted)
```
Esta máquina de estados de dos fases evita el "rebote" (counting noise): una repetición solo se cuenta si el usuario alcanzó efectivamente el ángulo objetivo antes de regresar al inicio.

#### 4.2.4 SkeletonOverlay — Visualización del Esqueleto

**`SkeletonOverlay.kt`** dibuja en un `Canvas` de Compose:
1. **Escala aspect-fill**: `scale = max(canvasWidth / logicalWidth, canvasHeight / logicalHeight)` para que el esqueleto llene la pantalla sin deformarse.
2. **Compensación de rotación de imagen**: ML Kit con `InputImage.fromMediaImage` entrega coordenadas en espacio "upright". Se intercambian width/height si `rotation == 90 || 270`.
3. **Color semántico animado**: `animateColorAsState(targetValue = colorFeedback)` aplica una transición suave entre `EmeraldIdeal`, `AmberWarning` y `CoralDanger` según el estado de precisión.
4. **Conexiones corporales hardcodeadas**: Lista de 17 pares `(landmarkA, landmarkB)` que representan el esqueleto humano estándar del modelo de 33 puntos de ML Kit.
