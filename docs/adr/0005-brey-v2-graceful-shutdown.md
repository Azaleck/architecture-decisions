# 0005. Graceful shutdown of Br'eye v2: RP2040 power state machine and UART handshake

- Status: Accepted
- Date: June 2026 (first double-press and shutdown logic), refined July 2026
- Deciders: Melchior Vila (CTO); electronics by an external hardware designer

## Context

Br'eye v2 is battery powered: a Raspberry Pi CM4 runs the application, and an RP2040 co-processor on the main board drives power, LEDs, vibration and the buttons. Cutting power to a Linux board while it writes (SQLite database, update files) corrupts data. The device is used by children who press buttons, unplug it, or let the battery run flat.

The power hardware is layered. An always-on supervisor chip latches the power on a short press of the button and drops it on a long press (about 4.2 s) or on a `KILL` pulse. The RP2040 is powered only while that latch is set or USB is plugged in. The CM4 can only boot when the RP2040 allows it (`SD_PI`), and the CM4 reports that it is alive through `RUN_PG`.

## Decision

1. **The RP2040 owns the power**, through a state machine: `OFF → POWERING_ON → ON → GRACEFUL_SHUTDOWN → SHUTTING_DOWN → OFF`. It decides when the CM4 may boot and when the supervisor must cut power (`KILL` pulse). The hardware long press stays as a last-resort kill switch that works even if every piece of software is stuck.
2. **Shutdown is a handshake over the UART protocol**: frames with start and end markers, a length and a CRC-16 (CCITT), and a parser that resynchronises on the next start marker after any corrupted frame. The RP2040 asks the CM4 to shut down (`CMD_SHUTDOWN_REQUEST`), the CM4 acknowledges (`CMD_SHUTDOWN_ACK`) and runs a clean `shutdown`; the RP2040 retries if no acknowledgement arrives and cuts power only when the CM4 is down or after a timeout (8 s).
3. **Every shutdown has a cause code** sent over UART (power request, long press, empty battery, confirmed double press, graceful done, graceful timeout, requested by the CM4), so field problems can be explained.
4. **Accidental presses are filtered**: one press asks for confirmation (voice prompt and LEDs), a second press confirms; long-press thresholds are debounced.
5. **The long press is caught before the hardware acts**: the firmware measures how long the button signal has been low and starts the graceful shutdown after a few tens of milliseconds, well before the supervisor cuts the power. A user who holds the button therefore gets a clean shutdown, not a hard cut.
6. A heartbeat from the CM4 lets the RP2040 detect a frozen application, and its silence (a few seconds) is also how the RP2040 knows the CM4 has finished shutting down.
7. **The RP2040 can restart without touching the CM4**: at boot it reads `RUN_PG` and goes straight to `ON` if the CM4 is already running. A firmware update or a watchdog reset of the RP2040 therefore never cuts a running CM4 (needed by 0006).
8. **The CM4 decides which firmware the RP2040 runs**: at start it asks the RP2040 for its version over UART and, if it is not the one the application expects, flashes the expected one. The firmware ships with the CM4 application, which is updated from the web (0007), so the power logic can be fixed in the field like the rest of the software (details in 0006).
9. **`RUN_PG` is read only**: driving it low would hard-reset the CM4, which is exactly the uncontrolled cut this design avoids.
10. **Power inputs are filtered**: the battery, USB and power-request levels are read through the ADC over a small rolling window, and a missing power request must last 3 s before it counts, so electrical noise does not shut the device down.

## Alternatives considered

- Hard power switch: cheapest, but corrupts data.
- Shutdown handled only on the CM4: impossible when the CM4 is frozen or the battery is empty.
- Relying on the hardware long press alone: always works, but it is a hard cut, the very case that corrupts data.
- The first specification had more states (separate idle, battery check, alert and error states) and 30 s timeouts. The firmware merged them into five states with shorter timeouts, which is easier to test and keeps the device responsive.

## Consequences

- Positive: no more corrupted databases from power loss; the battery state is visible on the LEDs; causes are logged; the RP2040 can be updated in the field without stopping the device.
- Negative: two pieces of software (C firmware and Python) must agree on the protocol; timeouts are a trade-off between safety and responsiveness.
- Hardware and firmware cover for each other: on the first board revision the 5 V rail leaked a few volts after a `KILL` in battery mode. The firmware keeps the CM4 held off until the next button press, so the device behaves correctly, and the leak is fixed in hardware on the next revision.

## Validation

- Bench tests of each cause: long press, double press, unplug, battery drained, frozen CM4 (timeout path).
- Long-press detection checked on real hardware in battery mode: the CM4 is stopped before the power rails go down.
- Recovery side: a malformed database is detected at start, removed and rebuilt (see 0007, Validation).

## References

- Firmware: `pio_ws2812/power_state.c`, `pio_ws2812/include/command_handler.h` (commands 0x10-0x15, cause codes)
- Specifications: `pio_ws2812/documentations/dev/power-sequence.md`, `pio_ws2812/documentations/dev/serial-protocol.md`
- CM4 side: `exceller/Source/Singleton/microcontroller.py` (`prepareShutdown`, `shutdown`, UART dispatch)
- History: "feat(shutdown): adding short press double check" (4a8f2df, 2026-06-01), "feat(shutdown): shutdown + versionning" (7303041, 2026-06-03), "feat(shutdown): better power REQ" (e45f00d, 2026-07-30)
