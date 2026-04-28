# GreyCat GCL Language Guide

GreyCat (https://greycat.io) is a graph database AND programming language in one.
Files use `.gcl` extension. Docs: https://doc.greycat.io

## Core Concepts

| Concept | Description |
|---------|-------------|
| `@node` | Persistent graph node (stored in DB) |
| `nodeIndex<T>` | Indexed collection of nodes, auto-persisted |
| `nodeList<T>` | Ordered list of nodes |
| `nodeTime<T>` | Time-series node collection |
| `nodeGeo<T>` | Geo-indexed node collection |
| `@expose` | Expose function as HTTP endpoint |
| `@mcp` | Expose function as MCP tool |
| `@permission` | Access control decorator |
| `@volatile` | Non-persisted field |

## Type Definitions

```gcl
@node
type Country {
  name: String;
  code: String;
  cities: nodeIndex<City>;    // contains cities
}

@node
type City {
  name: String;
  lat: float;
  lon: float;
  country: Country;           // back reference
  population: int;
}

@node
type SensorReading {
  value: float;
  timestamp: time;
}
```

## Collections

```gcl
// nodeIndex — unique key lookup
var countries: nodeIndex<Country>;
var c = countries.get("LU") ?? countries.getOrCreate("LU");
c.name = "Luxembourg";

// nodeList — ordered, by insertion
var log: nodeList<SensorReading>;
log.add(reading);

// nodeTime — time-series (time key)
var history: nodeTime<SensorReading>;
history.set(reading.timestamp, reading);
// Query range:
history.iterateFrom(start, end, fn (t: time, v: SensorReading) {
  // process each reading
});
```

## Functions & API Exposure

```gcl
// HTTP GET /api/countries
@expose
fn getCountries(): Array<Country> {
  return countries.all();
}

// HTTP POST /api/country (body: {name, code})
@expose
fn createCountry(name: String, code: String): Country {
  var c = countries.getOrCreate(code);
  c.name = name;
  return c;
}

// MCP tool
@mcp
fn findCity(name: String): City? {
  return cities.findBy("name", name);
}

// With permission
@expose
@permission("admin")
fn deleteAll(): void {
  countries.clear();
}
```

## Importing CSV Data

```gcl
use io;

fn importAddresses(): void {
  io.CSV.read("./data/addresses.csv", fn (row: Array<String>) {
    var house = houses.getOrCreate(row[0]);
    house.street = row[1];
    house.lat = float::parse(row[2]);
    house.lon = float::parse(row[3]);
  });
}

// Call on main if DB is empty
fn main(): void {
  if (countries.size() == 0) {
    importAddresses();
    importTemperatures();
  }
}
```

## Concurrency (Jobs)

```gcl
use runtime;

// Run in background job
fn runAnalysis(): void {
  var job = runtime.Job.create(fn () {
    // long-running computation
    computeOptimalFlow();
  });
  job.start();
}

// Await result
fn syncCall(): float {
  var result = await computeAsync();
  return result;
}
```

## Standard Libraries

| Library | Import | Key Features |
|---------|--------|--------------|
| `core` | built-in | String, Array, Map, int, float, time |
| `io` | `use io;` | CSV, JSON, file I/O |
| `runtime` | `use runtime;` | Jobs, scheduling, system info |
| `util` | `use util;` | Math, sorting, UUID |
| `ai` | `use ai;` | ML models, inference (Pro) |
| `algebra` | `use algebra;` | Matrices, linear algebra (Pro) |
| `sql` | `use sql;` | SQL connector (Pro) |
| `kafka` | `use kafka;` | Kafka streams (Pro) |
| `s3` | `use s3;` | AWS S3 (Pro) |
| `powerflow` | `use powerflow;` | Power grid analysis (Pro) |

## Frontend Integration (@greycat/web SDK)

```typescript
import { GreyCat } from '@greycat/web';

const greycat = await GreyCat.init({ url: 'http://localhost:8080' });

// Call exposed functions
const countries = await greycat.call('getCountries');
const city = await greycat.call('findCity', ['Luxembourg']);

// Subscribe to real-time updates
greycat.subscribe('sensorUpdates', (data) => {
  console.log(data);
});
```

## Project Setup

```bash
# Install GreyCat (Windows)
iwr https://get.greycat.io/install_dev.ps1 -useb | iex

# Install GreyCat (Linux/Mac)
curl -fsSL https://get.greycat.io/install.sh | bash -s dev

# Restart terminal after install, then:
greycat --version
greycat-lang --version   # LSP server

# Initialize project
mkdir my-project && cd my-project
greycat init             # Creates project.gcl, package.json

# Start development server
greycat serve            # Starts on http://localhost:8080
```

## Typical Full-Stack Project Structure

```
my-project/
├── project.gcl           # Entry point with main() fn
├── model/
│   ├── geo.gcl           # Country, City, Street, House nodes
│   └── sensors.gcl       # SensorReading, Device nodes
├── api/
│   ├── geo-api.gcl       # @expose geo query functions
│   └── sensor-api.gcl    # @expose sensor data functions
├── data/
│   ├── seed.csv          # Initial data
│   └── importer.gcl      # CSV import logic
└── frontend/
    ├── index.html
    └── src/
        └── app.ts        # @greycat/web SDK usage
```

## Example: Geographic Hierarchy with Time Series

```gcl
@node
type Country {
  name: String;
  cities: nodeIndex<City>;
}

@node
type City {
  name: String;
  lat: float;
  lon: float;
  country: Country;
  streets: nodeIndex<Street>;
}

@node
type House {
  id: String;
  number: int;
  lat: float;
  lon: float;
  street: Street;
  temperatures: nodeTime<float>;
}

var countries: nodeIndex<Country>;

@expose
fn getHouseTemperatureRange(houseId: String, from: time, to: time): Array<float> {
  var house = findHouse(houseId) ?? throw "House not found";
  var result: Array<float> = [];
  house.temperatures.iterateFrom(from, to, fn (t: time, v: float) {
    result.add(v);
  });
  return result;
}

@mcp
fn getCountryStats(name: String): Map<String, any> {
  var country = countries.findBy("name", name) ?? throw "Not found";
  return {
    "cities": country.cities.size(),
    "name": country.name
  };
}
```
