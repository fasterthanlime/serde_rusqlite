# serde_rusqlite

## Documentation

See [full documentation](https://docs.rs/serde_rusqlite)

## Usage

Add this to your Cargo.toml:
```
[dependencies]
serde_rusqlite = "0.19"
```

[![Build Status](https://travis-ci.org/twistedfall/serde_rusqlite.svg?branch=master)](https://travis-ci.org/twistedfall/serde_rusqlite)

## Serde Rusqlite

This crate provides convenience functions to bridge serde and rusqlite. With their help
you can "deserialize" rusqlite `Row`'s into serde `Deserialize` types and "serialize" types
implementing `Serialize` into bound query arguments (positional or named) that rusqlite expects.

Serialization of named bound arguments is only supported from `struct`s and `map`s because other
serde types lack column name information. Likewise, serialization of positional bound arguments
is only supported from `tuple`s, `sequence`s and primitive non-iterable types. In the latter case
the result will be single-element vector. Each serialized field or element must implement
`rusqlite::types::ToSql`.

For deserialization you can use two families of functions: `from_*()` and `from_*_with_columns()`.
The most used one is the former. The latter allows you to specify column names for types that need
them, but don't supply them. This includes different `Map` types like `HashMap`. Specifying columns
for deserialization into e.g. `struct` doesn't have any effect as the field list of the struct itself
will be used in any case.

SQLite only supports 5 types: `NULL` (`None`), `INTEGER` (`i64`), `REAL` (`f64`), `TEXT` (`String`)
and `BLOB` (`Vec<u8>`). Corresponding rust types are inside brackets.

Some types employ non-trivial handling, these are described below:

* Serialization of `u64` will fail if it can't be represented by `i64` due to sqlite limitations.
* Simple `enum`s will be serialized as strings so:

  ```
  enum Gender {
     M,
     F,
  }
  ```

  will have two possible `TEXT` options in the database "M" and "F". Deserialization into `enum`
  from `TEXT` is also supported.
* `bool`s are serialized as `INTEGER`s 0 or 1, can be deserialized from `INTEGER` and `REAL` where
  0 and 0.0 are `false`, anything else is `true`.
* `f64` and `f32` values of `NaN` are serialized as `NULL`s. When deserializing such value `Option<f64>`
  will have value of `None` and `f64` will have value of `NaN`. The same applies to `f32`.
* `Bytes`, `ByteBuf` from `serde_bytes` are supported as optimized way of handling `BLOB`s.
* `unit` serializes to `NULL`.
* Only `sequence`s of `u8` are serialized and deserialized, `BLOB` database type is used. It's
  more optimal though to use `Bytes` and `ByteBuf` from `serde_bytes` for such fields.
* `unit_struct` serializes to `struct` name as `TEXT`, when deserializing the check is made to ensure
  that `struct` name coincides with the string in the database.

## Examples
```rust
extern crate rusqlite;
#[macro_use]
extern crate serde_derive;
extern crate serde_rusqlite;

use serde_rusqlite::*;
use rusqlite::NO_PARAMS;

#[derive(Serialize, Deserialize, Debug, PartialEq)]
struct Example {
   id: i64,
   name: String,
}

fn main() {
   let connection = rusqlite::Connection::open_in_memory().unwrap();
   connection.execute("CREATE TABLE example (id INT, name TEXT)", NO_PARAMS).unwrap();

   // using structure to generate named bound query arguments
   let row1 = Example{ id: 1, name: "first name".into() };
   connection.execute_named("INSERT INTO example (id, name) VALUES (:id, :name)", &to_params_named(&row1).unwrap().to_slice()).unwrap();

   // using tuple to generate positional bound query arguments
   let row2 = (2, "second name");
   connection.execute("INSERT INTO example (id, name) VALUES (?, ?)", &to_params(&row2).unwrap().to_slice()).unwrap();

   // deserializing using query() and from_rows()
   let mut statement = connection.prepare("SELECT * FROM example").unwrap();
   let mut res = from_rows::<Example>(statement.query(NO_PARAMS).unwrap());
   assert_eq!(res.next().unwrap(), row1);
   assert_eq!(res.next().unwrap(), Example{ id: 2, name: "second name".into() });

   // deserializing using query_and_then() and from_row()
   let mut statement = connection.prepare("SELECT * FROM example").unwrap();
   let mut rows = statement.query_and_then(NO_PARAMS, from_row::<Example>).unwrap();
   assert_eq!(rows.next().unwrap().unwrap(), row1);
   assert_eq!(rows.next().unwrap().unwrap(), Example{ id: 2, name: "second name".into() });

   // deserializing using query() and from_rows_ref()
   let mut statement = connection.prepare("SELECT * FROM example").unwrap();
   let mut rows = statement.query(NO_PARAMS).unwrap();
   {
      // only first record is deserialized here
      let mut res = from_rows_ref::<Example>(&mut rows);
      assert_eq!(res.next().unwrap(), row1);
   }
   // the second record is deserialized using the original Rows iterator
   assert_eq!(from_row::<Example>(&rows.next().unwrap().unwrap()).unwrap(), Example{ id: 2, name: "second name".into() });

   // deserializing using query() and from_rows_with_columns()
   let mut statement = connection.prepare("SELECT * FROM example").unwrap();
   let columns = columns_from_statement(&statement);
   let mut res = from_rows_with_columns::<Example, _>(statement.query(NO_PARAMS).unwrap(), &columns);
   assert_eq!(res.next().unwrap(), row1);
   assert_eq!(res.next().unwrap(), Example{ id: 2, name: "second name".into() });

   // deserializing using query_and_then() and from_row_with_columns()
   let mut statement = connection.prepare("SELECT * FROM example").unwrap();
   let columns = columns_from_statement(&statement);
   let mut rows = statement.query_and_then(NO_PARAMS, |row| from_row_with_columns::<Example, _>(row, &columns)).unwrap();
   assert_eq!(rows.next().unwrap().unwrap(), row1);
   assert_eq!(rows.next().unwrap().unwrap(), Example{ id: 2, name: "second name".into() });

   // deserializing using query() and from_rows_ref_with_columns()
   let mut statement = connection.prepare("SELECT * FROM example").unwrap();
   let columns = columns_from_statement(&statement);
   let mut rows = statement.query(NO_PARAMS).unwrap();
   {
      // only first record is deserialized here
      let mut res = from_rows_ref_with_columns::<Example, _>(&mut rows, &columns);
      assert_eq!(res.next().unwrap(), row1);
   }
   // the second record is deserialized using the original Rows iterator
   assert_eq!(from_row_with_columns::<Example, _>(&rows.next().unwrap().unwrap(), &columns).unwrap(), Example{ id: 2, name: "second name".into() });

}
```

License: LGPL-3.0
