# pmx

Minimalist golang data mapping library for postgres and pgx.

## Install

```
go get -u github.com/wcamarao/pmx
```

## Features

- Simple data mapping with struct tags
- Select database records into an annotated struct or slice
- Insert and update database records from an annotated struct
- Compatible with pgx Query interface i.e. `pgxpool.Pool`, `pgx.Conn`, `pgx.Tx`
- Explicit by design, no magic or conventions
- Allow auto generated values

## Data Mapping

Given the following table:

```sql
create table events (
    id uuid primary key,
    recorded_at timestamptz
);
```

Annotate a data model with struct tags:

```go
type Event struct {
    ID         string    `db:"id" table:"events"`
    RecordedAt time.Time `db:"recorded_at"`
    Transient  string    // ignored by pmx
    transient  string    // ignored by pmx
}
```

- The *first* struct field must be annotated with `table`
- Struct fields annotated with `db` must be exported
- Transient fields can be optionally exported

## Inserting

You must always provide a struct pointer.

Auto generated values are populated back into the struct pointer.

```go
type Event struct {
    ID         string    `db:"id" table:"events"`
    RecordedAt time.Time `db:"recorded_at"`
}

func main() {
    ctx := context.Background()

    conn, err := pgx.Connect(ctx, "postgresql://...")
    if err != nil {
        panic(err)
    }
    defer conn.Close(ctx)

    event := Event{
        ID:         "a1eff19b-4624-46c6-9e09-5910e7b2938d",
        RecordedAt: time.Now(),
    }

    // Generated query: insert into events (id, recorded_at) values ($1, $2) returning *
    err = pmx.Insert(ctx, conn, &event)
    if err != nil {
        panic(err)
    }

    fmt.Printf("%+v\n", event)
}
```

## Selecting into a Struct

When selecting records into a struct, you must provide a struct pointer.

You can handle "Event not found" with `pmx.IsZero()`.

```go
type Event struct {
    ID         string    `db:"id" table:"events"`
    RecordedAt time.Time `db:"recorded_at"`
}

func main() {
    ctx := context.Background()

    conn, err := pgx.Connect(ctx, "postgresql://...")
    if err != nil {
        panic(err)
    }
    defer conn.Close(ctx)

    var event Event
    err = pmx.Select(ctx, conn, &event, "select * from events where id = $1", "a1eff19b-4624-46c6-9e09-5910e7b2938d")
    if err != nil {
        panic(err)
    }

    if pmx.IsZero(event) {
        panic("Event not found")
    }

    fmt.Printf("%+v\n", event)
}
```

## Selecting into a Slice

When selecting records into a slice, you must provide a slice pointer.

The underlying slice type must be a struct pointer.

```go
type Event struct {
    ID         string    `db:"id" table:"events"`
    RecordedAt time.Time `db:"recorded_at"`
}

func main() {
    ctx := context.Background()

    conn, err := pgx.Connect(ctx, "postgresql://...")
    if err != nil {
        panic(err)
    }
    defer conn.Close(ctx)

    var events []*Event
    err = pmx.Select(ctx, conn, &events, "select * from events limit 3")
    if err != nil {
        panic(err)
    }

    if len(events) == 0 {
        panic("No events found")
    }

    fmt.Printf("%+v\n", events)
}
```

## Updating

You must always provide a struct pointer.

The last argument explicitly specifies:

- Which struct fields are allowed to be updated
- Which struct fields are matched in the `where` clause

Auto generated values are populated back into the struct pointer.

```go
type Event struct {
    ID         string    `db:"id" table:"events"`
    RecordedAt time.Time `db:"recorded_at"`
}

func main() {
    ctx := context.Background()

    conn, err := pgx.Connect(ctx, "postgresql://...")
    if err != nil {
        panic(err)
    }
    defer conn.Close(ctx)

    event := Event{
        ID:         "a1eff19b-4624-46c6-9e09-5910e7b2938d",
        RecordedAt: time.Now(),
    }

    // Generated query: update events set recorded_at = $1 where id = $2 returning *
    err = pmx.Update(ctx, conn, &event, &pmx.UpdateOptions{
        Allow: []string{"RecordedAt"},
        Match: []string{"ID"},
    })
    if err != nil {
        panic(err)
    }

    fmt.Printf("%+v\n", event)
}
```

## Auto Generated Values

Given the following table with an auto generated id:

```sql
create table events (
    id bigserial primary key,
    recorded_at timestamptz
);
```

Annotate the `ID` field with a `generated:"always"` struct tag:

```go
type Event struct {
    ID         int64     `db:"id" generated:"always" table:"events"`
    RecordedAt time.Time `db:"recorded_at"`
}
```

So, the `ID` will be excluded from `insert` and `update` statements.

Auto generated values are populated back into the struct pointer after `insert` and `update`.

## ErrInvalidRef

The error `pmx.ErrInvalidRef` ("invalid ref") means you provided an invalid pointer or value.

Valid options are:

- When calling `pmx.Insert()` and `pmx.Update()`, you must always provide a struct pointer.
- When calling `pmx.Select()`, you must provide either a struct pointer or slice pointer.
- When calling `pmx.Select()` with a slice pointer, the underlying slice type must be a struct pointer.

## Roadmap

Potential improvements:

- Multirow insert
- On conflict
- Delete
