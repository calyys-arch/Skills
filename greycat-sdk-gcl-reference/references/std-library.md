# GCL Standard Library Reference

Import with `@library("std", "<version>");` in `project.gcl`.

---

## std::core

### Date & Time

```gcl
// Construct
var t: time = time::now();
var t2: time = time::parse("2025-06-01 12:00", "%Y-%m-%d %H:%M");
var t3: time = time::new(1_717_236_000_000_000, DurationUnit::microseconds);

// Convert
var d: Date = t.to_date(TimeZone::"Europe/Luxembourg");
var t4: time = d.to_time(TimeZone::UTC);

// Arithmetic
var later: time = t + 1_hour;
var diff: duration = later - t;   // returns duration

// Formatting
var s: String = t.format("%Y-%m-%d %H:%M:%S", TimeZone::UTC);
```

**TimeZone**: Always use `TimeZone::` enum (e.g. `TimeZone::UTC`, `TimeZone::"America/New_York"`). Never pass raw strings.

```gcl
// Duration literals
1_us       // 1 microsecond
500_ms     // 500 milliseconds
5_s        // 5 seconds
30_min     // 30 minutes
7_hour     // 7 hours
2_day      // 2 days

// DurationUnit enum
DurationUnit::microseconds
DurationUnit::milliseconds
DurationUnit::seconds
DurationUnit::minutes
DurationUnit::hours
DurationUnit::days
```

### Geo

```gcl
var loc: geo = geo{48.8566, 2.3522};   // positional: lat, lng
var lat = loc.lat();
var lng = loc.lng();
var dist = loc.distance(other_geo);    // metres

// Shapes
var box  = GeoBox { ne: geo{1.0, 2.0}, sw: geo{0.0, 1.0} };
var circ = GeoCircle { center: geo{0.0, 0.0}, radius: 1000.0 };
var poly = GeoPoly { points: [geo{0,0}, geo{1,0}, geo{1,1}] };

if (box.contains(loc)) { ... }
```

### Tuple

```gcl
var pair: Tuple<String, int> = Tuple<String, int> { "hello", 42 };
var fst = pair.first();
var snd = pair.second();
```

---

## std::runtime

### Scheduler

```gcl
// Schedule a recurring task (cron expression)
Scheduler::schedule("0 * * * *", fn() {
    processHourly();
});

// Schedule with a name (allows cancellation)
Scheduler::scheduleNamed("my-job", "*/5 * * * *", fn() {
    poll();
});

// Cancel a named job
Scheduler::cancel("my-job");

// One-shot delay
Scheduler::delay(5_min, fn() { cleanup(); });
```

### Task / Parallelisation

```gcl
// Launch parallel jobs
var jobs = Array<Task<int>> {};
for (item in items) {
    jobs.add(Task<int> { function: processItem, arguments: [item] });
}
await(jobs, MergeStrategy::strict);   // blocks until all complete

for (job in jobs) {
    var result: int = job.result();
}
```

**MergeStrategy**:
- `strict` — throw if any task throws
- `lenient` — collect errors, continue

### Logger

```gcl
info("message ${var}");
warn("warning ${val}");
error("error ${err}");

// Structured logging
Logger::log(LogLevel::info, "event", Map<String, any> {
    "key": "value",
    "count": 42
});
```

### User & Security

```gcl
// Get current authenticated user
var u: User? = Security::currentUser();

// Check role
if (Security::hasRole("admin")) { ... }

// Hash a password (bcrypt)
var hash: String = Security::hashPassword("secret");
var ok: bool     = Security::verifyPassword("secret", hash);

// Generate JWT
var token: String = Security::signJWT(claims, expiry_seconds);
var payload: Map<String, any>? = Security::verifyJWT(token);
```

### System

```gcl
// Environment variables
var val: String? = System::env("MY_VAR");

// Execute shell command
var res: ExecResult = System::exec(["ls", "-la"]);
info(res.stdout);

// Process exit
System::exit(0);
```

### OpenAPI / MCP

```gcl
// Functions tagged @expose are automatically included in OpenAPI spec
// served at /openapi.json

// Tag for MCP tool exposure (use sparingly)
@expose
@tag("mcp")
fn searchData(query: String): Array<Result> { ... }
```

---

## std::io

### CSV

```gcl
// Reading
var reader = CsvReader::open("data.csv", CsvFormat {
    separator: ',',       // char, not String
    hasHeader: true
});
for (row in reader) {
    var name: String = row.getString(0);
    var val: float   = row.getFloat(1);
}
reader.close();

// Writing
var writer = CsvWriter::open("out.csv", CsvFormat { separator: ',' });
writer.writeHeader(["name", "value"]);
writer.writeRow(["Alice", "42.0"]);
writer.close();
```

### JSON

```gcl
// Parsing
var json = JsonReader::parse('{"key": 42}');
var val: int = json.getInt("key");

// Serialization
var out = JsonWriter::stringify(myObject);
```

### HTTP Client

```gcl
// GET
var res: HttpResponse = HttpClient::get("https://api.example.com/data", null);
var body: String = res.text();
var status: int  = res.status();

// POST with JSON body
var headers = Map<String, String> { "Content-Type": "application/json" };
var res2 = HttpClient::post("https://api.example.com/create",
                             '{"name":"test"}', headers);

// Timeout (ms)
var opts = HttpClientOptions { timeout: 5000 };
var res3 = HttpClient::getWithOptions("https://...", null, opts);
```

### Email

```gcl
var cfg = SmtpConfig {
    host: "smtp.example.com",
    port: 587,
    user: "sender@example.com",
    password: System::env("SMTP_PASS") ?? ""
};
Email::send(cfg, EmailMessage {
    to: ["recipient@example.com"],
    subject: "Hello",
    body: "World"
});
```

### FileWalker

```gcl
var walker = FileWalker::new("./data");
for (path in walker) {
    if (path.endsWith(".csv")) {
        processFile(path);
    }
}
```

---

## std::util

### Collections

```gcl
var q = Queue<int> {};
q.push(1); q.push(2);
var head: int? = q.pop();   // returns null when empty

var s = Stack<String> {};
s.push("a"); s.push("b");
var top: String? = s.pop();
```

### Statistics

```gcl
// Sliding window (fixed capacity, FIFO eviction)
var win = SlidingWindow<float> { capacity: 100 };
win.push(3.14);
var avg: float = win.mean();
var std: float = win.stddev();

// Histogram
var hist = Histogram { buckets: 10, min: 0.0, max: 100.0 };
hist.add(42.0);
var count: int = hist.bucketCount(42.0);

// Gaussian
var g = Gaussian {};
g.add(1.0); g.add(2.0); g.add(3.0);
var mean: float = g.mean();
var variance: float = g.variance();
```

### Random

```gcl
var rng = Random {};                        // seeded from system entropy
var r: float = rng.uniform(0.0, 1.0);
var i: int   = rng.uniformInt(0, 100);

var seeded = Random { seed: 42 };           // reproducible
```

### Plot

```gcl
// Build chart data for frontend rendering
var chart = Plot::line("My Chart");
chart.addSeries("temperature", times, values);
chart.xLabel("Time");
chart.yLabel("°C");
// Return chart from @expose fn to render in UI
```
