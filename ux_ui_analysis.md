# Análisis de Experiencia de Usuario (UX) e Interfaz (UI) en Android

Este informe evalúa el diseño de interacción, la accesibilidad (A11y), el comportamiento inmersivo y la estética visual de la aplicación móvil de rehabilitación. El análisis está estructurado bajo las cuatro categorías solicitadas: **Falta Agregar**, **Mejoraría**, **Cambiaría** y **Eliminaría**.

---

## 1. Qué Falta Agregar (Nuevas Características y Feedback UI)

### A. Indicador de Nivel de Luz en Tiempo Real (`EnvironmentCheckScreen`)
- **El Problema**: El usuario ve una lista estática de instrucciones de iluminación, pero la aplicación no le confirma si el lugar donde está parado es óptimo.
- **Solución UX**: Agregar una tarjeta dinámica con un medidor analógico (Gauge) o un semáforo de tres estados (`GOOD` / `FAIR` / `POOR`) alimentado por `EnvironmentCheckViewModel`.
- **Ejemplo Visual**:
  - `GOOD` (lux >= 300): Icono de check verde y texto: *“¡Excelente iluminación! Listo para iniciar.”*
  - `FAIR` (lux 100-299): Icono de alerta amarillo y texto: *“Iluminación regular. Recomendamos encender otra luz.”*
  - `POOR` (lux < 100): Icono de cruz roja y texto: *“Muy oscuro. ML Kit podría no detectar tus movimientos.”*

### B. Interfaz de Alerta de Caídas (Seguridad Crítica)
- **El Problema**: La lógica del acelerómetro detecta caídas (`_fallDetected`), pero no existe un componente en la pantalla `RehabSessionScreen` para responder ante este evento de emergencia.
- **Solución UX**: Diseñar un overlay modal de pantalla completa y alta visibilidad que se active inmediatamente cuando `fallDetected == true`.
- **Estructura del Overlay**:
  - Título rojo y parpadeante: **“¿Te encuentras bien?”**
  - Un contador de cuenta regresiva audible (TTS) de 15 segundos para cancelar en caso de falso positivo.
  - Botón de gran tamaño de descarte: *“Estoy bien, continuar ejercicio.”*
  - Al expirar el temporizador, disparar llamada de auxilio o enviar SMS con geolocalización.

### C. Feedback Auditivo y Text-To-Speech (TTS)
- **El Problema**: El usuario debe colocar el celular a **2 metros de distancia** para que la cámara capte su cuerpo completo. A esa distancia, leer los números pequeños de repeticiones o los grados de flexión en pantalla es imposible.
- **Solución UX**: Integrar soporte TTS en la sesión para dar retroalimentación de voz:
  - *"¡Buena flexión! Vas 5 repeticiones."*
  - *"Estira más el brazo derecho."*
  - Sonidos de chime agudos y estimulantes al completar cada repetición correcta.

---

## 2. Qué Mejoraría (Optimización de Componentes y Accesibilidad)

### A. Corrección de Contraste en Modo Claro (WCAG Compliance)
- **El Problema**: El tono `CyanWave` (`#06B6D4`) se utiliza como color secundario. Al colocarse sobre fondos claros (blanco o gris claro), su ratio de contraste es de **1.8:1**, lo cual viola gravemente el estándar WCAG AA (requiere mínimo **4.5:1** para texto normal y **3:1** para elementos de interfaz de usuario).
- **Solución UI**: Modificar la paleta del modo claro en `Color.kt`. Utilizar un tono cian o azul más oscuro (ejemplo: `#0891B2` o `#0369A1`) para asegurar la legibilidad a usuarios con daltonismo o fatiga visual.

### B. Visualización Selectiva de Esqueleto en `SkeletonOverlay.kt`
- **El Problema**: Actualmente se dibujan y colorean **todos los puntos y conexiones corporales** (cara, ojos, tronco, extremidades no usadas) del esqueleto según la precisión de la repetición.
- **Solución UI**:
  - Ocultar las conexiones faciales (`LEFT_EYE`, `RIGHT_EAR`, etc.) durante los ejercicios de extremidades para evitar ruido visual en el centro del encuadre.
  - Resaltar en un tamaño mayor y con brillo los 3 puntos articulares específicos del ejercicio (ej. cadera, rodilla y tobillo para flexión de rodilla) mientras que los demás huesos del cuerpo se dibujan en una línea delgada gris y semitransparente (`Color.Gray.copy(alpha = 0.3f)`).

### C. Evitar Desplazamientos Bruscos de Layout (Navigation Shift)
- **El Problema**: En `MainScreen.kt`, la barra inferior de navegación `BottomBar` se oculta cuando el mapa está cargando (`isMapLoading`). Esto causa un "salto" de interfaz molesto (Layout Shift) cuando la pantalla pasa del estado de carga al mapa renderizado.
- **Solución UX**: Mantener la barra inferior de navegación siempre estática e inamovible. Si el mapa está cargando, mostrar el loader de Lottie exclusivamente dentro del contenedor del mapa sin colapsar la barra de navegación del sistema.

---

## 3. Qué Cambiaría (Patrones de Diseño y Estándares de Android)

### A. Flujo de Inicio (SplashScreen como Entrada)
- **El Problema**: El inicio de la aplicación está forzado directamente hacia el Dashboard. Esto salta las pantallas de Splash y Onboarding, dejando al usuario final desorientado si es su primer inicio de sesión.
- **Solución UX**: Redirigir el `startDestination` del NavHost hacia `Screen.Splash.route` y permitir que el ViewModel decida de forma transparente e imperceptible si salta al Dashboard o a la presentación inicial.

### B. Migrar a Navigation Safe-Type
- **El Problema**: Las transiciones entre pantallas usan rutas en formato String directo (ej. `"rehab_session/{exerciseId}"`), lo que incrementa la probabilidad de errores tipográficos y dificulta la mutabilidad de parámetros complejos.
- **Solución UI**: Adoptar el sistema de navegación tipada nativo de **Jetpack Navigation 2.8+** a través de la serialización de Kotlin (`@Serializable`).

---

## 4. Qué Eliminaría (Eliminación de Elementos Obstructivos y Código Residual)

### A. Ocultamiento de la Barra del Sistema (System Bar Hide)
- **El Problema**: En `MainActivity.kt`, se fuerzan las directivas de ventana para ocultar totalmente la barra de estado y la barra de navegación. Esto priva al usuario de ver la hora, el nivel de batería de su dispositivo o notificaciones urgentes del sistema mientras hace rehabilitación.
- **Solución UX**: Eliminar el método `windowInsetsController.hide(systemBars())`. En su lugar, utilizar de forma limpia la API `enableEdgeToEdge()` de Compose, configurando barras transparentes que permitan que el fondo fluya detrás de ellas de forma elegante sin interferir en la lectura del contenido.

### B. Eliminación de Componentes de Plantilla y Rutas Huérfanas
- **El Problema**: Hay componentes redundantes de plantillas genéricas de Android Studio (`Greeting.kt`) y rutas vacías en `Navigation.kt` (`Screen.User`, `Screen.ClinicDetail`).
- **Solución**: Limpiar el repositorio eliminándolos para agilizar la comprensión del árbol de navegación por nuevos desarrolladores.
