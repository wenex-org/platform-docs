# Thing

**Port:** REST `:3150` · gRPC `:5150`

Internet of Things device management and sensor telemetry. Manages physical devices, their measurement channels (sensors), and the time-series readings (metrics) those sensors produce.

## Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Devices | `/thing/devices` | Physical IoT device or gateway records |
| Sensors | `/thing/sensors` | Measurement channels attached to devices |
| Metrics | `/thing/metrics` | Time-series telemetry readings |

## `thing/devices`

A physical IoT unit or gateway.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `name` | ✅ | string | Human-readable device name |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `type` | string | Device type / category |
| `token` | string | Device communication token |
| `state` | `State` | `PENDING`, `APPROVED`, `REJECTED`, `VERIFIED`, `UNKNOWN` |
| `status` | `Status` | `ACTIVE`, `INACTIVE` |
| `location` | MongoId | Raw `logistic/locations` ID — **not populatable** |

> `thing/devices.location` is a raw MongoId. To resolve location details, read the ID and query `logistic/locations` separately.

## `thing/sensors`

A logical measurement channel attached to a device.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `device` | ✅ | MongoId | Parent device ID |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | Sensor name |
| `type` | string | Sensor category |
| `unit` | string | Measurement unit (e.g. `°C`, `hPa`) |
| `metric` | string | Metric label / key family |
| `state` | `State` | Operational state |
| `status` | `Status` | Administrative status |

### Population

| Path | Collection |
| --- | --- |
| `device` | `thing/devices` |

## `thing/metrics`

A telemetry reading produced by a sensor.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `sensor` | ✅ | MongoId | Source sensor ID |
| `value` | ✅ | `number \| number[]` | Scalar or numeric array reading |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `key` | string | Metric key identifier |
| `state` | `State` | Reading state classification |
| `device` | MongoId | Direct device reference for efficient filtering — returned in responses and populatable |

> `thing/metrics.device` is included in serializer responses and can be populated directly (`populate: [{ path: "device" }]`), or you can populate `sensor` and resolve the device through it.

### Population

| Path | Collection |
| --- | --- |
| `sensor` | `thing/sensors` |
| `device` | `thing/devices` |

## Query Tips

- Always filter metrics by `sensor`, `device`, `key`, and/or a `created_at` time range — metric collections grow quickly.
- Use `count` before paginating high-frequency metric datasets.
- Sort recent metrics by `created_at: desc`.
