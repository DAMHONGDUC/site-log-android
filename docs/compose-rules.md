# Compose rules

<dl><dd>

> **Small composables, state hoisted, logic in the view model, styles from the design system, text from resources, meaningful names.**

- Android twin of [SwiftUI rules](https://github.com/DAMHONGDUC/site_log_swift_ui/blob/main/docs/swiftui-rules.md).

</dd></dl>

## Contents

<dl><dd>

1. [Composables](#1-composables)
2. [State](#2-state)
3. [Architecture](#3-architecture)
4. [Strings](#4-strings)
5. [Design system](#5-design-system)
6. [Coroutines](#6-coroutines)
7. [Quality](#7-quality)
8. [PR checklist](#8-pr-checklist)

</dd></dl>

## 1. Composables

<dl><dd>

1. **Small composables, one job each.** Past ~50 lines, extract a private composable.
2. **Screen + Content split.** `<Name>Screen` gets the view model and collects state; `<Name>Content(state, onAction…)` is stateless and holds the layout and the previews.
3. **No work in composition.** No I/O, no sorting large lists, no business `if`; derive with `remember(key) { }` or in the view model.
4. **`Modifier` is the first optional parameter** of every reusable composable, applied to the root element.
5. **Edge to edge:** a screen outside a `Scaffold` adds `Modifier.safeDrawingPadding()`; content never sits under the status or navigation bar.
6. **Lists:** `LazyColumn` with `key = { it.id }`; row classes are immutable `data class`es (`@Immutable` when they hold a `List`).
7. **Dialogs in full form.** Name every argument; one argument per line.

    ```kotlin
    if (isDeleteDialogShown) {
        AlertDialog(
            onDismissRequest = { isDeleteDialogShown = false },
            title = {
                Text(text = stringResource(R.string.project_delete_title))
            },
            text = {
                Text(text = stringResource(R.string.project_delete_message))
            },
            confirmButton = {
                TextButton(
                    onClick = {
                        isDeleteDialogShown = false
                        onDelete()
                    },
                ) {
                    Text(text = stringResource(R.string.project_delete_confirm))
                }
            },
            dismissButton = {
                TextButton(
                    onClick = { isDeleteDialogShown = false },
                ) {
                    Text(text = stringResource(R.string.common_cancel))
                }
            },
        )
    }
    ```

</dd></dl>

## 2. State

<dl><dd>

8. **Pick the right holder, keep state as low as possible:**
    - `remember { mutableStateOf() }`: UI-only state of one composable (a dialog shown, a text field draft).
    - `rememberSaveable`: the same, surviving rotation and process death.
    - View model `StateFlow`: everything the screen shows from data.
    - `CompositionLocal`: app-wide objects only (`LocalNavigationCoordinator`), never data.
9. **Single source of truth.** Never copy view model state into `remember`.
10. **A sealed status, not loose booleans:** `Loading / Loaded / Failed`.
11. **Collect with lifecycle:** `collectAsStateWithLifecycle()`, never `collectAsState()`.

</dd></dl>

## 3. Architecture

<dl><dd>

12. **Composables know nothing about network or database.** They talk to the view model.
13. **Dependencies come through the constructor** (view models) or as parameters (composables). Name them by role: `loadSettingsUseCase`, `accessRepository`, `accessLocalDataSource`.
14. **Feature folders:** each feature owns its routes, screens and view models.
15. **View models import no Compose** (`androidx.compose.*`); they expose plain state.
16. **Navigation through the coordinator:** `coordinator.push(ProjectsRoute.Detail(projectId))`; a composable never builds another feature's screen.

</dd></dl>

## 4. Strings

<dl><dd>

17. **Every UI string is a resource:** `stringResource(R.string.access_sign_in_title)`; fixed values (names, dates, ids, counts, measurements, file names) are passed directly.

    ```kotlin
    Text(text = stringResource(R.string.access_sign_in_title))   // ✅ UI copy
    Text(text = project.name)                                    // ✅ fixed value
    Text(text = "Seed data")                                     // ❌ hard-coded UI copy
    ```

18. **Keys by UI role, `snake_case`:** `access_sign_in_title`, `access_sign_in_description`, `access_sign_in_benefit_1`.
19. **No concatenation:** placeholders and plurals.

    ```kotlin
    Text(text = stringResource(R.string.devmenu_finished, name))                          // ✅ "%1$s finished"
    Text(text = pluralStringResource(R.plurals.issues_open_count, count, count))          // ✅
    Text(text = name + " finished")                                                       // ❌
    ```

20. **View models return resource ids** (`@StringRes Int`) or a small `UiText` sealed type, never translated strings.
21. **Vietnamese:** `values-vi/strings.xml` holds every key; Lint `MissingTranslation` is an error.

</dd></dl>

## 5. Design system

<dl><dd>

22. **Every color, space, shape and text style comes from `:designsystem`:** `DSColor`, `DSSpacing`, `DSTextStyle`; no `Color(0x…)`, no raw `dp`, no `TextStyle(...)` in screens.
23. `SiteLogTheme` wraps the app: it maps the tokens onto `MaterialTheme` (`colorScheme`, `typography`, `shapes`) so Material 3 components follow the design; dynamic color is **off** (brand colors on every phone).
24. `DSSpacing` names use the dp value: `DSSpacing.spacing8` = `8.dp`, `DSSpacing.spacing48` = `48.dp` (minimum touch target).
25. `DSTextStyle` mirrors iOS sizes in `sp` so text scales with the user's font size:

| Style | Size | Weight | Line height |
|---|---:|---|---:|
| `largeTitle` | 34 sp | Bold | 41 sp |
| `pageTitle` | 28 sp | Bold | 34 sp |
| `title` | 24 sp | Bold | 30 sp |
| `section` | 20 sp | SemiBold | 25 sp |
| `headline` | 17 sp | SemiBold | 22 sp |
| `body` | 17 sp | Normal | 22 sp |
| `subhead` | 15 sp | Normal | 20 sp |
| `footnote` | 13 sp | Normal | 18 sp |
| `caption` | 11 sp | Medium | 13 sp |
| `code` | 13 sp | Normal (monospace) | 18 sp |
| `figure` | 28 sp | SemiBold | 34 sp |

- Font: Inter (bundled in `:designsystem` `res/font/`, covers Vietnamese) for sans; JetBrains Mono for `code`. `caption` is 11 sp, not iOS's 10, because Material's smallest readable label is 11.
- Repeated styling becomes a `DS*` component (`DSButton`, `DSListRow`, `DSEmptyState`).
- Every `Content` composable has previews: light, dark, `fontScale = 2f`.

</dd></dl>

## 6. Coroutines

<dl><dd>

26. **UI on `Main`.** View models launch in `viewModelScope`; heavy work moves to an injected dispatcher inside the data layer, not in the view model.
27. **`LaunchedEffect(key)`** for work tied to a composable's lifetime; never `GlobalScope`.
28. **Never catch `CancellationException`** without rethrowing it.

</dd></dl>

## 7. Quality

<dl><dd>

29. **Names by meaning:** `isLoading`, `onSaveClick()`. Never `flag1`, `handle()`.
30. **detekt + ktlint + Lint in CI**, zero warnings.
31. **Dark theme and accessibility from day one:** `contentDescription` on every icon button, `Modifier.semantics { heading() }` on titles, 48 dp targets, no information by color alone.
32. **Comments say why:** inside code at most 2 lines, a KDoc at most 3.
33. **Blank lines:** one after the variable declarations, one before every `return` unless it opens its block.

</dd></dl>

## 8. PR checklist

<dl><dd>

- [ ] Composables are short, with no business logic
- [ ] State lives in the right holder, with one source of truth
- [ ] Status is sealed, not several booleans
- [ ] No network, database or singletons in composables
- [ ] No raw colors, dp, or text styles: `DSColor`, `DSSpacing`, `DSTextStyle`
- [ ] No hard-coded UI strings; Vietnamese added
- [ ] Previews present (light, dark, large font)
- [ ] `contentDescription` on icons that act

</dd></dl>
