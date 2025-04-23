# ORM-like DSL in Pawn: How I built a typed data system with macros and enums
📅 Date of writeup: April 23, 2025

## 📌 Context
In a SAMP game server with no OOP, no suitable libraries, no real serialization, I needed to store and load complex game entities in MySQL.

So I created a macro-based ORM system using only enums and preprocessor macros.

## ⚠️ Constraints
- No classes
- No structs with metadata
- No libraries or reflection
- Only enums, defines, and SQL access

## 💡 The Solution

The core idea is simple:
> Use `enum` as a logical schema descriptor,
> and macros to generate everything else: serialization, deserialization, table creation, SQL migration.

I designed an enum-based DSL for database layout:

```c
enum e_PlayerInfo {
    piID,
    piName[24],
    piScore,
    bool: piBanned,
    // ...
}
SQL_ENUM_DEFINE(PlayerInfo) {
    {"id", FIELD_TYPE_INT, 0, FIELD_COLUMN_PRIMARY_KEY | FIELD_COLUMN_AUTO_INCREMENT},
    {"name", FIELD_TYPE_TEXT, 24},
    {"score", FIELD_TYPE_INT},
    {"banned", FIELD_TYPE_BOOL}
}
```

This definition drives **everything**:
- SQL table generation (`CREATE TABLE IF NOT EXISTS`)
- Auto-migration (`ALTER TABLE ADD COLUMN` if needed)
- `UPDATE`, `REPLACE INTO`, `SELECT` queries
- Type-aware serialization/deserialization

---

## 🧬 Interacting with enum as linear memory
Normally, enums in Pawn are used symbolically (`PlayerInfo[playerid][piID]`).
But I treat as linear memory and iterate by offsets, not names.

Two pointers are used:
- `fieldIndex` → iterates over metadata (`FIELD_TYPE`, `FIELD_NAME`, etc)
- `fieldRead` → iterates over data values in `_enum[]`, with dynamic stepping

```c
switch (db[fieldIndex][FIELD_TYPE])
{
    case FIELD_TYPE_BOOL, FIELD_TYPE_INT: {
        mysql_format(conn, buf, sizeof(buf), "%d", _enum[fieldRead]);
        fieldRead++;
    }
    case FIELD_TYPE_TEXT: {
        mysql_format(conn, buf, sizeof(buf), "'%e'", _enum[fieldRead]);
        fieldRead += db[fieldIndex][FIELD_SIZE];
    }
    // ...
}
```
This allows to:
- **skip fields** (`FIELD_TYPE_SKIP`)
- **support text fields** with variable cell sizes
- **build INSERT/UPDATE dynamically** without knowing field names at runtime

---

## 🔁 Passing enum by reference with macros
To pass complex `enum`-arrays by reference (by name), I used macros like this:
```c
#define SQL_LOAD_ENUM(%0,%1,%2) SQL_LoadEnum(%0, %2, %1, #%0)
#define SQL_GET_ENUM_DEFINE(%0) %0DB
```
This converts:
```c
SQL_LOAD_ENUM(PlayerInfo[playerid], SQL_GET_ENUM_DEFINE(PlayerInfo), i);
```
Into:
```c
SQL_LoadEnum(PlayerInfo[playerid], i, PlayerInfoDB, "PlayerInfo[playerid]");
```

So the function receives:
- The actual enum data (`_enum[]`)
- Its metadata (`db[][]`)
- Its name as string (for logging/debug)

In effect, this is a **primitive reflection system**, implemented manually through macros.

---
## 🧠 Bitmask field flags
Each column supports flags:
- `FIELD_COLUMN_PRIMARY_KEY`
- `FIELD_COLUMN_UNIQUE_PAIR`
- `FIELD_COLUMN_AUTO_INCREMENT`
- `FIELD_COLUMN_NOT_NULL`

They are interpreted at runtime to generate SQL statements:
```c
if (flags & FIELD_COLUMN_PRIMARY_KEY)
    strcat(buf, " PRIMARY KEY");
if (flags & FIELD_COLUMN_UNIQUE_PAIR)
    ... // handled separately for composite keys

```
This approarch **decouples data structure from SQL layout logic**, allowing one enum to describe both behavior and storage.

---

## 🧠 Why it matters
This is not about Pawn.
This is about:
- Writing systems where the language gives you nothing
- Manually describing memory layout and type behavior
- Creating a schema-aware data layer from scratch
- Thinking like a compiler and serializer at the same time

Even with no cases, no libraries, no types — I built a working, extensible, low-level serialization system.
And it ran in a live production SAMP server.

---

## 🧾 Source
📁 Full code: [mysql_tricks.pwn](./files/mysql_tricks.pwn)

---

## 🧱 Final thoughts

I didn't know what ORM meant when I wrote this.
I just didn't want to copy-paste SQL queries anymore.

This might not be a framework. But it works.
And it proves that **thinking clearly beats having the right tools**.
