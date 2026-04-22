---
name: greycat-sdk-gcl-reference
description: "GreyCat C API and GCL Standard Library reference. Use for: (1) Native C development with gc_machine_t context, tensors, objects, memory management, crypto, I/O; (2) GCL Standard Library modules - std::core (Date/Time/Tuple/geospatial types), std::runtime (Scheduler/Task/Logger/User/Security/System/OpenAPI/MCP), std::io (CSV/JSON/XML/HTTP/Email/FileWalker), std::util (Queue/Stack/SlidingWindow/Gaussian/Histogram/Quantizers/Random/Plot); (3) Plugin development patterns - lifecycle hooks, type configuration, nativegen, module-level and type-level function linking, global state, thread safety, conditional logging. Keywords: GreyCat, GCL, native functions, tensors, task automation, scheduler, plugin development, gc_machine_t, gc_slot_t."
---

# GreyCat SDK & GCL Reference

Native C extension development and GCL application reference for the GreyCat ecosystem.

## Core C API Primitives

| Type | Purpose |
|------|---------|
| `gc_machine_t*` | Execution context — passed to every native function |
| `gc_slot_t` | Tagged value container (holds any GCL value) |
| `gc_type_t` | Type enum (int, float, bool, string, object, null…) |
| `gc_object_t*` | Heap object handle |

### Native Function Signature

```c
void my_native_fn(gc_machine_t* m) {
    // read params from stack, push result
}
```

### Reading Parameters

```c
gc_slot_t* p0 = gc_machine_param(m, 0);   // first param
gc_slot_t* p1 = gc_machine_param(m, 1);   // second param
int64_t    v  = gc_slot_int(p0);           // extract int64
double     f  = gc_slot_float(p0);         // extract float
bool       b  = gc_slot_bool(p0);          // extract bool
```

### Returning Values

```c
gc_machine_return_int(m, 42);
gc_machine_return_float(m, 3.14);
gc_machine_return_bool(m, true);
gc_machine_return_null(m);
gc_machine_return_object(m, obj);
```

### Type Checking

```c
gc_type_t t = gc_slot_type(slot);
if (t == GC_TYPE_INT)    { ... }
if (t == GC_TYPE_FLOAT)  { ... }
if (t == GC_TYPE_NULL)   { ... }
if (t == GC_TYPE_STRING) { ... }
```

Full C API reference → [references/c-api.md](references/c-api.md)

---

## Tensors

```c
// Create tensor
gc_object_t* tensor = gc_tensor_new(m, GC_TYPE_FLOAT, rank, dims);

// Access element at indices [i][j]
size_t idx[] = { i, j };
gc_slot_t* elem = gc_tensor_at(m, tensor, idx);
gc_slot_set_float(elem, 3.14);

// Get tensor shape
size_t rank  = gc_tensor_rank(tensor);
size_t* dims = gc_tensor_dims(tensor);
```

---

## Object Manipulation

```c
// Create object of registered type
gc_object_t* obj = gc_object_new(m, type_id);

// Get/set fields
gc_slot_t* field = gc_object_field(obj, field_idx);
gc_slot_set_int(field, 100);

// Read string value
const char* str = gc_slot_string(slot);
size_t      len = gc_slot_string_len(slot);
```

---

## Memory & Scratch Buffers

```c
// Scratch buffer (freed after native call returns)
void* buf = gc_machine_scratch(m, byte_size);

// GC-managed allocation (do NOT free manually)
void* mem = gc_machine_alloc(m, byte_size);
```

**CRITICAL**: Never `free()` GC-managed memory. Scratch buffers are invalidated when the native call returns.

---

## GCL Standard Library Quick Reference

### std::core

| Type | Description |
|------|-------------|
| `Date` | Calendar date with timezone support |
| `Time` (alias: `time`) | Microsecond-precision timestamp |
| `Duration` (alias: `duration`) | Time span |
| `Tuple<A,B>` | Immutable pair |
| `geo` | WGS-84 coordinate `geo{lat, lng}` |
| `GeoBox`, `GeoCircle`, `GeoPoly` | Geospatial shapes |

### std::runtime

| Symbol | Purpose |
|--------|---------|
| `Scheduler` | Cron-style task scheduling |
| `Task<T>` | Async job (see parallelisation) |
| `Logger` | Structured log output |
| `User` / `Role` | Authentication & authorisation |
| `Security` | Password hashing, JWT |
| `System` | OS-level ops (env, exec, signal) |
| `OpenAPI` | Expose endpoints as OpenAPI spec |
| `MCP` | Expose endpoints as MCP tools |

**Scheduler example (GCL)**:
```gcl
@expose
fn startScheduler() {
    Scheduler::schedule("0 * * * *", fn() { processHourly(); });
}
```

### std::io

| Symbol | Purpose |
|--------|---------|
| `CsvReader` / `CsvWriter` | Typed CSV parsing/emission |
| `JsonReader` / `JsonWriter` | JSON I/O |
| `XmlReader` | XML parsing |
| `HttpClient` | Outbound HTTP requests |
| `Email` | SMTP send |
| `FileWalker` | Recursive directory traversal |

**HTTP example (GCL)**:
```gcl
var res = HttpClient::get("https://api.example.com/data", null);
var body = res.text();
```

### std::util

| Symbol | Purpose |
|--------|---------|
| `Queue<T>` / `Stack<T>` | In-memory FIFO/LIFO |
| `SlidingWindow<T>` | Rolling aggregation |
| `Gaussian` | Normal distribution stats |
| `Histogram` | Bucketed frequency counts |
| `Quantizer` | Online quantile estimation |
| `Random` | Seeded PRNG — `Random{}.uniform(min,max)` |
| `Plot` | Simple chart data structures |

Full standard library reference → [references/std-library.md](references/std-library.md)

---

## Plugin Development

### Lifecycle Hooks

```c
// Called once at plugin load
void gc_plugin_init(gc_machine_t* m) { }

// Called once at plugin unload
void gc_plugin_destroy(gc_machine_t* m) { }
```

### Registering Native Functions

```c
// In plugin init — link GCL function name to C function
gc_plugin_register_fn(m, "myModule::myFn", my_native_fn, param_count);
```

### Type-Level Functions (methods)

```c
// Register as method on a GCL type
gc_plugin_register_type_fn(m, type_id, "methodName", my_method_fn);
```

### nativegen Pattern

Use `greycat codegen c` to generate C header stubs from `.gcl` type definitions, then implement each stub:

```bash
greycat codegen c --out native/gen/
```

### Global State & Thread Safety

```c
// Store per-plugin state
static MyState* g_state = NULL;

void gc_plugin_init(gc_machine_t* m) {
    g_state = calloc(1, sizeof(MyState));
    gc_plugin_set_userdata(m, g_state);
}
// ALWAYS protect shared state with a mutex — GCL workers run in parallel
```

### Conditional Logging

```c
if (gc_machine_log_enabled(m, GC_LOG_DEBUG)) {
    gc_machine_log(m, GC_LOG_DEBUG, "value=%ld", val);
}
```

Full plugin patterns → [references/plugin-dev.md](references/plugin-dev.md)

---

## Common Pitfalls

| Wrong | Correct |
|-------|---------|
| `free(gc_alloc'd ptr)` | Never free GC memory manually |
| Reading scratch buf after return | Scratch is invalidated on return |
| Missing null check on `gc_slot_string()` | Always check `GC_TYPE_NULL` first |
| Using non-thread-safe globals | Protect with mutex |
| `gc_slot_int` on a float slot | Check type with `gc_slot_type()` first |
| Registering fn with wrong param count | Must match GCL declaration exactly |

---

## Resources

- **Docs**: https://doc.greycat.io/
- **Downloads / versions**: https://get.greycat.io/
- **GCL language syntax**: see the `greycat` skill (`.gcl` files, nodes, types)
