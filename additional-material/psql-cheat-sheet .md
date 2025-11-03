# psql cheat sheet

# Switches 

## Most useful

| Switch  | Meaning                | Notes / Examples                  |
| ------- | ---------------------- | --------------------------------- |
| `-d`    | Database / conn string | `-d prod`, or full conninfo       |
| `-h`    | Host                   | `/var/run/postgresql` for sockets |
| `-p`    | Port                   | Default 5432                      |
| `-U`    | User                   | Use `PGUSER` env var too          |
| `-W`    | Force password prompt  | Overrides `.pgpass`               |
| `-c`    | Execute command        | `psql -d db -c "SELECT now();" `  |
| `-f`    | Run file               | Migration scripts                 |
| `-1`    | Single transaction     | Ideal for schema loads            |
| `-v`    | Set psql variable      | `-v ON_ERROR_STOP=1`              |
| `-X`    | Ignore `~/.psqlrc`     | Clean environment                 |
| `-A`    | Unaligned output       | Pair with `-t` for scripts        |
| `-t`    | Tuples-only            | Data only                         |
| `-x`    | Expanded output        | For wide rows                     |
| `--csv` | CSV output             | Same as `\pset format csv`        |
| `-P`    | pset option            | `-P pager=off`                    |

## Others

| Switch    | Meaning                   | Notes                |
| --------- | ------------------------- | -------------------- |
| `-w`      | Never prompt password     | Fails if required    |
| `-e`      | Echo commands             | Debug/logging        |
| `-a`      | Echo all input            | Includes comments    |
| `-E`      | Echo hidden queries       | Shows `\d` internals |
| `-L FILE` | Log psql session          | Audit trail          |
| `-o FILE` | Output to file            | Same as `\o`         |
| `-F STR`  | Field separator           | Unaligned mode       |
| `-R STR`  | Record separator          | Scripting            |
| `-q`      | Quiet                     | Less chatter         |
| `-n`      | No readline               | Minimal shell        |
| `-H`      | HTML output               | Quick table export   |
| `-T`      | HTML `<table>` attributes | Combine with `-H`    |
| `-l`      | List databases            | Equivalent to `\l`   |
| `--help`  | Help                      |                      |
| `-V`      | Version                   |                      |

# Meta-commands

## Most Useful

| Command       | Purpose                  | Notes                  |
| ------------- | ------------------------ | ---------------------- |
| `\c`          | Connect / switch DB      | `\c dbname`            |
| `\l+`         | List DBs                 | Sizes, collations      |
| `\dn+`        | List schemas             | Show sizes             |
| `\dt+`        | List tables              | Include `S` for system |
| `\d+ name`    | Describe object          | Includes storage       |
| `\du+`        | List roles               | With privileges        |
| `\df+`        | List functions           | Full info              |
| `\dx+`        | List extensions          |                        |
| `\copy`       | Client-side copy         | CSV export/import      |
| `\i file.sql` | Run script file          | migrations             |
| `\g`          | Execute query buffer     |                        |
| `\gset`       | Map query result to vars | Awesome for automation |
| `\x`          | Expanded view            | For wide rows          |
| `\timing on`  | Show execution time      | Always for tuning      |
| `\watch n`    | Repeat command           | Monitoring             |

## Others

| Command           | Purpose                 | Notes                       |          |
| ----------------- | ----------------------- | --------------------------- | -------- |
| `\?`              | Help                    | Command list                |          |
| `\h`              | SQL help                | `\h create table`           |          |
| `\p`              | View query buffer       | Before `\g`                 |          |
| `\r`              | Reset buffer            |                             |          |
| `\s [file]`       | Show / save history     |                             |          |
| `\o file`         | Redirect output         | `\o                         | tee log` |
| `\cd`             | Change dir              |                             |          |
| `\set` / `\unset` | Manage psql vars        |                             |          |
| `\echo`           | Print text/vars         | Debug scripts               |          |
| `\password`       | Change password         |                             |          |
| `\encoding`       | Client encoding         |                             |          |
| `\e` / `\ef`      | Edit buffer / function  | Opens editor                |          |
| `\w file`         | Write buffer to file    |                             |          |
| `\H`              | HTML toggle             |                             |          |
| `\f` / `\R`       | Field/record separators | Unaligned only              |          |
| `\pset`           | Formatting options      | `border`, `format`, `pager` |          |
| `\q`              | Quit                    |                             |          |

## Suggested defaults for DBAs


| Setting         | Value                   | Purpose                     |
| --------------- | ----------------------- | --------------------------- |
| `ON_ERROR_STOP` | `\set ON_ERROR_STOP on` | Abort scripts safely        |
| Pager           | `\pset pager off`       | No pager in automation      |
| Format          | `\pset format aligned`  | Readable tables             |
| `\timing on`    | Always                  | See query time              |
| `\x auto`       | Auto expand             | Makes wide results readable |

