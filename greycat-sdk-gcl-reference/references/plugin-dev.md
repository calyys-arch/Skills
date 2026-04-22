# GreyCat Plugin Development Patterns

## Plugin Lifecycle

```c
#include <greycat/plugin.h>

// Called once when the plugin shared library is loaded
void gc_plugin_init(gc_machine_t* m) {
    // Register types, functions, allocate global state
}

// Called once when the plugin is unloaded / runtime shuts down
void gc_plugin_destroy(gc_machine_t* m) {
    // Free global state
}
```

Both symbols must be exported from the shared library (no name mangling on C).

---

## Registering Native Functions

### Module-Level Functions

```c
// Signature: void fn_name(gc_machine_t*)
// param_count must match the GCL declaration exactly
gc_plugin_register_fn(m, "myModule::myFn", my_c_fn, /*param_count=*/2);
```

### Type-Level Methods (Instance Methods)

```c
// Registered on a GCL type — receives `self` as param 0
gc_type_id_t tid = gc_machine_type_id(m, "myModule::MyType");
gc_plugin_register_type_fn(m, tid, "compute", my_compute_fn);
```

### Static Type Functions

```c
gc_plugin_register_static_type_fn(m, tid, "create", my_static_fn);
```

---

## nativegen Workflow

`nativegen` generates C stubs from GCL type declarations so the C side matches the GCL side exactly.

```bash
# Step 1 — generate stubs
greycat codegen c --out native/gen/

# Step 2 — implement each stub in native/impl/
# Step 3 — compile shared library
gcc -shared -fPIC -o libmyplugin.so native/impl/*.c -lgreycat
```

**Generated stub example** (`native/gen/my_module.h`):
```c
// Auto-generated — do not edit
void my_module__MyType__compute(gc_machine_t* m);
```

Your implementation:
```c
#include "native/gen/my_module.h"

void my_module__MyType__compute(gc_machine_t* m) {
    gc_slot_t* self = gc_machine_param(m, 0);
    gc_object_t* obj = gc_slot_object(self);
    // ... implement ...
    gc_machine_return_int(m, result);
}
```

---

## Global State

```c
typedef struct {
    pthread_mutex_t lock;
    int64_t         counter;
    void*           cache;
} MyPluginState;

static MyPluginState* g_state = NULL;

void gc_plugin_init(gc_machine_t* m) {
    g_state = calloc(1, sizeof(MyPluginState));
    pthread_mutex_init(&g_state->lock, NULL);
    gc_plugin_set_userdata(m, g_state);  // attach to plugin handle
}

void gc_plugin_destroy(gc_machine_t* m) {
    pthread_mutex_destroy(&g_state->lock);
    free(g_state->cache);
    free(g_state);
    g_state = NULL;
}
```

Retrieve in any native function:
```c
MyPluginState* state = gc_plugin_get_userdata(m);
```

---

## Thread Safety

GreyCat runs multiple worker threads. Any mutable global state **must** be protected.

```c
void my_increment_fn(gc_machine_t* m) {
    MyPluginState* s = gc_plugin_get_userdata(m);
    pthread_mutex_lock(&s->lock);
    s->counter++;
    int64_t val = s->counter;
    pthread_mutex_unlock(&s->lock);
    gc_machine_return_int(m, val);
}
```

**Rules**:
- Use `pthread_mutex_t` for short critical sections.
- Use `pthread_rwlock_t` for read-heavy state (many readers, rare writes).
- Never call GCL APIs (gc_machine_*) while holding a lock — risk of deadlock.

---

## Type Configuration

Register a GCL type's field layout from C so the runtime knows the struct shape:

```c
gc_type_builder_t* tb = gc_type_builder_new(m, "myModule::Config");
gc_type_builder_add_field(tb, "threshold", GC_TYPE_FLOAT);
gc_type_builder_add_field(tb, "enabled",   GC_TYPE_BOOL);
gc_type_id_t tid = gc_type_builder_build(tb);
```

This must be done in `gc_plugin_init` before any GCL code references the type.

---

## Conditional Logging

```c
// Only build the log string if the level is enabled (avoids allocation cost)
if (gc_machine_log_enabled(m, GC_LOG_DEBUG)) {
    gc_machine_log(m, GC_LOG_DEBUG,
                   "processing item id=%ld score=%.4f", id, score);
}

// Always-visible levels (still guard WARN/ERROR to avoid format string cost)
gc_machine_log(m, GC_LOG_INFO, "plugin initialised");
```

---

## Error Handling

```c
// Validate input and throw on bad state
gc_slot_t* p = gc_machine_param(m, 0);
if (gc_slot_type(p) != GC_TYPE_INT) {
    gc_machine_throw_fmt(m, "expected int, got type %d", gc_slot_type(p));
    return;   // always return immediately after throw
}

// Propagate exception from a nested call
gc_machine_call(m, "otherModule::helper", 1);
if (gc_machine_has_exception(m)) {
    return;   // let the exception bubble up
}
```

---

## CMakeLists.txt Template

```cmake
cmake_minimum_required(VERSION 3.16)
project(my_plugin C)

find_package(GreyCat REQUIRED)

add_library(my_plugin SHARED
    native/impl/my_module.c
)

target_include_directories(my_plugin PRIVATE
    ${GREYCAT_INCLUDE_DIRS}
    native/gen/
)

target_link_libraries(my_plugin PRIVATE ${GREYCAT_LIBRARIES})

set_target_properties(my_plugin PROPERTIES
    PREFIX ""           # output: my_plugin.so not libmy_plugin.so
    POSITION_INDEPENDENT_CODE ON
)
```

---

## Checklist Before Shipping a Plugin

- [ ] `gc_plugin_init` registers all types before any functions
- [ ] All native functions return a value (or explicitly call `gc_machine_return_null`)
- [ ] Shared state protected with mutex/rwlock
- [ ] No `free()` on GC-managed memory
- [ ] Scratch buffers not accessed after native function returns
- [ ] `gc_machine_throw` always followed by `return`
- [ ] `gc_plugin_destroy` frees all malloc'd global state
- [ ] Logging guarded with `gc_machine_log_enabled` for DEBUG level
- [ ] param_count in `gc_plugin_register_fn` matches GCL declaration
