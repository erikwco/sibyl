# Sibyl

Sibyl is an [OCI][1]-based driver for Rust applications to interface with Oracle databases.

## Example

```rust
use sibyl as oracle; // pun intended :)

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let oracle = oracle::env()?;
    let conn = oracle.connect("localhost/xepdb1", "sandbox", "password")?;
    let stmt = conn.prepare("
        SELECT first_name, last_name, hire_date
          FROM (
                SELECT first_name, last_name, hire_date
                     , row_number() OVER (ORDER BY hire_date) ord
                  FROM hr.employees
                 WHERE hire_date >= :hire_date
               )
         WHERE ord = 1
    ")?;
    let date = oracle::Date::from_string("January 1, 2005", "MONTH DD, YYYY", &oracle)?;
    let rows = stmt.query(&[ &date ])?;
    // The query returns at most one row, hence the use of `if let` to retrieve it
    if let Some( row ) = rows.next()? {
        // Note that sibyl uses 0-based column indexes.
        // LAST_NAME is retrieved as a &str. This is fast as it borrows directly from
        // the column buffer. However, it will be accessible only within the current
        // scope, i.e. only during the lifetime of the current row.
        // LAST_NAME is NOT NULL, therefore it is safe to simply unwrap it.
        let last_name = row.get::<&str>(0)?.unwrap();
        let name =
            // FIRST_NAME is NULL-able
            if let Some( first_name ) = row.get::<&str>(1)? {
                format!("{}, {}", last_name, first_name)
            } else {
                last_name.to_string()
            }
        ;
        let hire_date = row.get::<oracle::Date>(2)?.unwrap();
        let hire_date = hire_date.to_string("YYYY-MM-DD")?;

        println!("{} was hired {}", name, hire_date);
    } else {
        println!("No one was hired after {}", date.to_string("YYYY-MM-DD")?);
    }
    Ok(())
}
```

## Notes on Building

Sibyl needs an installed Oracle client in order to link either to `OCI.DLL` on Windows or to `libclntsh.so` on Linux. It is expected that on Windows the directory with `OCI.DLL` is listed in the `PATH`. On Linux, depending on the kind of Oracle client, an additional manual configuration might be needed. Oracle client's `lib` directory needs to be present in `ld.so.conf` and a link to `libclntsh.so` should exist in a directory searched by linkers - `/lib64` for example.

## Usage

### Environment

The OCI environment handle must be created before any other OCI function can be called. While there can be many environments - for example, one can create an environment per connection - usually one is enought. Sibyl initializes it to be the most compatible with Rust requirements - thread-safe using UTF8 character encoding. That single environment handle can be created in `main`:
```rust
fn main() {
    let oracle = sibyl::env().expect("Oracle OCI environment");
    // ...
}
```
Note however that some function will need a direct refrence to this handle, so instead of passing it around some applications might create it statically:
```rust
use sibyl::Environment;
use lazy_static::lazy_static;

lazy_static!{
    pub static ref ORACLE : Environment = sibyl::env().expect("Oracle OCI environment");
}
```
Then later one would be able to create, for example, a current timestamp as:
```rust
use sibyl::TimestampTZ;

let current_timestamp = TimestampTZ::from_systimestamp(&ORACLE)?;
```

### Connections

Use `Environment::connect` method to connect to a database:
```rust
fn main() {
    let oracle = sibyl::env().expect("Oracle OCI environment");
    let conn = oracle.connect("dbname", "username", "password").expect("New database connection");
    // ...
}
```
Where `dbname` can be any name that is acceptable by Oracle clients - from local TNS name to Eazy Connect identifier to a connect descriptor.

### SQL Statement Execution

All SQL or PL/SQL statements must be prepared before they can be executed:
```rust
let stmt = conn.prepare("
    SELECT employee_id, last_name, first_name
      FROM hr.employees
     WHERE manager_id = :id
  ORDER BY employee_id
")?;
```
A prepared statement can be executed either with the `query` or `execute` or `execute_into` methods:
- `query` is used for `SELECT` statements. In fact, it will complain if you try to `query` any other statement.
- `execute` is used for all other, non-SELECT, DML and DDL that do not have OUT parameters.
- `execute_into` is used with DML and DDL that have OUT parameters.

`query` and `execute` take a slice of IN arguments, which can be specified as positional arguments or as name-value tuples. For example, to execute the above SELECT we can call `query` using a positional argument as:
```rust
let rows = stmt.query(&[ &103 ])?;
```
or binding `:id` by name as:
```rust
let rows = stmt.query(&[
    &( ":id", 103 )
])?;
```

In most cases which binding style to use is a matter of convenience and/or personal preferences. However, in some cases named arguments would be preferable and less ambiguous. For example, statement changes during development might force the change in argument positions. Also SQL and PL/SQL statements have different interpretation of a parameter position. SQL statements create positions for every parameter but allow a single argument to be used for the primary parameter and all its duplicares. PL/SQL on the other hand creates positions for unique parameter names and this might make positioning arguments correctly a bit awkward when there is more than one "duplicate" name in a statement.

`execute_into` allows execution of statements with OUT parameters. For example:
```rust
let stmt = conn.prepare("
    INSERT INTO hr.departments
           ( department_id, department_name, manager_id, location_id )
    VALUES ( hr.departments_seq.nextval, :department_name, :manager_id, :location_id )
 RETURNING department_id
      INTO :department_id
")?;
let mut department_id: u32 = 0;
let num_inserted = stmt.execute(&[
    &( ":department_name", "Security" ),
    &( ":manager_id",      ""         ),
    &( ":location_id",     1700       ),
], &mut [
    &mut ( ":department_id", &mut department_id )
])?;
```

`execute` and `execute_into` return the number of rows affected by the statement. `query` returns what is colloquially called a "streaming iterator" which is typically iterated using `while`. For example (continuing the SELECT example from above):
```rust
let employees = HashMap::new();

let rows = stmt.query(&[ &103 ])?;
while let Some( row ) = rows.next()? {
    let employee_id = row.get::<u32>(0)?.unwrap();
    let last_name   = row.get::<&str>(1)?.unwrap();
    let name =
        if let Some( first_name ) = row.get::<&str>(2)? {
            format!("{}, {}", last_name, first_name)
        } else {
            last_name.to_string()
        }
    ;
    employees.insert(employee_id, name);
}
```
There are a few notable elements in the last example:
- Sibyl uses 0-based indexing of columns in a projection.
- Column values are returned as an `Option`. However if a column declared as NOT NULL, like EMPLOYEE_ID and LAST_NAME, the result will always be `Some` and therefore can be safely unwrapped.
- LAST_NAME and FIRST_NAME are retrieved as `&str`. This is fast as they are borrowed directly from the respective column buffers. However those values will only be valid during the lifetime of the row. If the value needs to continue to exist beyond the lifetime of a row, it should be retrieved as a `String`.



## Oracle Data Types

Sibyl provides API to access several Oracle native data types.

### Number
```rust
use sibyl::Number;

let pi = Number::pi(&oracle);
let two = Number::from_int(2, &oracle);
let two_pi = pi.mul(two)?;
let h = Number::from_string("6.62607004E-34", "9D999999999EEEE", &oracle)?;
let hbar = h.div(two_pi)?;
println!("{}", hbar.to_string("TME")?);
```

### Date
```rust
use sibyl::Date;

let apr18_1996 = Date::from_string("18-APR-1996", "DD-MON-YYYY", &oracle)?;
let next_monday = apr18_1996.next_week_day("MONDAY")?;
println!("{}", next_monday.to_string("MM/DD/YYYY")?);
```

### Timestamp

There are 3 types of timestamps:
- `Timestamp` which is equivalent to Oracle TIMESTAMP,
- `TimestampTZ` - TIMESTAMP WITH TIME ZONE,
- `TimestampLTZ` - TIMESTAMP WITH LOCAL TIME ZONE

```rust
use sibyl::TimestampTZ;

let ts = oracle::TimestampTZ::from_string(
    "July 20, 1969 8:18:04.16 pm UTC",
    "MONTH DD, YYYY HH:MI:SS.FF PM TZR",
    &oracle
)?;
let txt = ts.to_string("Dy, Mon DD, YYYY HH:MI:SS.FF PM TZR", 3)?;

```

### Interval

There are 2 types of intervals:
- `IntervalYM` which is eqivalent to Oracle INTERVAL YEAR TO MONTH,
- `IntervalDS` - INTERVAL DAY TO SECOND

```rust
use sibyl::{ TimestampTZ, IntervalDS };

let launch = TimestampTZ::from_datetime(1969,7,16,13,32,0,0,"UTC", &oracle)?;
let landing = TimestampTZ::from_datetime(1969,7,24,16,50,35,0,"UTC", &oracle)?;
let duration : IntervalDS = landing.subtract(&launch)?;

assert_eq!("+8 03:18:35.000", duration.to_string(1,3)?);
```

### RowID

Oracle ROWID can be selected and retrieved explicitly into an instance of the `RowID`. However, one interesting case is SELECT FOR UPDATE queries where Oracle returns ROWIDs implicitly. Those can be retrieved using `Row::get_rowid` method.

```rust
let stmt = conn.prepare("
    SELECT manager_id
      FROM hr.employees
     WHERE employee_id = :id
       FOR UPDATE
")?;
let rows = stmt.query(&[ &107 ])?;
let cur_row = rows.next()?;
assert!(cur_row.is_some());
let row = cur_row.unwrap();

let manager_id: u32 = row.get(0)?.unwrap_or_default();
assert_eq!(103, manager_id);

let rowid = row.get_rowid()?;

let stmt = conn.prepare("
    UPDATE hr.employees
       SET manager_id = :manager_id
     WHERE rowid = :row_id
")?;
let num_updated = stmt.execute(&[
    &( ":manager_id", 102 ),
    &( ":row_id",  &rowid )
])?;
assert_eq!(1, num_updated);
```

### Cursors

Cursors can be returned explicitly:
```rust
let stmt = conn.prepare("
    BEGIN
        OPEN :emp FOR
            SELECT department_name, first_name, last_name, salary
              FROM hr.employees e
              JOIN hr.departments d
                ON d.department_id = e.department_id;
    END;
")?;
let mut cursor = Cursor::new(&stmt)?;
stmt.execute_into(&[], &mut [ &mut cursor ])?;
let rows = cursor.rows()?;
// ...
```

Or, beginning with Oracle 12.1, implicitly:
```rust
let stmt = conn.prepare("
    DECLARE
        emp SYS_REFCURSOR;
    BEGIN
        OPEN emp FOR
            SELECT department_name, first_name, last_name, salary
              FROM hr.employees e
              JOIN hr.departments d
                ON d.department_id = e.department_id;
        ;
        DBMS_SQL.RETURN_RESULT(emp);
    END;
")?;
stmt.execute(&[])?;
if let Some( cursor ) = stmt.next_result()? {
    let rows = cursor.rows()?;
    // ...
}
```

## Testing

Some of sibyl's tests connect to the database and expect certain objects to exist in it and certain privileges granted:
- At least the HR demo schema should be [installed][2]. If you are using Express Edition, it is already pre-installed.
- While there is no need to install other demo schemas at least `MEDIA_DIR` should be created (see `$ORACLE_HOME/demo/schema/mk_dir.sql`) and point to the directory with (a few of) the files that are provided in the `demo/schema/product_media`.
- Some of the LOB tests need text files that have to be created manually. Those should be UTF-16 BE encoded without BOM. The hexadecimal dump of the expected content is provided in the respective doc tests.
- A test user should be created. The user needs acccess to the HR schema and to the `MEDIA_DIR` directory. See `etc/create_sandbox.sql` for an example of how it can be accomplished.

## Limitations

At this time sibyl provides only the most commonly needed means to interface with the Oracle database. Some of the missing features are:
- Non-blocking execution
- Array interface for multi-row operations
- User defined data types
- PL/SQL collections and tables
- Objects
- JSON data
- LDAP and proxy authentications
- Global transactions
- Session and connection pooling
- High Availability
- Continuous query and publish-subscribe notifications
- Advanced queuing
- Shards
- Direct path load

Some of these features will be added in the upcoming releases. Some will be likely kept on a backburner until the need arises or they are explicitly requested. And some might never be implemented.

[1]: https://docs.oracle.com/en/database/oracle/oracle-database/19/lnoci/index.html
[2]: https://docs.oracle.com/en/database/oracle/oracle-database/19/comsc/installing-sample-schemas.html#GUID-1E645D09-F91F-4BA6-A286-57C5EC66321D