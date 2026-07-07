# Análisis de Deuda Técnica, Principios SOLID y Guía de Estándares de Código

Este informe recopila los principales puntos de deuda técnica acumulada en la base de código del proyecto, expone las inconsistencias de programación detectadas, analiza los principios **SOLID** y las **Mejores Prácticas de Android** que no se están cumpliendo, e identifica las coordenadas exactas de los archivos donde debe ejecutarse la refactorización.

---

## 1. Análisis de Deuda Técnica (Tech Debt Ledger)

A continuación se identifican los mayores focos de deuda técnica estructural, clasificados según su tipo y el esfuerzo requerido para su resolución.

### A. Deuda Arquitectónica: Duplicidad de Capa de Negocio
- **Qué es**: Mezcla del patrón clásico de tres capas (UI-Domain-Data) con Arquitectura Hexagonal basada en puertos (`port.out`, `port.in`).
- **Consecuencia**: Complejidad innecesaria en la inyección de dependencias (`AppModule.kt`), donde algunas interfaces se inyectan como repositorios y otras como puertos.
- **Esfuerzo de corrección**: Alto. Se debe elegir un único patrón (ej. Clean Architecture convencional, que es el estándar de Google para Android) y migrar los puertos de `application` a interfaces de repositorios en `domain/repository`.

### B. Deuda de Control de Concurrencia: Recolecciones de Flow Fugas (Leaks)
- **Qué es**: Lanzamiento de coroutines imperativas que recolectan Flows reactivos de base de datos sin un mecanismo de cancelación o reemplazo (ej. `RehabSessionViewModel.kt` y `PostSessionViewModel.kt`).
- **Consecuencia**: Posibles fugas de memoria y llamadas concurrentes duplicadas al mismo recurso de datos si se re-evalúan parámetros.
- **Esfuerzo de corrección**: Medio. Se deben refactorear las suscripciones manuales mediante operadores declarativos como `flatMapLatest` o vinculando el ciclo de vida del flujo al `viewModelScope` a través de `stateIn`.

### C. Deuda de Diseño del Ciclo de Vida: Inicialización de Datos en ViewModels
- **Qué es**: La base de datos local (Room) es sembrada con datos mock de ejercicios y logros directamente en los constructores/métodos `init` de `RoutineListViewModel` y `DashboardViewModel`.
- **Consecuencia**: Si el usuario no ingresa a esas pantallas específicas al iniciar la app, la base de datos permanecerá vacía, afectando otros flujos de datos. Adicionalmente, el ViewModel asume responsabilidades de control de datos que no le corresponden.
- **Esfuerzo de corrección**: Bajo/Medio. La siembra inicial debe ocurrir a nivel de base de datos en `RoomDatabase.Callback` o gestionarse en un repositorio de inicio de la aplicación en el arranque.

---

## 2. Incumplimiento de Principios SOLID

El análisis del código reveló violaciones directas a tres de los principios fundamentales de diseño orientado a objetos y clean code:

### A. Principio de Responsabilidad Única (SRP - Single Responsibility Principle)
*   **Violación en [DashboardViewModel.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/DashboardViewModel.kt#L98-L107) y [RoutineListViewModel.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/RoutineListViewModel.kt#L71-L131)**:
    Ambos ViewModels, además de gestionar el estado de visualización de la UI (`UiState`), asumen el rol de administradores de persistencia, coordinando la inserción inicial (seeding) de logros y ejercicios de prueba.
*   **Refactorización**:
    Separar la persistencia inicial y colocarla bajo una clase dedicada o callback del constructor de base de datos Room. Delegar la validación y desbloqueo de logros a un caso de uso independiente (ej. `CheckAchievementsUseCase`), manteniendo el ViewModel enfocado únicamente en reaccionar al cambio de estados y formatear datos para la interfaz.

### B. Principio de Abierto/Cerrado (OCP - Open/Closed Principle)
*   **Violación en [RehabSessionViewModel.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/RehabSessionViewModel.kt#L165-L180)**:
    El método `getLandmarkTypeByName(name: String)` mapea nombres en texto a enteros internos de ML Kit mediante un bloque estático `when`:
    ```kotlin
    private fun getLandmarkTypeByName(name: String): Int = when (name) { ... }
    ```
    Si en el futuro deseamos añadir un ejercicio terapéutico que involucre nuevas articulaciones corporales (como el tobillo, pie o cuello), este ViewModel deberá modificarse directamente para agregar los nuevos casos al `when`. Esto rompe OCP (abierto para extensión, cerrado para modificación).
*   **Refactorización**:
    Mapear estos valores mediante un enumerador de dominio (`JointType`) o asociar las constantes directamente al modelo de datos `Exercise`, abstrayendo la lógica del framework del ViewModel.

### C. Principio de Sustitución de Liskov (LSP - Liskov Substitution Principle)
*   **Violación en [ClinicsRepositoryImpl.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/data/repositories/ClinicsRepositoryImpl.kt#L34-L42)**:
    La clase `ClinicsRepositoryImpl` implementa el puerto `ClinicsRepositoryPort`, el cual establece los métodos de escritura y eliminación de clínicas. Sin embargo, dichos métodos están implementados vacíos (stubs) con el comentario *"currently no delete/update method in DAO"*.
    Si un componente cliente asume que al invocar `deleteClinic()` se borrará el elemento, el contrato se rompe silenciosamente, rompiendo LSP.
*   **Refactorización**:
    Si la base de datos de clínicas es de solo lectura, se debe eliminar la definición de métodos de actualización e inserción de la interfaz del repositorio, o bien agregar soporte real a nivel DAO.

---

## 3. Mejores Prácticas de Android No Cumplidas

### A. Acoplamiento de SDK de Presentación en ViewModels (Violación de Arquitectura MVVM)
*   **El Problema**:
    En [MapScreenViewModel.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/MapScreenViewModel.kt#L29-L30), el ViewModel importa e inicializa de forma directa clases controladoras de vista específicas de MapTiler (`MTMapViewController`, `MTMapViewDelegate`, `MTCameraOptions`).
    Los ViewModels deben ser puros e independientes de los componentes de vista del sistema Android. Instanciar controladores que requieren contextos de UI o Canvas dentro del ViewModel impide escribir pruebas unitarias JVM estándar y provoca pérdidas de memoria si no se destruyen debidamente.
*   **Refactorización**:
    Toda la lógica de control del mapa y suscripciones a delegados debe pertenecer a la capa de UI (dentro del Composable `MapScreen.kt` o mediante un estado persistido con `rememberMapState`). El ViewModel solo debe exponer flujos puros de coordenadas, listas de clínicas y estados primitivos.

### B. Práctica Ineficiente de Consumo de Flujos (Flows)
*   **El Problema**:
    El uso de flujos anidados en ViewModels (como en `PostSessionViewModel.kt` y `RehabSessionViewModel.kt`) mediante llamadas encadenadas o colecciones dentro de coroutines lanzadas interrumpe el flujo reactivo y dificulta la cancelación de trabajos antiguos.
*   **Refactorización**:
    Transformar los flujos usando operadores declarativos (`flatMapLatest`, `map`, `combine`) y exponer el resultado final directamente a la vista a través de `stateIn()`.

---

## 4. Inconsistencias de Código (Estándares no Cumplidos)

Para asegurar que todo el equipo codee bajo un mismo patrón, se describen las inconsistencias reales encontradas en el código y el estándar que debe aplicarse de ahora en adelante.

### A. Estándar de Nomenclatura (Naming Conventions)
*   **Inconsistencia 1**:
    En `Navigation.kt`, `object MapScreen : Screen("MapScreen")` utiliza CamelCase, mientras que todos los demás destinos definen su ruta en minúsculas y snake_case (`"routine_list"`, `"onboarding"`, `"post_session"`).
*   **Inconsistencia 2**:
    Clase `UpdateClinicUseCAse` (con `CAse` en mayúsculas) en contraste con la clase `DeleteClinicUseCase` (formato CamelCase correcto).
*   **Regla Estándar**:
    *   Todas las rutas de navegación definidas como Strings deben escribirse en **minúsculas y snake_case** (ej. `"map_screen"`).
    *   Todos los nombres de archivo y clases deben seguir estrictamente el estándar **UpperCamelCase** de Kotlin (ej. `UpdateClinicUseCase`).

### B. Declaración de Pantallas y Clases Singulares
*   **Inconsistencia**:
    En `Navigation.kt`, las pantallas están declaradas utilizando la palabra clave `object`:
    `object Onboarding : Screen("onboarding")`
*   **Regla Estándar**:
    Desde Kotlin 1.9+, para cualquier objeto singleton que represente un estado o destino sellado, se debe declarar como **`data object`**. Esto asegura la implementación correcta de `toString()`, `equals()` y `hashCode()`, facilitando la depuración del árbol de navegación en los logs.
    *   *Incorrecto*: `object Login : Screen("login")`
    *   *Correcto*: `data object Login : Screen("login")`

### C. Consumo de Colores en el Diseño de UI (Jetpack Compose)
*   **Inconsistencia**:
    En pantallas como `EnvironmentCheckScreen.kt` y `PostSessionScreen.kt`, se consumen los colores de la marca de forma directa utilizando constantes hardcodeadas de `Color.kt` en los modificadores y botones:
    `colors = ButtonDefaults.buttonColors(containerColor = ElectricIndigo)`
*   **Regla Estándar**:
    **Nunca** se deben inyectar constantes de color crudas (como `ElectricIndigo`) en los composables de UI. Toda resolución cromática debe hacerse consultando el esquema de colores del tema (`MaterialTheme.colorScheme`), permitiendo el soporte nativo a Light/Dark Mode y colores dinámicos (Material You).
    *   *Incorrecto*: `color = ElectricIndigo`
    *   *Correcto*: `color = MaterialTheme.colorScheme.primary`

### D. Declaración de Constructor de Casos de Uso
*   **Inconsistencia**:
    En `SyncMotorUseCase.kt` y `CalculateJointAngleUseCase.kt`, la estructura de llaves coloca los métodos miembros dentro del bloque de instanciación del constructor:
    ```kotlin
    class SyncMotorUseCase
        @Inject
        constructor() {
            fun execute(...) { ... }
        }
    ```
*   **Regla Estándar**:
    Los métodos miembros deben declararse dentro del cuerpo de la clase y no indentados bajo el constructor. La sintaxis del constructor debe ser lineal:
    ```kotlin
    class SyncMotorUseCase @Inject constructor() {
        fun execute(...) { ... }
    }
    ```

---

## 5. Dónde y Cómo Refactorizar (Coordenadas de Código)

Para facilitar la asignación de estas tareas al equipo, se listan los archivos y bloques de código específicos a modificar:

1.  **Refactorizar Control de MapTiler**:
    *   **Archivo**: [MapScreenViewModel.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/MapScreenViewModel.kt)
    *   **Líneas**: [L102-L119](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/MapScreenViewModel.kt#L102-L119) y [L329-L368](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/MapScreenViewModel.kt#L329-L368)
    *   **Acción**: Extraer la instancia `mapController` y el método `setupClusters()` hacia la capa de UI. El ViewModel solo debe exponer el estado de las clínicas y recibir eventos.
2.  **Corregir Fugas de Coroutines**:
    *   **Archivo**: [RehabSessionViewModel.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/RehabSessionViewModel.kt)
    *   **Líneas**: [L157-L163](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/RehabSessionViewModel.kt#L157-L163)
    *   **Acción**: Agregar un puntero `Job?` a nivel de clase, cancelarlo antes de lanzar el scope coroutine y evitar colecciones acumulativas.
3.  **Unificar el Flujo de Datos Terapéutico**:
    *   **Archivo**: [PostSessionViewModel.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/PostSessionViewModel.kt)
    *   **Líneas**: [L28-L40](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/PostSessionViewModel.kt#L28-L40)
    *   **Acción**: Reemplazar los bloques anidados de `collectLatest` por una llamada limpia combinando flujos mediante el operador `.flatMapLatest`.
4.  **Limpiar Métodos Inactivos de Repositorio (LSP)**:
    *   **Archivo**: [ClinicsRepositoryImpl.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/data/repositories/ClinicsRepositoryImpl.kt)
    *   **Líneas**: [L34-L42](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/data/repositories/ClinicsRepositoryImpl.kt#L34-L42)
    *   **Acción**: Remover métodos de inserción/edición vacíos o implementarlos de forma completa y reactiva a nivel DAO.
5.  **Extraer Siembra de Base de Datos (SRP)**:
    *   **Archivo**: [RoutineListViewModel.kt](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/RoutineListViewModel.kt)
    *   **Líneas**: [L71-L131](file:///home/enltd/Escritorio/Programacion/A3-2026-H1-E1/app/src/main/java/ar/edu/unlam/mobile/scaffolding/ui/viewmodels/RoutineListViewModel.kt#L71-L131)
    *   **Acción**: Trasladar el método `prepareMockExercisesIfNeeded` a la clase de inicialización de Room en el hilo IO o callback de creación de BD.
