# JKLink EnergyTwin EMS --- Architecture Concept

## 1. Purpose

This document defines the concept architecture for a local, AI-assisted
Energy Management System (EMS) integrating:

-   Victron EasySolar-II and MultiPlus-II 5 kVA equipment
-   \~9 kWp PV generation
-   \~34 kWh nominal 15S NMC battery
-   JK BMS + JKLink adaptive Battery Twin
-   Victron EVCS 7 kW
-   Chofu AEYC-1049ZU-CH1 R290 10 kW heat pump
-   100 L heating buffer/puffer
-   Radiator circuit with external circulation pump
-   Chofu CMR-4100M room controller in the bedroom
-   Raspberry Pi + Home Assistant
-   InfluxDB + Grafana
-   Local AI/ML models and model-predictive control (MPC)

The design objective is to optimize electrical and thermal energy use
while preserving comfort, battery life, heat-pump efficiency and safe
autonomous operation.

------------------------------------------------------------------------

## 2. Design Principles

1.  **Local-first:** core operation continues without cloud services.
2.  **Safety hierarchy:** Victron, JK BMS and Chofu retain their native
    protection/control functions.
3.  **Supervisory AI:** AI predicts and optimizes; it does not directly
    bypass inverter, BMS or heat-pump safety logic.
4.  **Single decision authority per domain:**
    -   Victron GX/ESS: electrical/inverter authority.
    -   JKLink: battery intelligence and battery-side adaptive control.
    -   Chofu: compressor, defrost, hydraulic and thermal safety.
    -   Raspberry Pi EMS: cross-domain optimization.
5.  **Human comfort remains authoritative:** the bedroom CMR-4100M
    provides the practical room-temperature interface.
6.  **Fail-safe degradation:** loss of Raspberry Pi, MQTT or network
    must not stop normal heating protection.
7.  **Explainable automation:** every EMS action should have a reason
    and measured/predicted inputs.

------------------------------------------------------------------------

## 3. High-Level Architecture

``` mermaid
flowchart TB
    PV["~9 kWp PV"]
    GX["Victron GX / ESS"]
    MP1["MultiPlus-II 5 kVA"]
    ES["EasySolar-II 5 kVA"]
    EV["Victron EVCS 7 kW"]
    BAT["34 kWh 15S NMC Battery"]
    BMS["JK BMS"]
    JKL["JKLink Battery Twin"]
    PI["Raspberry Pi EMS"]
    HA["Home Assistant"]
    DB["InfluxDB"]
    G["Grafana"]
    AI["Thermal Twin + PV/Load Forecast + MPC"]
    CMR["CMR-4100M<br/>Bedroom"]
    CH["Chofu AEYC-1049ZU-CH1 R290"]
    BUF["100 L Puffer"]
    RP["Radiator Circulation Pump"]
    SENS["Additional Room / Outdoor Sensors"]

    PV --> GX
    GX --> MP1
    GX --> ES
    GX --> BAT
    GX --> EV

    BAT <--> BMS
    BMS <--> JKL

    GX <--> PI
    JKL <--> PI
    PI <--> HA
    HA --> DB
    DB --> G
    PI <--> AI

    CMR <--> CH
    SENS --> PI

    PI <--> CH
    CH --> BUF
    BUF --> RP
    PI --> RP
```

------------------------------------------------------------------------

## 4. Functional Domains

### 4.1 Electrical Energy Domain

The Victron system provides the authoritative electrical measurements
and ESS operation.

Primary signals:

-   PV power
-   AC load power
-   grid import/export
-   battery power
-   battery SOC
-   inverter state
-   charger state
-   EVCS power
-   system voltage/frequency
-   alarms

The EMS should preferably communicate with the GX environment through
supported local interfaces such as MQTT, Modbus TCP and/or Node-RED
rather than manipulating inverter hardware directly.

### 4.2 Battery Domain

JKLink remains the battery intelligence layer.

Signals:

-   cell minimum/maximum voltage
-   cell delta voltage
-   pack voltage/current
-   battery temperature
-   BMS SOC
-   charge/discharge limits
-   adaptive SOC estimation
-   battery twin predictions
-   estimated degradation / SOH indicators

The EMS uses the Battery Twin to estimate the value and constraints of
storing another kWh electrically.

### 4.3 Thermal Domain

The Chofu remains responsible for heat-pump operation.

Signals to collect:

-   heat-pump electrical power
-   leaving-water temperature
-   return-water temperature
-   buffer temperature
-   heating setpoint
-   operating mode
-   compressor state/frequency where available
-   outdoor temperature
-   alarms
-   SG Ready / external-control state where available

The 100 L puffer is treated as a small thermal battery.

### 4.4 Comfort Domain

The CMR-4100M is placed in the bedroom, where the room temperature is
representative of the desired comfort zone.

The CMR is not used as the master EMS.

The EMS reads room temperature and target temperature and determines
whether the radiator circuit should circulate.

Additional inexpensive room sensors can be added later for multi-room
learning.

------------------------------------------------------------------------

## 5. Control Hierarchy

``` text
                         SAFETY
                           │
        ┌──────────────────┼───────────────────┐
        │                  │                   │
     Victron              JK BMS             Chofu
   ESS protection       battery safety    compressor/defrost
        │                  │                   │
        └──────────────────┼───────────────────┘
                           │
                    SUPERVISORY EMS
                           │
             ┌─────────────┼─────────────┐
             │             │             │
          Comfort       Energy        Thermal
          demand       optimizer       storage
             │             │             │
             └─────────────┼─────────────┘
                           │
                         MPC
                           │
              ┌────────────┼────────────┐
              │            │            │
           Chofu        Radiator       EVCS
          demand          pump       charging
```

The EMS should not directly control compressor internals.

It should issue high-level requests such as:

-   normal heating
-   thermal charging / solar boost
-   heating permission
-   radiator circulation request
-   EV charging target/current
-   battery/ESS optimization request

------------------------------------------------------------------------

## 6. Thermal Battery Concept

The puffer is treated as controllable thermal storage.

Approximate water-storage capacity:

**100 L × 4.186 kJ/kg/K ≈ 0.116 kWh per °C**

Therefore:

    Puffer temperature span   Approx. stored energy
  ------------------------- -----------------------
                       5 °C                0.58 kWh
                      10 °C                1.16 kWh
                      15 °C                1.74 kWh
                      20 °C                2.32 kWh
                      25 °C                2.90 kWh

The EMS should not automatically maximize puffer temperature. It should
determine the minimum useful temperature that maintains predicted
comfort and takes advantage of available solar energy.

------------------------------------------------------------------------

## 7. Energy Allocation Strategy

The EMS considers three flexible storage destinations:

1.  Electrical battery
2.  Thermal buffer
3.  EV battery

and one immediate consumer:

4.  House load

Conceptually:

``` text
                   PV SURPLUS
                       │
                       ▼
                ┌─────────────┐
                │ EMS / MPC   │
                └──────┬──────┘
                       │
       ┌───────────────┼────────────────┐
       │               │                │
       ▼               ▼                ▼
   NMC Battery      Chofu/Puffer      EVCS
   electrical       thermal storage   vehicle
   storage          storage
```

The decision should consider:

-   current SOC
-   predicted PV
-   predicted household load
-   battery charge/discharge limits
-   battery degradation cost
-   current room temperature
-   predicted room temperature
-   puffer temperature
-   outdoor temperature
-   expected heating demand
-   EV departure/charging requirement
-   grid conditions
-   heat-pump efficiency at the required water temperature

------------------------------------------------------------------------

## 8. AI / Machine Learning Architecture

The system should use multiple specialized models rather than one
monolithic AI model.

### Model A --- Battery Twin

Existing JKLink AI Twin.

Inputs:

-   cell voltages
-   delta cell voltage
-   current
-   voltage
-   temperature
-   SOC history
-   charge/discharge history

Outputs:

-   adaptive SOC
-   charge acceptance estimate
-   discharge capability
-   anomaly indicators
-   battery state estimates

### Model B --- House Thermal Twin

New model trained locally from time-series data.

Inputs:

-   bedroom temperature
-   other room temperatures
-   outdoor temperature
-   puffer temperature
-   flow/return temperature
-   radiator-pump state
-   Chofu power
-   Chofu operating state
-   weather/solar information
-   time of day

Outputs:

-   room temperature after 1 h
-   room temperature after 3 h
-   room temperature after 6 h
-   heating energy required
-   estimated thermal inertia
-   estimated building heat-loss coefficient

### Model C --- PV / Load Forecast

Inputs:

-   historical PV
-   weather forecast
-   time/day/season
-   household load history
-   EV charging history

Outputs:

-   expected PV generation
-   expected house load
-   expected surplus

### Model D --- Optimization/MPC

MPC is preferred as the initial decision engine because it is
constrained and explainable.

Objective:

``` text
minimize:

  energy cost
+ battery degradation cost
+ comfort deviation
+ heat-pump cycling penalty
+ unnecessary thermal storage
+ unnecessary grid import
```

Subject to:

-   battery SOC limits
-   BMS charge/discharge limits
-   Chofu operating limits
-   puffer temperature limits
-   comfort bounds
-   EV charging requirements
-   grid import/export policy
-   actuator constraints

------------------------------------------------------------------------

## 9. Radiator Pump Control

The external radiator pump should not be directly controlled by a
conventional thermostat relay.

The EMS should calculate a heating-demand state.

Example:

``` text
if Troom < target - comfort_deadband:
    heating_demand = ON

elif Troom >= target:
    evaluate predicted cooling rate

if puffer is below minimum useful temperature:
    heating_demand = OFF

if room is expected to remain above target:
    heating_demand = OFF
```

A proportional demand can be added later:

``` text
heating_demand = f(
    room_temperature_error,
    room_cooling_rate,
    puffer_temperature,
    outdoor_temperature,
    predicted_room_temperature
)
```

A physical relay/contactor or suitable pump-control interface should be
used as the final actuator.

------------------------------------------------------------------------

## 10. CMR-4100M Role

Recommended role:

-   bedroom temperature measurement
-   human-adjustable target temperature
-   local Chofu room-control interface
-   native Chofu user interaction

The CMR-4100M should not become the cross-domain energy optimizer.

Concept:

``` text
Bedroom
   │
   ▼
CMR-4100M
   │
   ├── actual temperature
   └── target temperature
          │
          ▼
       Chofu
          │
          ▼
        Puffer
          │
          ▼
      Radiators
          ▲
          │
      EMS pump command
```

This keeps the user experience simple while preserving centralized
optimization.

------------------------------------------------------------------------

## 11. Chofu Integration Strategy

### Phase 1 --- Read-only

Collect:

-   water temperatures
-   buffer temperature
-   operating mode
-   compressor state
-   alarms
-   setpoints
-   electrical consumption where available

Do not write experimental registers.

### Phase 2 --- Validated high-level control

Use documented interfaces such as:

-   SG Ready
-   supported external ON/OFF
-   supported thermostat/control functions

### Phase 3 --- R290 protocol reverse engineering

Map the R290 controller's registers/parameters and validate each
register against physical measurements.

Every writable parameter should have:

-   parameter ID
-   Modbus/register address
-   scaling
-   unit
-   valid range
-   read/write capability
-   safe default
-   test procedure

Never infer a writable register solely from the older R32 Chofu maps.

------------------------------------------------------------------------

## 12. Data Architecture

InfluxDB should become the long-term learning dataset.

Recommended measurement groups:

``` text
victron/
  pv_power
  load_power
  grid_power
  battery_power
  battery_soc
  evcs_power

battery/
  pack_voltage
  pack_current
  cell_min
  cell_max
  cell_delta
  temperature
  adaptive_soc
  charge_limit
  discharge_limit

chofu/
  electrical_power
  flow_temperature
  return_temperature
  buffer_temperature
  setpoint
  mode
  compressor_state

house/
  bedroom_temperature
  bedroom_target
  outdoor_temperature
  radiator_pump
  humidity

prediction/
  pv_1h
  pv_3h
  load_1h
  room_1h
  room_3h
  room_6h

ems/
  selected_mode
  decision_reason
  confidence
  thermal_charge_request
  ev_charge_request
```

------------------------------------------------------------------------

## 13. Grafana / HA Dashboard

The main dashboard should expose both current state and the reason
behind automation.

Example:

``` text
ENERGY TWIN
────────────────────────────────────
PV                 5.8 kW
House              1.4 kW
Battery             94%
Battery power      +2.1 kW
EVCS                0.0 kW

BEDROOM            21.1 °C
TARGET             21.5 °C
OUTDOOR              4.2 °C

PUFFER TOP         47 °C
PUFFER BOTTOM      34 °C
CHOFU FLOW         42 °C
CHOFU RETURN       34 °C

PREDICTED ROOM
+1h                21.4 °C
+3h                21.0 °C
+6h                20.6 °C

EMS STATE
SOLAR THERMAL CHARGE

REASON
Battery > 90%
+ PV surplus > 2 kW
+ room below target
+ puffer has available thermal capacity
────────────────────────────────────
```

------------------------------------------------------------------------

## 14. Recommended Operating Modes

### COMFORT

Maintain the user-selected room temperature.

### ECO

Allow a larger temperature deadband and minimize energy use.

### SOLAR

Use available PV surplus to increase useful thermal storage within
configured limits.

### BOOST

Temporary user-requested heating.

### AWAY

Allow a lower comfort limit while maintaining frost/system protection.

------------------------------------------------------------------------

## 15. Fault and Fallback Strategy

The system must remain functional if the Raspberry Pi fails.

### Raspberry Pi offline

-   Chofu continues under its native controller.
-   CMR-4100M remains available.
-   Victron ESS continues independently.
-   JK BMS protection remains active.
-   Radiator pump falls back to a predefined safe control mode.

### MQTT unavailable

Local device control remains available.

### AI model unavailable

Use deterministic rule-based control.

### Chofu communication unavailable

Do not attempt experimental writes; continue native Chofu control.

### Sensor invalid

Reject the value and use the last validated value or safe fallback.

------------------------------------------------------------------------

## 16. Implementation Phases

### Phase 1 --- Telemetry foundation

-   Standardize Victron measurements.
-   Standardize JKLink measurements.
-   Integrate Chofu read-only telemetry.
-   Add bedroom and outdoor sensors.
-   Establish unified timestamps and units.
-   Create Grafana EMS dashboard.

### Phase 2 --- Thermal learning

Collect several weeks of heating data.

Learn:

-   room thermal inertia
-   heat-loss behavior
-   radiator response
-   puffer response
-   Chofu electrical/thermal behavior

### Phase 3 --- Rule-based EMS

Before AI optimization:

-   comfort control
-   puffer limits
-   solar surplus detection
-   battery SOC constraints
-   EV charging priority
-   radiator pump control

### Phase 4 --- Predictive thermal model

Deploy the House Thermal Twin.

Validate prediction error against real temperature.

### Phase 5 --- MPC optimizer

Optimize:

-   battery storage
-   thermal storage
-   EV charging
-   comfort

### Phase 6 --- Adaptive EnergyTwin

Combine:

-   Battery Twin
-   Thermal Twin
-   PV forecast
-   Load forecast
-   MPC

### Phase 7 --- Chofu closed-loop optimization

Only after validation:

-   SG Ready control
-   high-level setpoint optimization
-   validated R290 parameter control

------------------------------------------------------------------------

## 17. Recommended Hardware Architecture

### Keep

-   Victron EasySolar-II
-   2 × MultiPlus-II 5 kVA
-   Victron GX communication
-   9 kWp PV
-   34 kWh NMC battery
-   JK BMS
-   JKLink
-   Victron EVCS 7 kW
-   Raspberry Pi
-   Home Assistant
-   InfluxDB
-   Grafana
-   Chofu AEYC-1049ZU-CH1
-   100 L puffer

### Add

-   Chofu CMR-4100M in the bedroom
-   independent room-temperature sensor(s)
-   outdoor-temperature sensor if required
-   properly rated radiator-pump relay/contactor
-   optional isolated RS485 interface for development/reverse
    engineering

### Do not initially add

-   Salus IT800 as another heating controller
-   a second complete AI/EMS platform
-   direct compressor control

------------------------------------------------------------------------

## 18. Target System

The final system should behave like this:

``` text
                     SOLAR
                       │
                       ▼
               ┌──────────────┐
               │  VICtron ESS │
               └──────┬───────┘
                      │
        ┌─────────────┼──────────────┐
        ▼             ▼              ▼
     Battery        House           EV
       │
       ▼
  JKLink Battery
      Twin
       │
       └──────────────┐
                      ▼
              ┌───────────────┐
              │ Raspberry Pi  │
              │ EnergyTwin    │
              │ EMS + MPC     │
              └───────┬───────┘
                      │
          ┌───────────┼────────────┐
          ▼           ▼            ▼
       CHOFU       Pump relay     EVCS
          │
          ▼
       100 L
       Puffer
          │
          ▼
       Radiators
          ▲
          │
     CMR-4100M
     Bedroom
```

## 19. Core Innovation

The system should not ask:

> "Should I turn the heat pump on?"

It should ask:

> **"Given predicted solar production, house demand, battery state, EV
> requirements, thermal storage and comfort, where should the next
> available kWh go?"**

That transforms the installation from a collection of smart devices into
a coordinated **Electrical + Battery + Thermal Energy Twin**.

The final control chain becomes:

**Measure → Predict → Optimize → Actuate → Learn → Repeat.**

------------------------------------------------------------------------

## 20. Useful Reference Projects

-   Chofu/Grant Aerona Modbus reverse-engineering:
    https://github.com/aerona-chofu-ashp/modbus
-   Victron GX Modbus TCP:
    https://www.victronenergy.com/live/ccgx%3Amodbustcp_faq
-   Victron Node-RED integration:
    https://github.com/victronenergy/node-red-contrib-victron
-   Victron heat-pump control project:
    https://github.com/victronenergy/dbus-heatpump-control
-   OpenEnergyMonitor Grant R290 reverse-engineering discussion:
    https://community.openenergymonitor.org/t/grant-r290-econet-mqtt-publisher/29121
