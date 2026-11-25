# ilogical::schema — Schema Definition & Validation

Define structural schemas for Values and validate data against them.

## Schema Types

| Function | Description |
|----------|-------------|
| `any()` | Accepts any value |
| `type_schema(name)` | Exact type match (null, bool, int, float, string, array, object, bytes, error) |
| `object_schema()` | Object with defined fields |
| `array_schema(item)` | Array with uniform item schema |
| `union_schema(schemas)` | One of multiple schemas |
| `custom_schema(fn)` | Custom validator function |

## Object Fields

```mire
pub fn object_field: (schema :i64, name :str, field_schema :i64, required :bool) :i64
```

Fields store child schema + required flag.

## Validation

```mire
pub fn validate: (schema :i64, value :i64) :i64
```

Returns `0` for valid, error handle for invalid.

Validation rules:
- `any`: Always passes
- `type`: Exact type match
- `object`: All required fields present, all fields match their schemas
- `array`: All items match item schema
- `union`: At least one variant matches
- `custom`: Calls validator function (stub)

## Example

```mire
set user_schema = schema::object_schema()
set user_schema = schema::object_field(user_schema, "name", schema::type_schema("string"), true)
set user_schema = schema::object_field(user_schema, "age", schema::type_schema("int"), false)
set user_schema = schema::object_field(user_schema, "tags", schema::array_schema(schema::type_schema("string")), false)

set user = base::object() :i64 mut
set user = base::object_set(user, "name", base::string("Alice"))
set user = base::object_set(user, "age", base::int(30))
set user = base::object_set(user, "tags", base::array() :i64 mut)

set err = schema::validate(user_schema, user)
if err != 0 { use dasu(base::as_string(err)) }
```
