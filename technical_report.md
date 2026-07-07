# Informe Técnico de Arquitectura y Calidad de Código

Este informe presenta un análisis exhaustivo del repositorio de la aplicación móvil (basada en Kotlin, Jetpack Compose y Android Arquitectura de Capas/Limpia). Se detallan hallazgos críticos de diseño, bugs lógicos, redundancia de código, problemas de concurrencia y se provee una tabla priorizada de recomendaciones de refactorización.

---

## 1. Arquitectura General y Estructura de Paquetes

### Coexistencia de Paradigmas Arquitectónicos (Híbrido Clean vs. Hexagonal)
Se detectó una discrepancia fundamental en la organización estructural del proyecto. Existen dos paquetes principales que intentan modelar la capa de dominio y de aplicación mediante diferentes aproximaciones:
1. **`ar.edu.unlam.mobile.scaffolding.domain`**: Estructura clásica de Clean Architecture. Contiene subpaquetes como `model`, `repository` (interfaces) y `usecase` (ej., `SyncMotorUseCase.kt`, `CalculateJointAngleUseCase.kt`).
2. **`ar.edu.unlam.mobile.scaffolding.application`**: Estructura basada en Arquitectura Hexagonal (Ports & Adapters). Define puertos de entrada/salida como `port.out.local.db.ClinicsRepositoryPort`, casos de uso (`usecases.location.GetClinicsStoredUseCase`), e interactores como `service.remote.routing.GetRouteInteractor`.

#### Impacto en el Proyecto
- **Redundancia conceptual**: Se duplica la abstracción de repositorios. Unas pantallas consumen interfaces de `domain.repository` y otras consumen puertos de `application.port.out`.
- **Falta de consistencia**: Para un desarrollador que se suma al equipo, no queda claro bajo qué criterio estructurar una nueva funcionalidad (si crear un puerto/interactor o un repositorio/caso de uso).
- **Dificultad de escalabilidad**: La mezcla de flujos acopla innecesariamente la nomenclatura y las dependencias de Hilt.

### Separación de Capas e Inyección de Dependencias
- **UI (Jetpack Compose & ViewModels)**: En general, los componentes de interfaz de usuario están desacoplados a través de `UiState` y `StateFlow`. Sin embargo, hay dependencias directas del SDK de MapTiler (`MTMapViewController`) dentro de `MapScreenViewModel.kt`, lo que viola la independencia de la UI frente al framework de mapas en el ViewModel.
- **Datos (Data Layer)**: Los repositorios implementados bajo `data/repositories` implementan correctamente los contratos de la capa superior. Sin embargo, persisten stubs sin implementación y lógica de negocio acoplada en los ViewModels en lugar de los casos de uso.

---

## 2. Calidad de Código y Buenas Prácticas Kotlin

### Violación del Principio de Responsabilidad Única (SRP) y Flujos Complejos
En [DashboardViewModel.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/DashboardViewModel.kt#L98-L107), la lógica de negocio y efectos colaterales de base de datos están incrustados directamente dentro del operador `combine` del flujo reactivo:

```kotlin
val uiState: StateFlow<DashboardUiState> =
    combine(
        userRepository.getUser(),
        rehabRepository.getSessions(userId),
        stepCounterDataSource.getStepsFlow(),
        achievementRepository.getAchievements(),
    ) { user, sessions, steps, achievements ->
        // Efecto secundario directo en BD dentro del mapeo del flujo
        seedAchievementsIfNeeded()
        checkAndUnlockAchievements(steps, sessions, achievements)
        // ...
    }
```
#### Impacto
Dado que `getStepsFlow()` emite un valor en tiempo real con cada paso detectado por el sensor, este bloque se ejecutará de forma repetitiva. Generar inserciones o verificaciones de base de datos de manera síncrona dentro de la combinación del Flow satura el hilo de base de datos y rompe la naturaleza pura del flujo de datos (Unidirectional Data Flow).

### Bloques de Construcción de Casos de Uso con Indentación Anómala
En los archivos [SyncMotorUseCase.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/domain/usecase/SyncMotorUseCase.kt#L13-L54) y [CalculateJointAngleUseCase.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/domain/usecase/CalculateJointAngleUseCase.kt#L7-L45), los métodos miembro están indentados dentro de la llave del constructor primario de la clase.

```kotlin
class SyncMotorUseCase
    @Inject
    constructor() {
        fun execute(...) { ... }
    }
```
Esto genera confusión visual y no respeta el formato estándar de Kotlin/Ktlint, donde los métodos miembros se ubican al nivel del cuerpo de la clase y no anidados bajo la declaración del constructor.

### Código Muerto y Rutas Inalcanzables
- **Casos de Uso Muertos**: [DeleteClinicUseCase.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/application/usecases/location/DeleteClinicUseCase.kt) y [UpdateClinicUseCAse.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/application/usecases/location/UpdateClinicUseCAse.kt) (el cual tiene un error tipográfico en su nombre: `UseCAse`) llaman a `deleteClinic()` del repositorio, que a su vez es una función vacía en `ClinicsRepositoryImpl.kt`. Ninguno de estos archivos es inyectado ni consumido.
- **Rutas Huérfanas**: En [Navigation.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/navigation/Navigation.kt), se definen `Screen.User` y `Screen.ClinicDetail`. No obstante, no están registradas en el `NavHost` de `MainScreen.kt`, ni tienen pantallas asociadas.
- **Componentes no Usados**: [Greeting.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/components/Greeting.kt) es código residual autogenerado de la plantilla del proyecto que no tiene uso alguno.

### Confusión en Nombres de Componentes (Namespace Collision)
Se creó un componente personalizado llamado `LottieAnimation` en [LottieAnimation.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/components/LottieAnimation.kt#L18). Dado que se llama exactamente igual que el composable oficial de Airbnb, se genera colisión de nombres. Esto obliga a realizar imports específicos y evita que se use el wrapper nativo de forma intuitiva.

---

## 3. Gestión de Dependencias

- **Versión de Room inestable**: En `gradle/libs.versions.toml`, la biblioteca de persistencia Room se declara con la versión `2.7.0-alpha11`. Utilizar versiones alfa en producción introduce riesgos de inestabilidad y comportamiento no documentado en la base de datos.
- **Inyección redundante**: `LightSensorDataSource` se declara con constructor convencional pero se provee de manera explícita en `AppModule.kt`. Al no tener dependencias complejas, se podría inyectar directamente con la anotación `@Inject constructor` a nivel de clase, simplificando la clase de módulo de Hilt.

---

## 4. Manejo de Estado, Concurrencia y Ciclo de Vida

### Fuga de Suscripciones en Flujos (Leaks)
En [RehabSessionViewModel.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/RehabSessionViewModel.kt#L157-L163), el método `loadExercise` inicia una recolección directa de un Flow reactivo en cada invocación:

```kotlin
fun loadExercise(exerciseId: String) {
    viewModelScope.launch {
        rehabRepository.getExerciseById(exerciseId).collect { exercise ->
            _currentExercise.value = exercise
        }
    }
}
```
Si este método es invocado múltiples veces (por ejemplo, por cambios de configuración o reintentos), cada llamada lanzará un nuevo coroutine que recopila del mismo Flow sin cancelar el anterior. Esto genera acumulación de suscripciones inactivas en memoria.

### Flujos Imperativos Anidados
En [PostSessionViewModel.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/PostSessionViewModel.kt#L28-L40), se anidan llamadas a `collectLatest` de forma imperativa:

```kotlin
fun loadLastSession(userId: String = "user_imanol") {
    viewModelScope.launch {
        rehabRepository.getSessions(userId).collectLatest { sessions ->
            val session = sessions.firstOrNull()
            _lastSession.value = session
            session?.let {
                rehabRepository.getExerciseById(it.exerciseId).collectLatest { exercise ->
                    _exercise.value = exercise
                }
            }
        }
    }
}
```
Esto puede simplificarse sustancialmente y hacerse más reactivo/declarativo utilizando el operador `flatMapLatest` expuesto por Kotlin Coroutines.

### Ocultamiento de Barras del Sistema (Edge-to-Edge Violations)
En [MainActivity.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/MainActivity.kt#L69-L79), se ocultan permanentemente las barras de estado y navegación del sistema:

```kotlin
windowInsetsController.systemBarsBehavior =
    androidx.core.view.WindowInsetsControllerCompat.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE
windowInsetsController.hide(
    androidx.core.view.WindowInsetsCompat.Type.systemBars(),
)
```
Esto rompe con las guías modernas de Android de diseño inmersivo (Edge-to-Edge), ocultando información esencial (batería, hora, notificaciones) en pantallas que no son reproducciones multimedia a pantalla completa o juegos.

---

## 5. Escalabilidad, Integración Incompleta y Desalineación del UI

### Error de Integración de Sensor (EnvironmentCheckScreen)
El hallazgo de arquitectura más crítico es la desconexión total del sensor de luz en la pantalla de preparación.
- **El problema**: [EnvironmentCheckScreen.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/screens/rehab/EnvironmentCheckScreen.kt#L30) declara e inyecta `RoutineListViewModel` en lugar de su propio `EnvironmentCheckViewModel`.
- **Consecuencia**: La pantalla de preparación de entorno no muestra ningún indicador de iluminación, advertencias ni datos del sensor de luz ambiental. Toda la infraestructura implementada en `LightSensorDataSource` y `EnvironmentCheckViewModel` (incluyendo la clasificación del nivel de luz en `GOOD`, `FAIR` y `POOR`) está completamente desconectada e inaccesible para el usuario.

### Pantalla de Splash Ignorada
En el `NavHost` de [MainScreen.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/navigation/MainScreen.kt#L108), el destino inicial (`startDestination`) está fijado en `Screen.Dashboard.route`. 
Como consecuencia, la pantalla de [SplashScreen.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/screens/SplashScreen.kt) (y su lógica asociada en `SplashViewModel.kt` para evaluar si el onboarding fue completado) es completamente ignorada al iniciar la aplicación.

---

## 6. Testing

### Fortalezas del Suite de Pruebas
- El proyecto posee una cobertura de pruebas unitarias robusta (más de 30 archivos de prueba bajo `src/test/java`).
- Las pruebas de ViewModel hacen uso correcto de las bibliotecas MockK y Turbine para aserciones de flujos asíncronos y coroutines.
- El script local `pr-check.sh` integra y automatiza el formateo, pruebas, reporte de cobertura XML e inspección estática con Android Lint.

### Debilidades
- **Falta de Pruebas de Integración con Sensores**: La pantalla `EnvironmentCheckScreen` no tiene pruebas sobre el comportamiento dinámico del nivel de luz ambiental dado que no implementa su ViewModel correspondiente.

---

## 7. Recomendaciones Priorizadas para el Equipo de Desarrollo

A continuación se detalla el plan de acción sugerido para remediar los hallazgos descritos, ordenados por nivel de impacto y urgencia.

| Prioridad | Componente / Archivo | Hallazgo / Descripción | Impacto / Riesgo | Recomendación y Ejemplo de Refactorización |
| :--- | :--- | :--- | :--- | :--- |
| **Crítico** | `EnvironmentCheckScreen.kt` | Se inyecta `RoutineListViewModel` en lugar de `EnvironmentCheckViewModel`. | Funcionalidad rota. El sensor de luz ambiental no se visualiza en la interfaz. | **Cambiar la inyección** para utilizar `EnvironmentCheckViewModel`. Para mostrar la información del ejercicio a su vez, agregar `rehabRepository` a `EnvironmentCheckViewModel` y obtener el ejercicio por ID de forma reactiva. |
| **Crítico** | `RehabSessionViewModel.kt` | Fuga de coroutines al llamar repetidamente a `collect` en `loadExercise()`. | Fuga de memoria y acumulación de suscripciones activas de coroutines. | **Mantener una referencia al Job actual** y cancelarlo antes de iniciar una nueva recolección: <br><br>```kotlin private var loadExerciseJob: Job? = null<br>fun loadExercise(exerciseId: String) {<br>    loadExerciseJob?.cancel()<br>    loadExerciseJob = viewModelScope.launch {<br>        rehabRepository.getExerciseById(exerciseId).collect { ... }<br>    }<br>}``` |
| **Importante** | `DashboardViewModel.kt` | Acciones de base de datos (`seedAchievementsIfNeeded`, `checkAndUnlockAchievements`) dentro del bloque reactivo `combine`. | Degradación de rendimiento del hilo de UI, violaciones de arquitectura reactiva e inserciones repetitivas e innecesarias. | **Trasladar la lógica a coroutines independientes** disparadas por eventos de cambio específicos (por ejemplo, en respuesta a la emisión del flujo de pasos o mediante un caso de uso específico que controle el progreso), evitando llamadas directas dentro de constructores de flujo. |
| **Importante** | `PostSessionViewModel.kt` | Anidamiento de flujos con `collectLatest` imperativos. | Código complejo, propenso a race conditions y difícil de testear. | **Usar operadores reactivos como `flatMapLatest`** a nivel de flujo expuesto en lugar de lanzar recolecciones anidadas: <br><br>```kotlin val exercise: StateFlow<Exercise?> = lastSession.flatMapLatest { session -><br>    if (session == null) flowOf(null)<br>    else rehabRepository.getExerciseById(session.exerciseId)<br>}.stateIn(viewModelScope, SharingStarted.Lazily, null)``` |
| **Importante** | `MainScreen.kt` | Splash Screen omitida en el inicio (`startDestination`). | El flujo de onboarding se ignora; el usuario accede al Dashboard sin iniciar sesión ni onboarding. | **Fijar `startDestination` a `Screen.Splash.route`** en el `NavHost` para asegurar que se ejecute la verificación del estado de sesión y onboarding inicial. |
| **Normal** | `Navigation.kt` | Uso de strings literales para rutas y objetos planos (`object`). | Rutas propensas a errores tipográficos en tiempo de compilación. Falta de consistencia en nombres de rutas (ej: `MapScreen` vs `onboarding`). | **Migrar a navegación segura (Type-Safe Navigation)** usando la integración nativa de Jetpack Navigation 2.8+ mediante clases anotadas con `@Serializable`. Cambiar `object` por `data object` en la jerarquía sellada. |
| **Normal** | `MainActivity.kt` | Ocultamiento forzado de barras del sistema (`windowInsetsController.hide`). | Mala experiencia de usuario (UX) al ocultar notificaciones, reloj y barra de navegación de forma persistente. | **Remover la línea de ocultamiento** y configurar insets con la API Compose `EdgeToEdge` para que el diseño fluya detrás de las barras con transparencias correctas. |
| **Bajo** | `UpdateClinicUseCAse.kt` | Typo en el nombre de la clase (`UseCAse`) y código huérfano. | Código muerto que degrada la mantenibilidad. | **Eliminar las clases muertas** (`DeleteClinicUseCase.kt` y `UpdateClinicUseCAse.kt`) ya que la base de datos local no cuenta actualmente con métodos de edición o borrado para clínicas. |
| **Bajo** | `LottieAnimation.kt` | Conflicto de nombres con la dependencia de Airbnb. | Dificultad para importar componentes y confusión en el equipo. | **Renombrar el wrapper personalizado** a `LottieLoader` o `InfiniteLottieAnimation` para evitar colisión de nombres y clarificar su propósito. |
