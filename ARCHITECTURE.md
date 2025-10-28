# LibreHardwareMonitor Architecture Overview

LibreHardwareMonitor is split into a reusable hardware-monitoring library and a Windows Forms
front-end. The solution also contains supporting utilities and a WMI bridge that makes sensor
data accessible to external tooling.

## Solution Layout

| Project / Folder | Purpose |
|------------------|---------|
| `LibreHardwareMonitorLib` | Core library that discovers hardware, reads sensors, and exposes a uniform object model. |
| `LibreHardwareMonitor` | Desktop UI that visualizes sensors, manages user interaction, and hosts auxiliary services (web server, tray icon, gadgets). |
| `LibreHardwareMonitor/WMI` | Lightweight wrapper that publishes a subset of the sensor tree through WMI. |
| `Aga.Controls` | Third-party UI components used by the Windows Forms interface. |

## Core Library (`LibreHardwareMonitorLib`)

The library models the system as a tree of *groups*, *hardware*, and *sensors*.

- `Hardware.Computer` is the central façade. It maintains the enabled hardware groups
  (CPU, GPU, storage, controllers, battery, etc.) and coordinates their life cycle,
  raising `HardwareAdded`/`HardwareRemoved` events for consumers. Each group maps to a
  platform-specific implementation such as `CpuGroup`, `NvidiaGroup`, or `BatteryGroup`.
- Hardware components implement `IHardware` and own a collection of `ISensor`
  instances. Sensors represent live metrics (temperature, voltage, load, fan speed,
  throughput) and are refreshed via the visitor pattern (`IVisitor`, `ISensorVisitor`).
- The library uses `ISettings` to share persistent configuration across devices. It also
  provides abstractions for parameters (`IParameter`), controls (`IControl`), and
  composite sensors that aggregate multiple readings.
- `Interop` contains P/Invoke wrappers for platform APIs (e.g., SMBIOS, NVAPI, AMD ADL)
  that feed raw telemetry into the sensor model.
- `Resources` bundles firmware tables, microcode definitions, and localized strings used
  across the hardware implementations.

## Desktop Application (`LibreHardwareMonitor`)

The UI project hosts the Windows Forms application defined in `Program.cs`.

- `UI/MainForm` builds the primary tree view of hardware and sensors, provides graphing
  via OxyPlot, and exposes context menus for enabling, renaming, and configuring sensors.
  It relies on helper classes such as `HardwareNode`, `SensorNode`, and `TypeNode` to map
  the domain model into a hierarchical UI.
- Tray integration, sensor gadgets, and desktop overlays live under `UI/SystemTray.cs`,
  `UI/SensorGadget.cs`, and related classes. These components subscribe to library events
  to stay synchronized with the `Computer` instance.
- The `Utilities` folder offers cross-cutting services: `PersistentSettings` persists
  window layout and sensor preferences, `Logger` / `LoggerFileRotation` implement rolling
  log output, `HttpServer` provides the optional remote monitoring endpoint, and
  `EmbeddedResources` loads bundled themes and icons.
- `WMI/WmiProvider` mirrors the sensor hierarchy into Windows Management Instrumentation,
  allowing third-party scripts to query current values without accessing the UI.

## Data Flow

1. `Program.Main` starts the application, ensuring required assemblies are present before
   launching `MainForm`.
2. `MainForm` constructs a `Computer` instance, enabling hardware categories according to
   persisted settings. The form registers visitors that periodically update sensors and
   repopulate the tree view.
3. When new hardware is detected, UI nodes and background services (tray icon, HTTP API,
   WMI provider) react to `HardwareAdded` events to attach their respective renderers or
   data publishers.
4. Sensor updates propagate to bound views and exporters. The visitor pattern ensures the
   UI, logging, and remote endpoints can iterate over the current hierarchy without
   tightly coupling to specific hardware implementations.

## Extensibility

New hardware support typically involves adding a group under
`LibreHardwareMonitorLib/Hardware` that implements `IGroup` and produces one or more
`IHardware` instances. Each hardware exposes `ISensor` objects describing available
metrics. Once compiled, the UI automatically renders the new sensors because it consumes
only the shared abstractions.
