# GreyCat C API Reference

## gc_machine_t — Execution Context

The execution context is the main handle passed to every native function. It provides access to the value stack, type registry, allocator, and logger.

### Stack Operations

```c
// Read input parameters (0-indexed from bottom of frame)
gc_slot_t* gc_machine_param(gc_machine_t* m, size_t idx);

// Number of parameters passed to this native function
size_t gc_machine_param_count(gc_machine_t* m);

// Push return value
void gc_machine_return_int(gc_machine_t* m, int64_t val);
void gc_machine_return_float(gc_machine_t* m, double val);
void gc_machine_return_bool(gc_machine_t* m, bool val);
void gc_machine_return_null(gc_machine_t* m);
void gc_machine_return_object(gc_machine_t* m, gc_object_t* obj);
void gc_machine_return_string(gc_machine_t* m, const char* str, size_t len);
void gc_machine_return_time(gc_machine_t* m, int64_t us_epoch);
void gc_machine_return_duration(gc_machine_t* m, int64_t us);
void gc_machine_return_geo(gc_machine_t* m, double lat, double lng);
```

### Memory Allocation

```c
// GC-managed heap alloc — do NOT free manually
void* gc_machine_alloc(gc_machine_t* m, size_t bytes);

// Scratch buffer — valid only within this native call
void* gc_machine_scratch(gc_machine_t* m, size_t bytes);

// Resize a previous gc_machine_alloc block
void* gc_machine_realloc(gc_machine_t* m, void* ptr, size_t new_bytes);
```

### Type Registry

```c
// Look up a type ID by its fully-qualified GCL name
gc_type_id_t gc_machine_type_id(gc_machine_t* m, const char* fqn);

// Get type name from id
const char* gc_machine_type_name(gc_machine_t* m, gc_type_id_t id);
```

### Calling GCL Functions from C

```c
// Invoke a GCL function by name; params on stack before call
void gc_machine_call(gc_machine_t* m, const char* fn_fqn, size_t param_count);
```

---

## gc_slot_t — Tagged Value Container

```c
// Type inspection
gc_type_t gc_slot_type(const gc_slot_t* slot);

// Primitive extractors (check type first!)
int64_t      gc_slot_int(const gc_slot_t* slot);
double       gc_slot_float(const gc_slot_t* slot);
bool         gc_slot_bool(const gc_slot_t* slot);
int64_t      gc_slot_time(const gc_slot_t* slot);       // µs epoch
int64_t      gc_slot_duration(const gc_slot_t* slot);   // µs
double       gc_slot_geo_lat(const gc_slot_t* slot);
double       gc_slot_geo_lng(const gc_slot_t* slot);
const char*  gc_slot_string(const gc_slot_t* slot);
size_t       gc_slot_string_len(const gc_slot_t* slot);
gc_object_t* gc_slot_object(const gc_slot_t* slot);

// Setters (mutate an existing slot)
void gc_slot_set_int(gc_slot_t* slot, int64_t val);
void gc_slot_set_float(gc_slot_t* slot, double val);
void gc_slot_set_bool(gc_slot_t* slot, bool val);
void gc_slot_set_null(gc_slot_t* slot);
void gc_slot_set_string(gc_slot_t* slot, const char* str, size_t len);
void gc_slot_set_object(gc_slot_t* slot, gc_object_t* obj);
void gc_slot_set_time(gc_slot_t* slot, int64_t us_epoch);
void gc_slot_set_geo(gc_slot_t* slot, double lat, double lng);
```

---

## gc_type_t — Type Enum

```c
typedef enum {
    GC_TYPE_NULL     = 0,
    GC_TYPE_INT      = 1,
    GC_TYPE_FLOAT    = 2,
    GC_TYPE_BOOL     = 3,
    GC_TYPE_STRING   = 4,
    GC_TYPE_OBJECT   = 5,
    GC_TYPE_TIME     = 6,
    GC_TYPE_DURATION = 7,
    GC_TYPE_GEO      = 8,
    GC_TYPE_CHAR     = 9,
    GC_TYPE_TENSOR   = 10,
} gc_type_t;
```

---

## gc_object_t — Heap Object

```c
// Allocate a new object of a known type
gc_object_t* gc_object_new(gc_machine_t* m, gc_type_id_t type_id);

// Access a field by index (order matches GCL type declaration)
gc_slot_t* gc_object_field(gc_object_t* obj, size_t field_idx);

// Get the type ID of an object
gc_type_id_t gc_object_type_id(gc_object_t* obj);

// Get field count
size_t gc_object_field_count(gc_object_t* obj);
```

---

## Tensors

```c
// Create an N-dimensional tensor
// dtype: GC_TYPE_INT, GC_TYPE_FLOAT, GC_TYPE_BOOL
// dims[rank] — size in each dimension
gc_object_t* gc_tensor_new(gc_machine_t* m, gc_type_t dtype,
                            size_t rank, const size_t* dims);

// Access element — idx[rank] array of dimension indices
gc_slot_t* gc_tensor_at(gc_machine_t* m, gc_object_t* tensor,
                         const size_t* idx);

// Shape
size_t        gc_tensor_rank(gc_object_t* tensor);
const size_t* gc_tensor_dims(gc_object_t* tensor);
size_t        gc_tensor_size(gc_object_t* tensor);  // total elements

// Fill entire tensor with a scalar
void gc_tensor_fill_float(gc_object_t* tensor, double val);
void gc_tensor_fill_int(gc_object_t* tensor, int64_t val);
```

---

## Cryptography

```c
// SHA-256
void gc_crypto_sha256(const uint8_t* data, size_t len, uint8_t out[32]);

// HMAC-SHA256
void gc_crypto_hmac_sha256(const uint8_t* key, size_t key_len,
                            const uint8_t* data, size_t data_len,
                            uint8_t out[32]);

// Secure random bytes
void gc_crypto_random_bytes(uint8_t* buf, size_t len);

// Base64 encode/decode
size_t gc_base64_encode(const uint8_t* in, size_t in_len,
                         char* out, size_t out_cap);
size_t gc_base64_decode(const char* in, size_t in_len,
                         uint8_t* out, size_t out_cap);
```

---

## Buffers & I/O

```c
// Byte buffer (growable)
gc_buffer_t* gc_buffer_new(gc_machine_t* m);
void         gc_buffer_append(gc_buffer_t* b, const void* data, size_t len);
const void*  gc_buffer_data(gc_buffer_t* b);
size_t       gc_buffer_len(gc_buffer_t* b);
void         gc_buffer_reset(gc_buffer_t* b);

// String builder
gc_strbuf_t* gc_strbuf_new(gc_machine_t* m);
void         gc_strbuf_append(gc_strbuf_t* b, const char* str, size_t len);
void         gc_strbuf_append_int(gc_strbuf_t* b, int64_t val);
const char*  gc_strbuf_str(gc_strbuf_t* b);
size_t       gc_strbuf_len(gc_strbuf_t* b);
```

---

## Logging

```c
typedef enum {
    GC_LOG_DEBUG = 0,
    GC_LOG_INFO  = 1,
    GC_LOG_WARN  = 2,
    GC_LOG_ERROR = 3,
} gc_log_level_t;

bool gc_machine_log_enabled(gc_machine_t* m, gc_log_level_t level);
void gc_machine_log(gc_machine_t* m, gc_log_level_t level,
                    const char* fmt, ...);
```

**Best practice**: Guard with `gc_machine_log_enabled` before building expensive log strings.

---

## Error Handling

```c
// Throw a GCL exception from native code
void gc_machine_throw(gc_machine_t* m, const char* message);
void gc_machine_throw_fmt(gc_machine_t* m, const char* fmt, ...);

// Check if current frame has a pending exception
bool gc_machine_has_exception(gc_machine_t* m);
```
