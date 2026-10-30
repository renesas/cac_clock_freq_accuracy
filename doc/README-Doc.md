# CAC

## 1. Overview

This software building block (SBB) demonstrates the Renesas RA Clock Accuracy Circuit (CAC) driver. When the user presses any key in the serial terminal, the reference application measures a selected target clock against a selected reference clock, reports whether the measurement completed within the configured limits, and prints the CAC counter value.

This document applies only to these reference boards:

- FPB-RA2E3
- FPB-RA4E2
- FPB-RA6E2
- FPB-RA8E1

## 2. Use Case

The SBB demonstrates how to use the CAC peripheral to check the accuracy of one MCU clock source relative to another. The configured measurement clock is counted during a period derived from the configured reference clock. The CAC compares the resulting count against lower and upper limits configured in FSP.

The demonstrated sequence is:

1. Initialize the UART console.
2. Open the Clock Generation Circuit (CGC) driver.
3. Open the CAC driver.
4. Start and verify the configured reference and measurement clocks when required.
5. Wait for a terminal key press.
6. Start a CAC measurement.
7. Wait for the callback to report measurement completion or an error event.
8. Read and display the CAC counter value after successful completion.
9. Stop the measurement and wait for the next command.

Typical uses include startup clock validation, clock-source verification, and integration testing of clock selections and CAC thresholds. The supplied files demonstrate the mechanism but do not define a product-level clock-safety policy.

## 3. Software Organization

### 3.1 Reusable SBB core

The reusable CAC and CGC control logic is contained in:

- `cac_ep.h`
- `cac_ep.c`

This layer:

- opens the FSP CGC instance;
- opens the FSP CAC instance;
- determines the configured reference and measurement clocks;
- starts and verifies applicable clock sources;
- starts a CAC measurement;
- waits for notification from the CAC callback;
- reads the CAC counter after a successful measurement;
- maps CAC events to FSP return codes; and
- closes the CAC and CGC instances during cleanup.

### 3.2 Demo/reference application

The terminal-driven demonstration is contained in:

- `cac_example_app.h`
- `cac_example_app.c`

This layer is demo-only application code. It initializes the UART console, displays the SBB banner and instructions, waits for user input, invokes the reusable CAC functions, prints the result, and enters a breakpoint trap after a fatal error.

The supplied source expects `hal_entry.c` to call:

```c
cac_example_app_run();
```

An integrating application can replace the complete demo layer while retaining the reusable core functions.

## 4. Core Code Behavior

### 4.1 CGC initialization

`cgc_init()` opens the generated CGC instance using `R_CGC_Open()`.

### 4.2 CAC initialization and clock preparation

`cac_init()` opens the generated CAC instance and examines the reference and measurement clock selections in `g_cac_cfg`.

For clock sources that require explicit startup, the function maps the CAC clock source to its corresponding CGC clock source, starts it with `R_CGC_ClockStart()`, and polls `R_CGC_ClockCheck()` until the clock stabilizes or an error/timeout occurs.

The source does not explicitly start:

- the IWDT clock when selected as a reference or measurement clock;
- PCLKB, which the source treats as always running; or
- the external clock when selected as the reference clock.

### 4.3 Measurement operation

`cac_measurement_process()` performs one measurement:

1. Clears the stored event, completion flag, and counter value.
2. Calls `R_CAC_StartMeasurement()`.
3. Polls the callback-updated status flag with a `UINT16_MAX` loop-count timeout.
4. Handles the received event:
   - `CAC_EVENT_MEASUREMENT_COMPLETE`: reads the counter using `R_CAC_Read()` and stops the measurement using `R_CAC_StopMeasurement()`.
   - `CAC_EVENT_FREQUENCY_ERROR`: returns `FSP_ERR_INVALID_CAC_REF_CLOCK`.
   - any other event: returns `FSP_ERR_OVERFLOW`.

### 4.4 Callback

`cac_callback()` records the FSP CAC event and sets the completion flag used by `cac_measurement_process()`.

The callback is part of the reusable implementation even though it is not declared as a customer-facing function in `cac_ep.h`. The FSP CAC configuration must reference it.

### 4.5 Cleanup

`cac_deinit()` and `cgc_deinit()` close their respective generated instances when those instances are open.

## 5. Public Functions to Review First

The reusable public interface is declared in `cac_ep.h`:

```c
fsp_err_t cgc_init(void);
fsp_err_t cac_init(void);
fsp_err_t cac_measurement_process(void);
void cac_deinit(void);
void cgc_deinit(void);
```

Recommended review order:

1. `cac_init()` to understand selection and startup of the configured clocks.
2. `cac_measurement_process()` to understand measurement, callback synchronization, event handling, and result reading.
3. `cgc_init()` to understand the CGC dependency.
4. `cac_deinit()` and `cgc_deinit()` to understand cleanup.

The demo/reference application exposes:

```c
void cac_example_app_run(void);
```

Use this entry point only when running the supplied terminal demonstration.

The completed measurement count is exposed as:

```c
extern uint32_t g_cac_counter_val;
```

## 6. Reference Board Assumptions

### 6.1 Supported boards in this document

| Board | Documented hardware setup |
|---|---|
| FPB-RA2E3 | Connect the USB debug port to the host PC. Close E25, E28, E46, and E47 to use J-Link OB VCOM for the serial terminal. |
| FPB-RA4E2 | Connect the USB debug port to the host PC. The supplied README specifies no additional board-level connection. |
| FPB-RA6E2 | Connect the USB debug port to the host PC. The supplied README specifies no additional board-level connection. |
| FPB-RA8E1 | Connect the USB debug port to the host PC. The supplied README specifies no additional board-level connection. |

The USB cable type depends on the board model.

### 6.2 FSP-generated instances

The core source assumes that `hal_data.h` provides these configured FSP objects:

- `g_cgc_ctrl`
- `g_cgc_cfg`
- `g_cac_ctrl`
- `g_cac_cfg`

The CAC configuration supplies the reference clock, measurement clock, divisors, lower and upper limits, interrupt configuration, and callback association used by the source.

### 6.3 Clock availability

Before selecting the Main clock or Sub-clock, confirm that the oscillator is available on the selected reference board. The source cannot make unavailable board hardware operational.

Clock frequencies should be checked in the FSP configurator **Clocks** tab or in the user manual for the selected board or MCU.

### 6.4 Terminal settings

The reference application uses the J-Link on-board virtual COM port and expects:

- Speed: 115200 baud
- Data: 8 bits
- Parity: none
- Stop bits: 1
- Flow control: none
- Receive newline: CR
- Transmit newline: CR
- Local echo: enabled

## 7. CAC Clock and Threshold Configuration

The reference and measurement clocks and their divisors are selected in the FSP configurator. The lower and upper limits must be calculated for those selections.

Let:

- `M` be the measurement-clock frequency after its divider.
- `R` be the reference-clock frequency after its divider.
- `C = M / R` be the expected CAC count.
- `T` be the allowed count tolerance.

The supplied README uses a 10% tolerance example:

```text
T = C x 10 / 100
Upper limit = C + T
Lower limit = C - T
```

Its worked example uses:

```text
Measurement clock: HOCO = 24 MHz, divider = 1
M = 24 MHz

Reference clock: Main oscillator = 12 MHz, divider = 32
R = 12 MHz / 32

Expected count:
C = M / R = 64

10% tolerance:
T = 6.4

Upper limit = 70.4
Lower limit = 57.6
```

The limits entered in the configurator must use values supported by the CAC configuration fields. Choose the final integer limits according to the application's required tolerance policy and the applicable FSP/API constraints.

The supplied README warns that an excessively high reference-clock selection can cause repeated measurement-completion interrupts. It recommends using a lower reference-clock frequency or adjusting the reference-clock divider.

## 8. Changes Required for Another Board or Application

### 8.1 Board and pins

The demonstrated CAC measurement uses internal clock selections and does not define an external GPIO pin assignment in the supplied files. If an external reference clock is selected, configure and connect the required external clock input according to the target MCU hardware manual, board schematic, and FSP pin configuration. The supplied files do not specify that board-specific external-clock connection.

Replace the demo UART configuration, VCOM routing, or terminal service if the target board does not use the demonstrated J-Link OB VCOM connection.

### 8.2 Clocks and divisors

For the target board and clock tree:

- verify that each selected oscillator exists and is usable;
- select the reference and measurement clocks in the CAC configuration;
- configure their divisors;
- recalculate lower and upper thresholds; and
- review the CGC settings used to start each selected internal clock.

### 8.3 Timing and timeout behavior

The core uses decrementing loop counters initialized to `UINT16_MAX` for both clock-stability polling and measurement-completion polling. These are iteration-count timeouts, not elapsed-time guarantees. Their actual elapsed duration depends on MCU clock frequency, compiler optimization, execution context, and peripheral behavior.

If deterministic timeout duration is required, replace the polling loops with an application-approved timing mechanism and define a timeout based on the selected clocks and required response time.

### 8.4 Memory

The supplied files do not state measured ROM, RAM, heap, or stack usage. The core stores volatile event/status state and a 32-bit counter globally. The demo additionally uses standard I/O and the UART console service. Measure memory use in the final build configuration if the product has memory limits.

### 8.5 Task priorities and RTOS integration

The supplied reference application runs an endless polling loop and does not define RTOS tasks, priorities, mutexes, semaphores, or event objects. If used with an RTOS, the integrating application must define task ownership, priority, callback-to-task notification, timeout behavior, and synchronization.

Because the callback updates state observed by `cac_measurement_process()`, ensure that the chosen synchronization method is safe for the application's interrupt and task model.

### 8.6 Middleware and console

The reusable CAC core does not depend on the terminal menu. Product code can omit `cac_example_app.*`, `console/uart_console.h`, `printf()`, and `getchar()`.

If the demo is retained, provide the UART console service and stdio routing assumed by the source.

## 9. Dependencies and Hardware Resources

### 9.1 FSP drivers

The reusable core directly depends on:

- CAC driver `r_cac`
- CGC driver `r_cgc`
- FSP-generated configuration in `hal_data.h`

The code calls:

```text
R_CGC_Open
R_CGC_ClockStart
R_CGC_ClockCheck
R_CGC_Close
R_CAC_Open
R_CAC_StartMeasurement
R_CAC_Read
R_CAC_StopMeasurement
R_CAC_Close
```

The demo additionally calls `R_FSP_VersionGet()`.

### 9.2 Demo services

The reference application depends on:

- `console/uart_console.h`
- `uart_console_init()`
- `uart_console_has_char()`
- `uart_console_flush()`
- standard C `printf()` and `getchar()` routing

### 9.3 Hardware resources

The demonstrated configuration requires:

- the selected RA board;
- the CAC peripheral;
- the selected reference and measurement clock sources;
- the CGC for applicable clock startup and status checking;
- CAC interrupt/callback operation;
- a USB debug/programming connection; and
- a UART/VCOM path for the demo application.

## 10. Demonstrated Behavior and Performance

The supplied implementation demonstrates:

- initialization of CGC and CAC FSP drivers;
- startup and stability checking for selected internal clocks;
- interrupt/callback notification of CAC events;
- successful-measurement counter reading;
- frequency-error and overflow event handling;
- loop-count timeout handling; and
- terminal-triggered repeated measurements.

After a successful operation, the demo reports that the selected target clock is accurate and prints `g_cac_counter_val`.

No measured execution time, CAC accuracy, interrupt latency, CPU load, power consumption, ROM usage, RAM usage, or maximum measurement rate is provided in the supplied files. The 10% threshold in the README is a calculation example, not a reported accuracy measurement or a universal tolerance requirement.

## 11. File Classification and Modification Guidance

| File or category | Classification | Guidance |
|---|---|---|
| `cac_ep.h` | Reusable SBB interface | Modify only when intentionally changing the reusable API, shared definitions, or exported state. |
| `cac_ep.c` | Reusable SBB implementation | Modify when changing clock preparation, CAC measurement behavior, timeout handling, callback handling, or error mapping. Keep product-specific policy outside this file where practical. |
| `cac_example_app.h` | Demo/reference interface and terminal text | Demo-only. Safe to replace or remove when the product supplies another application entry point or user interface. |
| `cac_example_app.c` | Demo/reference workflow | Demo-only. Safe to replace or remove when integrating the reusable core into another application. |
| `hal_data.h` | FSP-generated interface | Generated. Do not use it as the primary place for persistent manual configuration changes. |
| FSP configuration and generated source | Generated/preset project content | Change CAC, CGC, clocks, interrupts, and callback settings through the FSP configurator, then regenerate. |
| UART console and stdio integration | Demo support service | Retain only if the product needs the terminal demonstration or equivalent logging. |
| `readme.txt` | Supplied reference documentation | Reference input consolidated by this Markdown document. |

## 12. Adopting a Newer SBB Release After Local Changes

The supplied files do not define an automated upgrade or compatibility process. Use a clear ownership boundary to reduce future merge effort:

- keep product-specific command handling, logging, RTOS logic, and error policy outside `cac_ep.c` and `cac_ep.h` where possible;
- keep board clock, CAC threshold, interrupt, and callback selections in the FSP configuration;
- avoid manually maintaining generated FSP output;
- record unavoidable reusable-core changes as small, focused commits; and
- keep a list of changed public interfaces and behavioral assumptions.

When adopting a newer SBB release:

1. Preserve the current project and generated configuration as a comparison baseline.
2. Compare the new `cac_ep.h` and `cac_ep.c` with the locally modified versions.
3. Reapply only required product changes rather than replacing the new core with the old modified copy.
4. Reconcile the CAC, CGC, clock, interrupt, callback, and threshold configuration in FSP.
5. Regenerate the project configuration.
6. Rebuild and resolve API or generated-instance differences.
7. Revalidate successful completion, frequency-error handling, timeout behavior, counter values, and cleanup on the target hardware.

## 13. Running the Reference Application

1. Connect the board's USB debug port to the host PC.
2. For FPB-RA2E3, close E25, E28, E46, and E47 for J-Link OB VCOM.
3. Verify the configured reference clock, measurement clock, divisors, and CAC limits.
4. Build and program the project.
5. Open the serial terminal using 115200 baud, 8 data bits, no parity, 1 stop bit, and no flow control.
6. Enable CR for receive/transmit newline handling and enable local echo.
7. Run the application.
8. Press any key to perform one CAC measurement.
9. Review the completion message and displayed CAC counter value.

## 14. Integration Summary

For product reuse, begin with `cac_init()` and `cac_measurement_process()` in the core layer. Replace the polling terminal application with the product's own trigger, reporting, synchronization, and error-handling logic. Recalculate CAC thresholds whenever the reference clock, measurement clock, divider, or acceptable tolerance changes, and validate the resulting configuration on the selected board.
