# 0003. Observer pattern for language, settings and UI changes

- Status: Accepted
- Date: June 2024 (language change), extended to settings in September 2024, made a singleton in July 2025
- Deciders: Melchior Vila (CTO)

## Context

When a user changes the language, the voice speed or a shortcut, many unrelated parts must react: menus, the braille display, the history panel, keyboard accelerators, the screen-reader focus in the Windows app. Before, the API, the games and the settings had to know each other and call one another directly, which caused circular imports, tight coupling and forgotten updates (a menu still in the old language).

## Decision

Use the **Observer pattern** with a single shared subject:

- `Observable` (a singleton) keeps a list of callbacks per event type.
- Event types are an enum (`Observe`: layout, user, refresh display, give focus, quit…), and settings (`SettingEnum`) are events too.
- Components `attach(callback, event, priority=False)` and `detach` when they close; settings and UI code call `notify(event)`. Priority observers run first when order matters.

## Alternatives considered

- Direct calls between the settings, the API and the games: what we had, and what caused the circular imports and the coupling.
- A full event-bus library: heavier and with features we did not need, and another dependency in a frozen Windows executable. A small in-house subject could be adapted exactly to our needs (priority observers, settings as events).

## Consequences

- Positive: a language or setting change reaches every component, including ones added later, without touching the settings code.
- Positive: the same mechanism serves the device and the wxPython app.
- Negative: implicit flow (harder to see who reacts to an event); observers must detach to avoid leaks; a global singleton shares its state across tests.

## Validation

- Exercised by the unit tests that change settings and language; manual tests with NVDA and JAWS for focus and refresh events.

## References

- `Source/Singleton/observer_singleton.py`
- History: "feat(observer): For lang change !" (2024-06-04), "new parameters: speed, voice + observable updated" (2024-09-17), Observable made a singleton (07/2025), "Refactor focus observer attachment in main_frame.py" (2026-07-15)
