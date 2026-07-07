# GambApp — Sprint Backlog Analysis
### Rol: Senior Mobile Dev · Orientación a producto · UX/UI end-to-end

---

## Diagnóstico del estado actual

Revisé el código existente. Esto es lo que realmente tenemos implementado vs lo que está en el plan:

| Pantalla | Existe en código | Estado real |
|---|---|---|
| SplashScreen | ✅ | Funcional con `SplashViewModel` |
| OnboardingScreen | ✅ | Funcional |
| LoginScreen | ✅ | Funcional con validaciones |
| RegisterScreen | ✅ | Funcional |
| DashboardScreen | ✅ | **1.142 líneas — demasiado grande, necesita refactor** |
| EnvironmentCheckScreen | ✅ | Funcional con fotómetro |
| RehabSessionScreen | ✅ | CameraX + ML Kit implementado |
| PostSessionScreen | ✅ | Funcional |
| MapScreen | ✅ | Google Maps Compose + pines |
| ClinicDetail (bottom sheet) | ✅ | Funcional |
| ProgressScreen | ✅ | Con gráficos |
| AchievementsView | ✅ | Existe — **pero está mezclada en dashboard** |
| ProfileScreen | ✅ | Funcional |
| RoutineListScreen | ✅ | Funcional |

### Problemas críticos detectados en código

> [!CAUTION]
> **DashboardScreen.kt tiene 1142 líneas en un solo archivo.** Esto es un riesgo de mantenimiento y un smell muy claro para un dev senior. Hay componentes (`AchievementsCard`, `RomProgressRing`, `StepsCard`, etc.) que deben vivir en `ui/components/`.

> [!WARNING]
> Los datos de pasos (calorías, distancia, tiempo activo) están **hardcodeados** en `StepsCard`: `"324 kcal"`, `"5.2 km"`, `"45 min"`. Esto rompe la promesa de producto y va a generar un bug report del primer usuario.

> [!WARNING]
> La card de `AchievementsCard` navega a `onNavigateToAchievements`, pero el plan de Imanol dice mover logros a perfil. La ruta `Achievements` existe separada en `Navigation.kt` — hay que consolidar esto.

> [!NOTE]
> El avatar de perfil en `DashboardHeader` tiene un ícono de `Person` estático. Imanol dice eliminar ese acceso al perfil desde home. La tarea está identificada.

---

## Los 4 nuevos ítems de Imanol — análisis honesto

### 1. Google Fit & Smartwatches
**¿Qué hay hoy?** `HealthConnectDataSource` ya existe y lee FC, calorías y SpO2 de Health Connect.
**¿Qué falta?** Health Connect ya es el puente hacia Wear OS y smartwatches en el ecosistema Android moderno. La "vinculación real" implica:
- Verificar que el usuario tenga Health Connect instalado y mostrar onboarding de permisos granulares
- Conectar los datos de Wear OS (FC en tiempo real durante sesión) al `RehabSessionScreen`
- Mostrar en `ProgressScreen` los datos de FC de Health Connect correctamente (hoy puede estar mockeado)

**Estimación honesta:** 3 pts (no es un sprint entero, pero tampoco es trivial)

### 2. Lottie en algunos íconos
**¿Qué hay hoy?** Emojis como `🏋️`, `🚶`, `🔥` hardcodeados. Íconos Material estáticos.
**¿Dónde tiene sentido?** Splash screen logo, achievement unlocked dialog, onboarding slides, estado vacío de progreso.
**Estimación honesta:** 2 pts (agregar dependencia + 3-4 animaciones específicas)

### 3. Fix Home Screen — eliminar acceso a perfil
**¿Qué hay hoy?** En `DashboardHeader` hay un `Icon(Icons.Default.Person)` que no tiene `onClick` activo (no navega). La `AchievementsCard` con `onNavigateToAchievements` navega a la screen separada de logros.
**¿Qué hacer?** Eliminar el avatar/ícono de perfil del header, asegurarse de que la Bottom Nav tenga Perfil como destino accesible (ya lo tiene en `MainScreen`).
**Estimación honesta:** 1 pt — es una tarea chica pero importante para la coherencia de IA.

### 4. Mover Achievements a Profile Screen
**¿Qué hay hoy?** `AchievementsView.kt` existe en `ui/screens/dashboard/`, `Achievements` es una `Screen` separada en navigation, y hay una `AchievementsCard` en Dashboard.
**¿Qué hacer?** Integrar `AchievementsView` dentro de `ProfileScreen` (sección expandible o tab), quitar la route `Achievements` standalone, remapear la card del dashboard a un preview compacto que navega a Profile.
**Estimación honesta:** 2 pts

---

## Deuda técnica identificada (no estaba en el plan original)

| # | Problema | Prioridad | Tipo |
|---|---|---|---|
| DT-1 | `DashboardScreen.kt` de 1142 líneas — extraer componentes | Alta | Refactor |
| DT-2 | Datos hardcodeados en `StepsCard` (calorías, distancia, tiempo) | Alta | Fix |
| DT-3 | `AccelerometerDataSource` detecta caída pero el ViewModel no lo usa en la UI | Media | Feature |
| DT-4 | `infraestructure/` → `data/` migración pendiente (README la menciona) | Media | Chore |
| DT-5 | No hay estado vacío real en `RoutineListScreen` | Media | UX |
| DT-6 | `StatsInfoItem` en Dashboard muestra datos de pasos hardcodeados | Alta | Fix |
| DT-7 | `AchievementsCard` muestra `totalCount = 3` hardcodeado | Baja | Fix |
| DT-8 | `UserScreen.kt` existe junto a `ProfileScreen.kt` — roles duplicados? | Media | Refactor |

---

## Distribución de tareas — 4 Devs · ~21 pts cada uno

> [!IMPORTANT]
> La distribución intenta respetar los dominios técnicos ya establecidos en el tablero actual (Dev 1 = Auth/Sensores, Dev 2 = Cámara/Rehab, Dev 3 = Mapa, Dev 4 = Dashboard/Progreso) pero redistribuye las nuevas tareas y deuda técnica de forma equitativa.

---

## 🧑‍💻 Dev 1 — Auth · Sensores · Profile · UX polish

**Dominio:** autenticación, perfil, sensor base, deuda técnica de navegación

| # | Tarea | Tipo | SP | Sprint |
|---|---|---|---|---|
| 1.1 | **Fix: eliminar ícono perfil del header del Dashboard** — Quitar `Icon(Person)` de `DashboardHeader`, verificar que Profile sea accesible únicamente por Bottom Nav | Fix | 1 | S1 |
| 1.2 | **Feat: mover Achievements a ProfileScreen** — Integrar `AchievementsView` como sección en `ProfileScreen`, eliminar route standalone `Achievements`, cambiar `AchievementsCard` del dashboard a preview compacto con navegación a perfil | Feature | 3 | S1 |
| 1.3 | **Refactor: extraer componentes de DashboardScreen** — Mover `RomProgressCard`, `StepsCard`, `AchievementsCard`, `LastSessionCard`, `ActiveRoutineBanner`, `RomProgressRing` a archivos separados en `ui/components/dashboard/` | Refactor | 3 | S1 |
| 1.4 | **Fix: datos hardcodeados en StepsCard** — Conectar calorías, distancia y tiempo activo a datos reales del `DashboardViewModel` (pueden venir de Health Connect o de las sesiones Room) | Fix | 2 | S2 |
| 1.5 | **Chore: migración `infraestructure/` → `data/`** — Completar la migración documentada en el README: mover archivos, actualizar package declarations e imports | Chore | 3 | S2 |
| 1.6 | **Feat: Lottie en Splash, Achievements Dialog y Empty State** — Agregar dependencia Lottie, reemplazar logo estático del Splash por animación Lottie, animar el Achievement Unlocked Dialog, agregar ilustración Lottie al estado vacío de Progreso | Feature | 2 | S3 |
| 1.7 | **Test: ProfileViewModel + SignOut flow** — Testear `UpdateUserUseCase`, `SignOutUseCase`, toggle dark mode en DataStore, limpieza de back stack | Test | 3 | S4 |
| 1.8 | **Docs: README actualizado con nueva estructura de carpetas y flujo de navegación** | Docs | 1 | S4 |
| 1.9 | **Fix: `UserScreen.kt` vs `ProfileScreen.kt`** — Revisar si son duplicados y consolidar en uno | Fix | 1 | S3 |

**Total: 19 pts** _(ajustable: +2 si se suman edge cases de Lottie)_

---

## 🧑‍💻 Dev 2 — Cámara · Rehab · Health Connect · Wearables

**Dominio:** CameraX, ML Kit, Health Connect, integración wearables

| # | Tarea | Tipo | SP |Sprint |
|---|---|---|---|---|
| 2.1 | **Feat: Health Connect — permisos granulares y onboarding real** — Mostrar pantalla de permisos de Health Connect si no están concedidos, manejar el caso de que Health Connect no esté instalado (link a Play Store), flujo de re-permiso | Feature | 3 | S1 |
| 2.2 | **Feat: FC en tiempo real desde Wear OS durante sesión rehab** — En `RehabSessionScreen`, leer FC desde Health Connect Ejercice Session o via pasive monitoring; mostrar en el panel de telemetría junto al ángulo | Feature | 3 | S2 |
| 2.3 | **Fix: `AccelerometerDataSource` conectado a UI** — Actualmente el datasource existe y el test también, pero la detección de movimiento brusco (`>15 m/s²`) no pausa la sesión ni muestra alerta en `RehabSessionScreen`. Conectar el Flow al ViewModel y manejar el evento | Fix | 2 | S1 |
| 2.4 | **Feat: detección de fatiga automática real** — Implementar la lógica de velocidad de movimiento en el `RehabSessionViewModel`: si la velocidad baja 30% vs la primera rep, emitir evento de pausa automática con mensaje "Tomá un descanso · 60s" | Feature | 3 | S2 |
| 2.5 | **Feat: vinculación a Google Fit (legacy) via Health Connect bridge** — Health Connect actúa como bridge hacia Google Fit en Android 14+. Verificar que los datos de sesión escritos en Health Connect sean visibles desde Google Fit. Documentar el límite de lo que se puede hacer sin la API de Google Fit directa | Feature | 2 | S3 |
| 2.6 | **Chore: smartwatch data — investigación y spike** — Evaluar si conviene usar Health Connect Passive Monitoring API o una companion app Wear OS. Documentar decisión técnica en un ADR. No implementar la companion app (está fuera del scope) | Chore | 2 | S3 |
| 2.7 | **Test: `RehabSessionViewModel` — fatiga, acelerómetro, rep counter** — Agregar casos para pausa por acelerómetro, pausa por fatiga, conteo de reps exitosas, flujo SOS | Test | 3 | S4 |
| 2.8 | **Test: `HealthConnectDataSource` — completar tests existentes** — El test existe pero está incompleto. Agregar casos: permiso denegado, HC no instalado, lectura de SpO2, FC promedio | Test | 2 | S4 |

**Total: 20 pts**

---

## 🧑‍💻 Dev 3 — Mapa · Red Médica · UX environment · Routing

**Dominio:** Maps, Directions, ubicación, UX de pantallas secundarias

| # | Tarea | Tipo | SP | Sprint |
|---|---|---|---|---|
| 3.1 | **UX: Environment Check — estado vacío y feedback háptico** — Si el sensor de luz no está disponible en el dispositivo, mostrar estado alternativo. Agregar `HapticFeedback` cuando el semáforo cambia de estado (verde/amarillo/rojo) | Feature | 2 | S1 |
| 3.2 | **Fix: mapa — filtro de radio 5km funcional** — Verificar que `ClinicEntity` filtra correctamente por radio desde la ubicación del usuario. Actualmente el README indica "5 clínicas de prueba" mockeadas — validar que el filtro de distancia real funcione | Fix | 2 | S1 |
| 3.3 | **Feat: banner de emergencia SOS en MapScreen** — Implementar el estado de banner rojo cuando viene el flag de emergencia (navegación desde botón SOS). Actualmente el flujo SOS navega al mapa pero no hay banner visible en el código | Feature | 3 | S2 |
| 3.4 | **Feat: Directions API — polilínea + info distancia/tiempo** — En `ClinicDetailScreen` (bottom sheet), el botón "Cómo llegar" debe llamar a la API de Routing (GraphHopper ya integrado), dibujar la polilínea en el mapa y mostrar distancia y tiempo estimado. Verificar que limpiar la polilínea al volver funcione | Feature | 3 | S2 |
| 3.5 | **UX: skeleton loader en ClinicDetail** — Mientras carga la ruta, mostrar skeleton en el área inferior del bottom sheet. Actualmente se usa `CircularProgressIndicator` genérico | Feature | 2 | S3 |
| 3.6 | **Feat: "Llamar a la clínica" — intent de llamada** — El tap en el número de teléfono debe abrir el dialer del sistema. Implementar `Intent(Intent.ACTION_DIAL)` con el número de `ClinicEntity` | Feature | 1 | S3 |
| 3.7 | **Chore: permisos de ubicación — mejor flujo de denegación** — Si el usuario deniega `ACCESS_FINE_LOCATION`, mostrar un Snackbar con opción de ir a Settings en lugar de simplemente no cargar el mapa | Chore | 2 | S3 |
| 3.8 | **Test: `MapScreenViewModel` — estados de emergencia, carga de clínicas, permiso denegado** — Completar los tests del ViewModel que ya existe | Test | 3 | S4 |
| 3.9 | **Test: routing repository — polilínea, distancia, error API** | Test | 2 | S4 |

**Total: 20 pts**

---

## 🧑‍💻 Dev 4 — Dashboard · Progreso · Logros · Rutinas

**Dominio:** Dashboard refactorizado, estadísticas, gamificación, Health Connect en progreso

| # | Tarea | Tipo | SP | Sprint |
|---|---|---|---|---|
| 4.1 | **Fix: `AchievementsCard` hardcode `totalCount = 3`** — Leer el total real de achievements desde `AchievementRepository` en el ViewModel, no hardcodearlo en la UI | Fix | 1 | S1 |
| 4.2 | **Feat: estado vacío real en ProgressScreen** — Cuando no hay sesiones registradas, mostrar la ilustración + mensaje definido en el plan. Actualmente falta el empty state | Feature | 2 | S1 |
| 4.3 | **Feat: tooltip en gráfico ROM — tap en punto** — Al tocar un punto del gráfico de línea en `ProgressScreen`, mostrar un tooltip con fecha, ROM exacto, ejercicio y repeticiones. Vico/MPAndroidChart tiene soporte nativo | Feature | 3 | S2 |
| 4.4 | **Feat: gráfico de barras FC promedio por sesión** — Leer datos de Health Connect (FC promedio por sesión) y mostrarlos en el gráfico de barras de `ProgressScreen`. Verificar que el `HealthConnectDataSource` exista (sí, existe) | Feature | 3 | S2 |
| 4.5 | **Feat: tarjetas de resumen en Progress** — Pasos semana, calorías totales y sesiones completadas. Combinar datos de Room + Health Connect en `ProgressViewModel` | Feature | 2 | S2 |
| 4.6 | **UX: `RoutineListScreen` — estado vacío** — Si no hay ejercicios asignados para el día, mostrar un estado vacío con mensaje motivacional en lugar de una lista vacía | Feature | 1 | S3 |
| 4.7 | **Feat: barra de progreso total del día en RoutineList** — X de Y ejercicios completados con porcentaje animado (ya está en el plan, verificar si está implementado o solo diseñado) | Feature | 2 | S3 |
| 4.8 | **Chore: `DashboardViewModel` — separar responsabilidades** — Actualmente el ViewModel puede estar haciendo demasiado. Revisar si la lógica de pasos y logros debe dividirse o si el ViewModel es manejable | Chore | 2 | S3 |
| 4.9 | **Test: `DashboardViewModel` + `ProgressViewModel`** — Completar los tests existentes: StateFlow con Turbine, datos mockeados de Room, casos de logro desbloqueado | Test | 3 | S4 |
| 4.10 | **Test: `AchievementsViewModel`** — El test existe, revisar cobertura: criterios de desbloqueo, spring animation no es testeable pero el estado sí | Test | 2 | S4 |

**Total: 21 pts**

---

## Vista de sprints — qué entra en cada uno

```
S1 — Base y correcciones críticas (fixes que bloquean UX)
├─ Dev 1: Fix home screen (1.1) · Achievements → Profile (1.2) · Refactor Dashboard (1.3)
├─ Dev 2: HC permisos reales (2.1) · Acelerómetro → UI (2.3)
├─ Dev 3: Env Check UX (3.1) · Fix radio 5km (3.2)
└─ Dev 4: Fix totalCount (4.1) · Empty state Progress (4.2)

S2 — Features core del producto
├─ Dev 1: Fix StepsCard hardcode (1.4) · Migración infraestructure→data (1.5)
├─ Dev 2: FC en tiempo real Wear OS (2.2) · Fatiga automática (2.4)
├─ Dev 3: Banner SOS (3.3) · Directions API polilínea (3.4)
└─ Dev 4: Tooltip gráfico ROM (4.3) · Gráfico FC Health Connect (4.4) · Tarjetas resumen (4.5)

S3 — UX polish + integraciones secundarias
├─ Dev 1: Lottie animations (1.6) · Fix UserScreen duplicado (1.9)
├─ Dev 2: Google Fit bridge (2.5) · Spike smartwatches (2.6)
├─ Dev 3: Skeleton loader (3.5) · Intent llamada (3.6) · Permisos UX (3.7)
└─ Dev 4: Empty state RoutineList (4.6) · Barra progreso día (4.7) · Refactor DashVM (4.8)

S4 — Tests, docs y cierre
├─ Dev 1: ProfileViewModel tests (1.7) · README docs (1.8)
├─ Dev 2: RehabSessionVM tests (2.7) · HealthConnect tests (2.8)
├─ Dev 3: MapScreenVM tests (3.8) · Routing repo tests (3.9)
└─ Dev 4: DashboardVM + ProgressVM tests (4.9) · AchievementsVM tests (4.10)
```

---

## Totales por developer

| Developer | S1 | S2 | S3 | S4 | Total |
|---|---|---|---|---|---|
| Dev 1 | 7 pts | 5 pts | 3 pts | 4 pts | **19 pts** |
| Dev 2 | 5 pts | 6 pts | 4 pts | 5 pts | **20 pts** |
| Dev 3 | 4 pts | 6 pts | 5 pts | 5 pts | **20 pts** |
| Dev 4 | 3 pts | 8 pts | 5 pts | 5 pts | **21 pts** |

> [!NOTE]
> Dev 1 tiene 19 pts por el refactor del Dashboard (1.3) que desbloquea trabajo de Dev 4. Si el refactor se subestima, se puede mover la tarea de docs (1.8) a un buffer de S4.

---

## Features que NO recomiendo hacer ahora (scope creep honesto)

| Ítem | Por qué no |
|---|---|
| Companion app Wear OS separada | Es un proyecto aparte. Health Connect como bridge ya cubre el caso de uso para este MVP |
| Directions API con Google Maps (cambiar GraphHopper) | GraphHopper ya está integrado y funciona. Cambiar tiene costo alto y cero beneficio para el usuario |
| Autenticación con Google Sign-In | Room local + DataStore es suficiente para el contexto académico. OAuth aumentaría la complejidad innecesariamente |
| Notificaciones push remotas | Solo hay StepCounter como servicio foreground. Las notificaciones push requieren backend que no está en scope |

---

## Criterios de Definition of Done (DoD) que propongo establecer

Antes de cerrar cualquier tarea:
- [ ] No hay datos hardcodeados en la UI (todo viene del ViewModel o de Room)
- [ ] El estado vacío está implementado en todas las pantallas con listas
- [ ] El estado de carga (`isLoading`) está manejado en todas las pantallas
- [ ] Los errores se muestran al usuario (no solo en logs)
- [ ] La navegación de back stack es correcta (especialmente en flujos SOS y rehab)
- [ ] El test del ViewModel correspondiente cubre el happy path y al menos un error path
- [ ] Cobertura ≥ 60% (requerimiento del pipeline CI ya configurado)
