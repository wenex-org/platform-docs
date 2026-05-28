# Logistic

**Port:** REST `:3120` · gRPC `:5120`

Tracks physical assets and movements — geographic locations, drivers, vehicles, cargoes, and travel routes. Integrates with OpenStreetMap for geocoding and the Valhalla engine for routing.

## Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Locations | `/logistic/locations` | GeoJSON places and areas |
| Drivers | `/logistic/drivers` | Driver records |
| Vehicles | `/logistic/vehicles` | Fleet vehicle records |
| Travels | `/logistic/travels` | Ordered trip records linking locations, drivers, vehicles, and cargoes |
| Cargoes | `/logistic/cargoes` | Shipment dimension and handling data |

## `logistic/locations`

GeoJSON-based physical places or areas (warehouses, depots, customer sites, checkpoints).

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `geometry` | ✅ | object | GeoJSON geometry |

### Geometry Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `LocationGeometryType` | `Point`, `MultiPoint`, `LineString`, `MultiLineString`, `Polygon`, `MultiPolygon` |
| `coordinates` | ✅ | array | GeoJSON coordinates — **`[longitude, latitude]` order** |

> GeoJSON uses `[longitude, latitude]` — not `[latitude, longitude]`. This order is safety-critical.

### Special Operations (platform REST only)

| Operation | HTTP | Scope | Description |
| --- | --- | --- | --- |
| Reverse geocode | `POST /logistic/locations/resolve/address` | `resolve:logistic:locations` | Coordinates → address |
| Forward geocode | `POST /logistic/locations/resolve/geocode` | `resolve:logistic:locations` | Text query → coordinates |

## `logistic/drivers`

Driver identity and operational status.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `DriverType` | `CLASS_A`, `CLASS_B`, `CLASS_C`, `CLASS_M`, `CLASS_E` |
| `gender` | ✅ | `Gender` | `MALE`, `FEMALE`, `UNKNOWN` |
| `state` | ✅ | `State` | `PENDING`, `APPROVED`, `REJECTED`, `VERIFIED`, `UNKNOWN` |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |
| `license` | ✅ | string | License number |
| `expiration_date` | ✅ | date | License expiration date |

## `logistic/vehicles`

Fleet vehicle registry with assigned driver linkage.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `VehicleType` | `CAR`, `TRUCK`, `MOTORCYCLE` |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |
| `plates` | ✅ | string[] | Plate numbers |

### Population

| Path | Collection |
| --- | --- |
| `drivers` | `logistic/drivers` |

## `logistic/travels`

Ordered trip record linking route locations, cargoes, drivers, and vehicles.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `locations` | ✅ | MongoId[] | Ordered location IDs — first is origin, last is destination |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `cargoes` | MongoId[] | Assigned cargo IDs |
| `drivers` | MongoId[] | Assigned driver IDs |
| `vehicles` | MongoId[] | Assigned vehicle IDs |

### Population

| Path | Collection |
| --- | --- |
| `locations` | `logistic/locations` |
| `cargoes` | `logistic/cargoes` |
| `drivers` | `logistic/drivers` |
| `vehicles` | `logistic/vehicles` |

### Special Operation (platform REST only)

| Operation | HTTP | Scope | Description |
| --- | --- | --- | --- |
| Routing | `POST /logistic/travels/resolve/routing` | `resolve:logistic:travels` | Route planning via Valhalla |

Routing requires `service` (`route`, `isochrone`, `sources_to_targets`, `optimized_route`) and `options` (raw Valhalla payload).

## `logistic/cargoes`

Physical shipment metadata, dimensions, and handling flags.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `weight` | ✅ | number | Cargo weight |
| `width` | ✅ | number | Cargo width |
| `height` | ✅ | number | Cargo height |
| `length` | ✅ | number | Cargo length |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `title` | string | Human-readable label |
| `fragile` | boolean | Fragile handling flag |
| `perishable` | boolean | Perishable handling flag |
| `travels` | MongoId[] | Linked travel IDs |

### Population

| Path | Collection |
| --- | --- |
| `travels` | `logistic/travels` |
