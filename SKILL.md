---
name: flowvi
description: >
  Use when creating or modifying flowvi MVI state management in a Kotlin Multiplatform
  + Compose project. Triggers: flowvi, rememberInteractor, Interactor, Reducer,
  SideEffect, Dispatcher, MVI state management, creating new interactors, adding
  reducers or side effects, editing any file importing tv.dpal.flowvi.
---

# FlowVi Skill

A tiny, Compose-first MVI (Model-View-Intent) toolkit for Kotlin Multiplatform.
It provides just enough structure to build unidirectional data flows without
locking you into a large framework.

- **Package:** `tv.dpal.flowvi`
- **Modules:** `flowvi-core` (primitives), `flowvi-compose` (Compose integration)
- **Author:** Evan Holtrop
- **License:** Apache 2.0

## When This Skill MUST Be Used

**ALWAYS invoke this skill when:**

- Creating a new screen or feature that needs state management
- Editing ANY file that imports `tv.dpal.flowvi.*`
- Writing `rememberInteractor()`, `Reducer`, `SideEffect`, or `Interactor` code
- Refactoring existing MVI state management
- Testing reducers or interactors
- Configuring flowvi as a dependency or composite build

## Core Concepts

| Primitive | Signature | Purpose |
|-----------|-----------|---------|
| **`State`** | Data class / sealed interface | Immutable snapshot of UI data. Must be `@Serializable` for `rememberSaveable` persistence |
| **`Event`** | Sealed class / interface | User or system intent (clicks, results, timers) |
| **`Reducer`** | `suspend (State, Event) -> State` | Pure function computing the next state. Side-effect free. |
| **`SideEffect`** | `suspend (Dispatcher<Event>, State, Event) -> Unit` | Fire-and-forget async operations (I/O, analytics, navigation). Can dispatch follow-up events. |
| **`Dispatcher`** | `(Event) -> Unit` | Function that sends an event into the MVI channel. The `Interactor` itself is a `Dispatcher`. |
| **`Interactor`** | Class | Orchestrates the loop: merges source `Flow<State>` + dispatched events, applies reducers, runs side effects, exposes `state: StateFlow<State>`. |

### Pipeline

```
source Flow<State>
  ─flatMapLatest─> events Channel
    ─scan─> reducers.fold(state, event) -> newState
      ─withContext(Default)─> sideEffects.forEach { it(dispatcher, newState, event) }
        ─stateIn─> hot StateFlow<State>
```

## Gradle Setup

### Dependency Declaration (`gradle/libs.versions.toml`)

```toml
[versions]
flowvi-version = "0.1.0"

[libraries]
flowvi-core = { module = "tv.dpal:flowvi-core", version.ref = "flowvi-version" }
flowvi-compose = { module = "tv.dpal:flowvi-compose", version.ref = "flowvi-version" }
```

### Module Usage (`build.gradle.kts`)

```kotlin
dependencies {
    implementation(libs.flowvi.core)
    implementation(libs.flowvi.compose)
}
```

### Composite Build for Local Development

Flow into `settings.gradle.kts` for local development alongside the host project:

```kotlin
if (file("libs/flowvi/enablecompositebuilds").exists()) {
    includeBuild("libs/flowvi") {
        dependencySubstitution {
            substitute(module("tv.dpal:flowvi-core")).using(project(":core"))
            substitute(module("tv.dpal:flowvi-compose")).using(project(":compose"))
        }
    }
}
```

Activate via Gradle tasks that create/delete the `enablecompositebuilds` marker file.

## Patterns

### Pattern 1: Basic Interactor (No Upstream Source)

For simple UI state that doesn't need a repository source.

```kotlin
@Serializable
data class CalculatorState(
    val expression: List<CalculatorExpressionItem> = emptyList(),
)

sealed interface CalculatorEvent {
    data class DigitAdded(val digit: Int) : CalculatorEvent
    data object Clear : CalculatorEvent
}

@Composable
fun rememberCalculatorInteractor(
    selectedUOM: UOM,
): Interactor<CalculatorState, CalculatorEvent> = rememberInteractor(
    initialState = CalculatorState(),
    reducers = calculatorReducers(selectedUOM) + listOf(EqualsReducer),
)

// Reducer is a pure function — easy to test
private fun calculatorReducers(uom: UOM): List<Reducer<CalculatorState, CalculatorEvent>> = listOf(
    Reducer { state, event ->
        when (event) {
            is CalculatorEvent.DigitAdded -> state.copy(
                expression = state.expression + CalculatorExpressionItem(event.digit, uom)
            )
            CalculatorEvent.Clear -> CalculatorState()
        }
    }
)
```

**When to use:** Self-contained UI with no external data dependencies. The interactor owns all state.

---

### Pattern 2: Interactor with Upstream Source

Use the `source` parameter to connect repository flows to your interactor. The interactor folds events on top of the latest source emission.

```kotlin
typealias HomeInteractor = Interactor<HomeState, HomeEvent>

@Serializable
sealed interface HomeState {
    @Serializable data object Loading : HomeState
    @Serializable data object Empty : HomeState
    @Serializable data class Content(
        val selectedTab: Tab,
        val dashboardV3: Boolean = false,
    ) : HomeState
}

sealed class HomeEvent {
    data object DashboardClicked : HomeEvent()
    data object CalendarClicked : HomeEvent()
    data object SettingsClicked : HomeEvent()
    data object AddSetClicked : HomeEvent()
    data object AddLiftClicked : HomeEvent()
}

@Composable
fun rememberHomeInteractor(
    initialTab: Tab = Tab.Dashboard,
    liftRepository: ILiftRepository = dependencies.liftRepository,
    goalsRepository: IGoalRepository = dependencies.goalsRepository,
    settingsRepository: ISettingsRepository = dependencies.settingsRepository,
    analytics: Analytics = dependencies.analytics,
    navCoordinator: NavCoordinator = LocalNavCoordinator.current,
): HomeInteractor = rememberInteractor(
    initialState = HomeState.Loading,
    source = { state ->
        combine(
            liftRepository.listenAll(),
            goalsRepository.getAll(),
        ) { lifts, goals ->
            if (lifts.isEmpty()) HomeState.Empty
            else HomeState.Content(
                selectedTab = (state as? HomeState.Content)?.selectedTab ?: initialTab,
                dashboardV3 = settingsRepository.get(Setting.DashboardV3)
            )
        }
    },
    reducers = listOf(homeReducer),
    sideEffects = homeSideEffects(navCoordinator, analytics)
)

internal val homeReducer = Reducer<HomeState, HomeEvent> { state, event ->
    when (state) {
        is HomeState.Content -> when (event) {
            HomeEvent.DashboardClicked -> state.copy(selectedTab = Tab.Dashboard)
            HomeEvent.CalendarClicked -> state.copy(selectedTab = Tab.WorkoutCalendar)
            else -> state
        }
        else -> state
    }
}

internal fun homeSideEffects(
    navCoordinator: NavCoordinator,
    analytics: Analytics,
) = listOf<SideEffect<HomeState, HomeEvent>>(
    SideEffect { _, _, event ->
        when (event) {
            HomeEvent.AddSetClicked -> navCoordinator.present(CreateSet())
            HomeEvent.SettingsClicked -> navCoordinator.present(Settings)
            HomeEvent.DashboardClicked -> analytics.trackScreenView(...)
            HomeEvent.CalendarClicked -> analytics.trackScreenView(...)
            else -> {}
        }
    }
)
```

**Key patterns:**
- `typealias` for cleaner type references
- `combine` multiple repository flows in `source`
- Extracted `val homeReducer` and `fun homeSideEffects()` for testability
- Navigation and analytics are side effects, not reducer concerns

---

### Pattern 3: Side Effects Dispatching Follow-up Events

After async work completes, dispatch a new event back into the loop via the `dispatcher` parameter.

```kotlin
rememberInteractor(
    initialState = ServerSettingsState(status = ServerStatus.Off),
    reducers = listOf(
        Reducer { state, event ->
            when (event) {
                ServerSettingsEvent.TurnOffServer,
                ServerSettingsEvent.TurnOnServer -> state.copy(status = ServerStatus.Unknown)

                is ServerSettingsEvent.ServerStatusUpdated -> state.copy(status = event.status)
            }
        }
    ),
    sideEffects = listOf(
        SideEffect { dispatcher, _, event ->
            when (event) {
                ServerSettingsEvent.TurnOffServer -> {
                    server.stop()
                    while (server.isRunning()) { delay(500) }
                    dispatcher(ServerSettingsEvent.ServerStatusUpdated(ServerStatus.Off))
                }
                ServerSettingsEvent.TurnOnServer -> {
                    server.start()
                    withTimeoutOrNull(2000L) {
                        dispatcher(ServerSettingsEvent.ServerStatusUpdated(ServerStatus.On))
                    }
                    while (!server.isRunning()) { delay(500) }
                }
                else -> {}
            }
        }
    )
)
```

**Key pattern:**
- Optimistic state: reducer immediately sets `ServerStatus.Unknown`
- Side effect performs async I/O (server start/stop)
- On completion, side effect calls `dispatcher(ServerStatusUpdated(...))` which flows back through the reducer
- This avoids exposing non-serializable state or complex async logic in the reducer

---

### Pattern 4: State Resolver + Composed Side Effects

Use `stateResolver` to preserve UI state (e.g., selected date, scroll position) when the upstream source re-emits. Compose multiple side effect lists with `+`.

```kotlin
fun rememberWorkoutCalendarInteractor(
    initialDate: LocalDate = today,
) = rememberInteractor(
    initialState = WorkoutCalendarState(selectedDate = initialDate),
    stateResolver = { saved, source ->
        source.copy(
            selectedDate = saved.selectedDate,
            selectedWorkout = saved.selectedWorkout,
            log = saved.log,
        )
    },
    source = { workoutCalendarSourceData() },
    reducers = listOf(WorkoutCalendarReducer),
    sideEffects = navigationSideEffects() + dataSideEffects(),
)

private fun navigationSideEffects(
    navCoordinator: NavCoordinator = LocalNavCoordinator.current,
) = listOf(
    SideEffect { _, _, event ->
        when (event) {
            is WorkoutCalendarEvent.AddWorkoutClicked ->
                navCoordinator.present(Destination.CreateWorkout(event.onDate))
            is WorkoutCalendarEvent.WorkoutClicked ->
                navCoordinator.present(Destination.CreateWorkout(event.workout.date))
            else -> {}
        }
    }
)

private fun dataSideEffects() = listOf(
    SideEffect { _, state, event ->
        when (event) {
            is WorkoutCalendarEvent.AddToWorkout -> { /* persist to repository */ }
            else -> {}
        }
    }
)
```

**Key patterns:**
- `stateResolver` merges saved/restored state with fresh source emissions — prevents losing user's place
- Side effect lists concatenated with `+` for clean separation of concerns (navigation vs persistence)
- Source is extracted as a separate function

---

### Pattern 5: Inline Reducer + Nested Event Hierarchy

For simpler screens, define the reducer and side effects inline within the `rememberInteractor` call. Use nested sealed event hierarchies to group related events.

```kotlin
rememberInteractor(
    initialState = MovementDetailsState(movement = Movement(id = movementId)),
    source = { state ->
        combine(
            variationRepository.listen(movementId),
            setRepository.listenAll(variationId = movementId),
        ) { movement, sets ->
            MovementDetailsState(
                movement = movement ?: state.movement,
                cards = sets.groupBy { it.date }.map { (date, sets) ->
                    MovementDetailsCard(date.toString("EEEE, MM d"), sets)
                }
            )
        }
    },
    sideEffects = listOf(
        SideEffect { _, state, event ->
            when (event) {
                is MovementDetailsEvent.UpdateMovement.NotesUpdated ->
                    movementRepository.save(state.movement.copy(notes = event.notes))
                MovementDetailsEvent.AddSetClicked ->
                    navCoordinator.present(CreateSet(movementId))
                else -> {}
            }
        }
    ),
    reducers = listOf(
        Reducer { state, event ->
            when (event) {
                is MovementDetailsEvent.UpdateMovement.NotesUpdated ->
                    state.copy(movement = state.movement.copy(notes = event.notes))
                else -> state
            }
        }
    )
)

// Nested event hierarchy groups related intents
sealed interface MovementDetailsEvent {
    sealed interface UpdateMovement : MovementDetailsEvent {
        data class NotesUpdated(val notes: String) : UpdateMovement
        data class NameUpdated(val name: String) : UpdateMovement
        data object ToggleBodyWeight : UpdateMovement
    }
    data object AddSetClicked : MovementDetailsEvent
    data class SetClicked(val setId: String) : MovementDetailsEvent
}
```

**Key patterns:**
- All-in-one definition for simpler screens
- Nested sealed interface for event organization
- Optimistic UI update in reducer, persistence in side effect

---

## State Persistence

`rememberInteractor` uses `rememberSaveable` with `kotlinx.serialization` under the hood.

Requirements:
- **State must be `@Serializable`** — applies to all nested data classes
- Sealed interfaces need `@Serializable` on each subclass

```kotlin
@Serializable
sealed interface HomeState {
    @Serializable data object Loading : HomeState
    @Serializable data class Content(val selectedTab: Tab) : HomeState
}
```

The `stateResolver` parameter controls how a restored (saved) state merges with the latest source emission:

```kotlin
stateResolver = { savedState, sourceEmittedState ->
    // Default: drop saved state, use source emission
    // Custom: preserve selected UI elements from saved state
    sourceEmittedState.copy(selectedTab = savedState.selectedTab)
}
```

If your `State` is not serializable, provide a custom Saver or disable persistence by wrapping `rememberInteractor`.

## Testing Reducers

Reducers are pure functions — test them directly without instantiating an `Interactor`.

```kotlin
class CalculatorInteractorTest {
    @Test
    fun `digitReducer adds first digit to empty state`() = runTest {
        val digitReducer = Reducer<CalculatorState, CalculatorEvent> { state, event ->
            when (event) {
                is CalculatorEvent.DigitAdded -> state.copy(
                    expression = state.expression + CalculatorExpressionItem(event.digit, UOM.POUNDS)
                )
                else -> state
            }
        }

        val result = digitReducer(CalculatorState(), CalculatorEvent.DigitAdded(5))

        assertEquals(1, result.expression.size)
        assertEquals(5.0, result.expression.first().weight.value)
    }
}
```

For interactor-level tests (testing the full event flow), use Turbine on `interactor.state`:

```kotlin
interactor.state.test {
    skipItems(1) // skip initial state

    interactor(TestEvent.Increment)
    assertEquals(1, awaitItem().count)

    interactor(TestEvent.Add(10))
    assertEquals(11, awaitItem().count)
}
```

## Integration with Navi

When using flowvi with `tv.dpal:navi`, navigation is handled in side effects:

```kotlin
val navCoordinator = LocalNavCoordinator.current

SideEffect { _, _, event ->
    when (event) {
        is HomeEvent.AddSetClicked -> navCoordinator.present(CreateSet())
        is HomeEvent.SettingsClicked -> navCoordinator.present(Settings)
        else -> {}
    }
}
```

Import the NavCoordinator via `LocalNavCoordinator.current` (CompositionLocal), then call `present(Destination)` to navigate.

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| **State doesn't update on dispatch** | Reducer must return a **new instance** (use `.copy()`). Returning the same object won't trigger emission. |
| **Side effects not running** | Side effects run on `Dispatchers.Default`. For UI work (e.g., showing a dialog), wrap in `withContext(Dispatchers.Main)`. |
| **rememberSaveable crash: State not serializable** | Annotate State with `@Serializable` or provide a custom Saver. |
| **"ClassNotFoundException: tv.dpal.flowvi.Interactor"** | Ensure `flowvi-core` is on the classpath. |
| **Source re-emissions reset UI state** | Use `stateResolver` to preserve transient UI state. |
| **Events processed out of order** | Events are sequential via `Channel` — they process in order. Dispatched events from side effects go to the back of the queue. |
| **Type inference fails on `rememberInteractor`** | Be explicit: `rememberInteractor<State, Event>(...)` or use `typealias`. |

## Decision Framework

When building a new screen:

1. **Does the screen have external data dependencies (repositories, DB)?**
   - Yes → Use `source` parameter to combine repository flows
   - No → Use `source = { flow { emit(initialState) } }` (default)

2. **Does the event trigger I/O (network, database, file)?**
   - Yes → Put it in a `SideEffect`
   - No → Handle it in a `Reducer`

3. **Does the event need to trigger a follow-up event after async work?**
   - Write a `SideEffect` that calls `dispatcher(FollowUpEvent)` when done
   - Handle `FollowUpEvent` in the reducer

4. **Does the source produce state that overrides user selections?**
   - Use `stateResolver` to preserve the user's choice

5. **Does mutating state need persistence?**
   - Optimistic UI update in `Reducer` (instant feedback)
   - Async save in `SideEffect` (eventual consistency)

6. **Do you need to test the logic?**
   - Test `Reducer` directly as a pure function
   - Test full `Interactor` with Turbine for integration tests

## Example Requests

- "Create a settings screen" → Use `rememberInteractor` with a `SettingsState` and `SettingsEvent`, repository source, side effects for toggles
- "Add a loading state to the home screen" → Add `Loading` to the sealed `State` interface, map repository nulls to it in `source`
- "Persist data when user edits a field" → Optimistic update in reducer, save in side effect
- "Navigate when user taps an item" → `SideEffect` calls `navCoordinator.present(Destination)`
- "Test the counter reducer" → `Reducer.invoke(state, event)` directly with assertions
- "Fix state not updating after dispatch" → Ensure reducer returns `state.copy(...)` not the same reference
