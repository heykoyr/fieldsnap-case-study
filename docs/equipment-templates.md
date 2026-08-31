# The equipment taxonomy

The part of FieldSnap that is design work rather than plumbing.

AI extraction is only as good as the schema it is asked to fill. A prompt that says "read this label" returns a wall of text. A prompt that says "return exactly these seven keys for a transformer" returns a record. This file is that set of schemas.

**28 equipment types · 179 fields · 6 categories.**

Each field defines a storage key, a human label, a placeholder drawn from a real nameplate, and a type. The placeholder matters more than it looks: it is what tells a technician which of the four numbers stamped on a plate belongs in that box.

---

## Electrical Panels

6 types · 39 fields

| Equipment type | Fields | Captured |
|---|---|---|
| ⚡ **Distribution Panel** | 7 | Panel ID · Breaker Number · Circuit Label · Amperage · Voltage · Phase · Manufacturer |
| 🔄 **Transformer** | 7 | Unit ID · Primary Voltage · Secondary Voltage · KVA Rating · Phase · Serial Number · Manufacturer |
| 🔲 **Breaker Box** | 6 | Box ID · Number of Circuits · Main Breaker Rating · Voltage · Phase · Location |
| 🔋 **Generator** | 7 | Unit ID · KW Rating · Fuel Type · Serial Number · Last Service Date · Runtime Hours · Manufacturer |
| 🔌 **Switch Gear** | 6 | Gear ID · Voltage Rating · Current Rating · Number of Panels · Manufacturer · Serial Number |
| 🎛️ **Control Panel** | 6 | Panel ID · Control Type · Input Voltage · Output Voltage · IP Rating · Manufacturer |

## HVAC Equipment

6 types · 39 fields

| Equipment type | Fields | Captured |
|---|---|---|
| 💨 **Air Handling Unit** | 7 | Unit ID · CFM Rating · Filter Size · Coil Type · Motor HP · Serial Number · Location/Zone |
| 🧊 **Chiller** | 7 | Unit ID · Tonnage · Refrigerant Type · COP/EER · Chilled Water Flow · Serial Number · Manufacturer |
| 🏭 **Cooling Tower** | 6 | Unit ID · Capacity (Tons) · Fan HP · Water Flow Rate · Fill Type · Serial Number |
| 📦 **VAV Box** | 6 | Box ID · Zone/Room · Min CFM · Max CFM · Actuator Type · Reheat Coil |
| 🏠 **Rooftop Unit** | 7 | Unit ID · Tonnage · SEER Rating · Refrigerant · Gas Heat (BTU) · Serial Number · Manufacturer |
| 🌀 **Fan Coil Unit** | 6 | Unit ID · CFM · Coil Type · Filter Size · Motor Speed · Location |

## Power Systems

5 types · 32 fields

| Equipment type | Fields | Captured |
|---|---|---|
| 🔌 **UPS System** | 7 | Unit ID · KVA Rating · Battery Type · Runtime (mins) · Input Voltage · Output Voltage · Serial Number |
| 🔀 **Transfer Switch** | 6 | Switch ID · Amperage · Voltage · Type · Transfer Time · Serial Number |
| 🔋 **Power Distribution Unit** | 6 | PDU ID · Input Voltage · Output Circuits · Amperage · Outlet Type · Serial Number |
| ⚡ **Capacitor Bank** | 6 | Bank ID · KVAR Rating · Voltage · Phases · Steps · Serial Number |
| ☀️ **Solar Inverter** | 7 | Unit ID · KW Rating · DC Input Voltage · AC Output Voltage · Efficiency · Serial Number · Manufacturer |

## Motors & Drives

4 types · 26 fields

| Equipment type | Fields | Captured |
|---|---|---|
| ⚙️ **Electric Motor** | 7 | Motor ID · HP Rating · RPM · Voltage · Frame Size · Insulation Class · Serial Number |
| 📻 **Variable Frequency Drive** | 7 | Drive ID · HP/KW Rating · Input Voltage · Output Frequency · Control Mode · Serial Number · Manufacturer |
| 💧 **Pump** | 6 | Pump ID · Flow Rate (GPM) · Head Pressure · HP · Pump Type · Serial Number |
| 🔩 **Compressor** | 6 | Unit ID · PSI Rating · CFM Output · HP · Refrigerant · Serial Number |

## Telecom / Data

4 types · 25 fields

| Equipment type | Fields | Captured |
|---|---|---|
| 🖥️ **Server Rack** | 6 | Rack ID · Unit Height (U) · Power Draw (kW) · Location/Row · PDU Type · Cooling Type |
| 🔌 **Patch Panel** | 6 | Panel ID · Port Count · Cable Type · Location · Rack Unit Position · Label Scheme |
| 🌐 **Network Switch** | 7 | Device ID · Port Count · Speed · Management IP · VLAN Config · Serial Number · Manufacturer |
| 📷 **CCTV System** | 6 | System ID · Camera Count · Resolution · DVR/NVR Model · Storage (TB) · Retention (days) |

## Others

3 types · 18 fields

| Equipment type | Fields | Captured |
|---|---|---|
| 🚨 **Fire Alarm Panel** | 6 | Panel ID · Model Number · Number of Zones · Supply Voltage · Battery Backup · Manufacturer |
| 📷 **Security Camera** | 6 | Camera ID · Model Number · Resolution · Lens (mm) · IR Night Range · Manufacturer |
| 🛗 **Elevator Controller** | 6 | Unit ID · Model Number · Number of Floors · Capacity · Drive Type · Manufacturer |

---

## A worked example

The **Transformer** template, as the model receives it:

| Field | Placeholder |
|---|---|
| Unit ID | e.g. TR-01 |
| Primary Voltage | e.g. 4160V |
| Secondary Voltage | e.g. 480V |
| KVA Rating | e.g. 500 KVA |
| Phase | e.g. 3-Phase |
| Serial Number | e.g. SN-2948301 |
| Manufacturer | e.g. ABB |

The extraction prompt is built from these exact keys and constrained to return JSON containing them and nothing else. Where a value is not clearly legible on the plate the model returns an empty string rather than a guess, and the field is left for the technician to fill.

---

## Custom fields

The 28 templates are a starting point, not a ceiling. A technician can add fields to any template from Settings. Custom fields are stored per account, rendered into the scan form alongside the built-in fields, and written into the exported PDF. A commissioning engineer who needs a torque-check column on every panel adds it once.

Custom fields are filled by hand, not by the model: the extraction prompt is built from the built-in template fields only. Sending user-authored labels straight into the prompt would let a vague field name ("Notes", "Check") pull hallucinated values into a record that is supposed to be a faithful transcription of a plate. Manual entry for custom fields is the conservative default, and the right one until there is a way to tell a good custom label from a bad one.

---

## Where it came from

Desk research: manufacturer nameplate documentation and specification sheets across the six categories, plus trade forums where electricians, HVAC technicians and facilities engineers discuss what actually gets recorded on a commissioning or asset survey — and what gets missed.

This was not validated through field interviews. The taxonomy is a researched hypothesis about what these trades need to capture, shipped so that use can correct it.
