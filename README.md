# About xo

`xo` is a command-line tool to generate [Go](https://golang.org/project/)
code based on a database schema or a custom query.

`xo` works by using database metadata and SQL introspection queries to discover
the types and relationships contained within a schema, and applying a standard
set of base (or customized) Go templates against the discovered relationships.

Currently, `xo` can generate types for tables, enums, stored procedures, and
custom SQL queries for PostgreSQL, MySQL, Oracle, Microsoft SQL Server, and
SQLite3 databases.

**NOTE:** While the code generated by xo is production quality, it is not the
goal, nor the intention for xo to be a "silver bullet," nor to completely
eliminate the manual authoring of SQL / Go code.

## Database Feature Support

The following is a matrix of the feature support for each database:

|              | PostgreSQL       | MySQL            | Oracle           | Microsoft SQL Server| SQLite           |
| ------------ |:----------------:|:----------------:|:----------------:|:-------------------:|:----------------:|
| Models       |:white_check_mark:|:white_check_mark:|:white_check_mark:|:white_check_mark:   |:white_check_mark:|
| Primary Keys |:white_check_mark:|:white_check_mark:|:white_check_mark:|:white_check_mark:   |:white_check_mark:|
| Foreign Keys |:white_check_mark:|:white_check_mark:|:white_check_mark:|:white_check_mark:   |:white_check_mark:|
| Indexes      |:white_check_mark:|:white_check_mark:|:white_check_mark:|:white_check_mark:   |:white_check_mark:|
| Stored Procs |:white_check_mark:|:white_check_mark:|                  |                     |                  |
| ENUM types   |:white_check_mark:|:white_check_mark:|                  |                     |                  |
| Custom types |:white_check_mark:|                  |                  |                     |                  |

## Installation

Install `goimports` dependency (if not already installed):

```sh
$ go get -u golang.org/x/tools/cmd/goimports
```

Then, install in the usual Go way:

```sh
$ go get -u github.com/xo/xo

# install with oracle support (see notes below)
$ go get -tags oracle -u github.com/xo/xo
```

**_NOTE:_** Go 1.6+ is needed for installing `xo` from source, as it makes use
of the trim template syntax in Go templates, which is not compatible with
previous versions of Go. However, code generated by `xo` should compile with Go
1.3+.

## Quickstart

The following is a quick overview of using `xo` on the command-line:

```sh
# change to project directory
$ cd $GOPATH/src/path/to/project

# make an output directory
$ mkdir -p models

# generate code for a postgres schema
$ xo pgsql://user:pass@host/dbname -o models

# generate code for a Microsoft SQL schema using a custom template directory (see notes below)
$ mkdir -p mssqlmodels
$ xo mysql://user:pass@host/dbname -o mssqlmodels --template-path /path/to/custom/templates

# generate code for a custom SQL query for postgres
$ xo pgsql://user:pass@host/dbname -N -M -B -T AuthorResult -o models/ << ENDSQL
SELECT
  a.name::varchar AS name,
  b.type::integer AS my_type
FROM authors a
  INNER JOIN authortypes b ON a.id = b.author_id
WHERE
  a.id = %%authorID int%%
LIMIT %%limit int%%
ENDSQL

# build generated code
$ go build ./models/
$ go build ./mssqlmodels/

# do standard go install
$ go install ./models/
$ go install ./mssqlmodels/
```

## Command Line Options

The following are `xo`'s command-line arguments and options:

```sh
$ xo --help
usage: xo [--verbose] [--schema SCHEMA] [--out OUT] [--append] [--suffix SUFFIX] [--single-file] [--package PACKAGE] [--custom-type-package CUSTOM-TYPE-PACKAGE] [--int32-type INT32-TYPE] [--uint32-type UINT32-TYPE] [--ignore-fields IGNORE-FIELDS] [--fk-mode FK-MODE] [--use-index-names] [--use-reversed-enum-const-names] [--query-mode] [--query QUERY] [--query-type QUERY-TYPE] [--query-func QUERY-FUNC] [--query-only-one] [--query-trim] [--query-strip] [--query-interpolate] [--query-type-comment QUERY-TYPE-COMMENT] [--query-func-comment QUERY-FUNC-COMMENT] [--query-delimiter QUERY-DELIMITER] [--query-fields QUERY-FIELDS] [--escape-all] [--escape-schema] [--escape-table] [--escape-column] [--enable-postgres-oids] [--name-conflict-suffix NAME-CONFLICT-SUFFIX] [--template-path TEMPLATE-PATH] DSN

positional arguments:
  dsn                    data source name

options:
  --verbose, -v          toggle verbose
  --schema SCHEMA, -s SCHEMA
                         schema name to generate Go types for
  --out OUT, -o OUT      output path or file name
  --append, -a           append to existing files
  --suffix SUFFIX, -f SUFFIX
                         output file suffix [default: .xo.go]
  --single-file          toggle single file output
  --package PACKAGE, -p PACKAGE
                         package name used in generated Go code
  --custom-type-package CUSTOM-TYPE-PACKAGE, -C CUSTOM-TYPE-PACKAGE
                         Go package name to use for custom or unknown types
  --int32-type INT32-TYPE, -i INT32-TYPE
                         Go type to assign to integers [default: int]
  --uint32-type UINT32-TYPE, -u UINT32-TYPE
                         Go type to assign to unsigned integers [default: uint]
  --ignore-fields IGNORE-FIELDS
                         fields to exclude from the generated Go code types
  --include-tables INCLUDE-TABLES
                         only generate code for tables matching all of these regular expressions
  --exclude-tables EXCLUDE-TABLES
                         don't generate code for tables matching any of these regular expressions
  --rename-types RENAME-TYPE
                         use specified type names instead of deriving them from the table names. --type-names "table_1=FooType table_2=BarType"
  --extra TYPES
                         generate *Extra fields for all given types
  --fk-mode FK-MODE, -k FK-MODE
                         sets mode for naming foreign key funcs in generated Go code [values: <smart|parent|field|key>] [default: smart]
  --use-index-names, -j
                         use index names as defined in schema for generated Go code
  --use-reversed-enum-const-names, -R
                         use reversed enum names for generated consts in Go code
  --query-mode, -N       enable query mode
  --query QUERY, -Q QUERY
                         query to generate Go type and func from
  --query-type QUERY-TYPE, -T QUERY-TYPE
                         query's generated Go type
  --query-func QUERY-FUNC, -F QUERY-FUNC
                         query's generated Go func name
  --query-only-one, -1   toggle query's generated Go func to return only one result
  --query-trim, -M       toggle trimming of query whitespace in generated Go code
  --query-strip, -B      toggle stripping type casts from query in generated Go code
  --query-interpolate, -I
                         toggle query interpolation in generated Go code
  --query-type-comment QUERY-TYPE-COMMENT
                         comment for query's generated Go type
  --query-func-comment QUERY-FUNC-COMMENT
                         comment for query's generated Go func
  --query-delimiter QUERY-DELIMITER, -D QUERY-DELIMITER
                         delimiter for query's embedded Go parameters [default: %%]
  --query-fields QUERY-FIELDS, -Z QUERY-FIELDS
                         comma separated list of field names to scan query's results to the query's associated Go type
  --escape-all, -X       escape all names in SQL queries
  --escape-schema, -z    escape schema name in SQL queries
  --escape-table, -y     escape table names in SQL queries
  --escape-column, -x    escape column names in SQL queries
  --enable-postgres-oids
                         enable postgres oids
  --name-conflict-suffix NAME-CONFLICT-SUFFIX, -w NAME-CONFLICT-SUFFIX
                         suffix to append when a name conflicts with a Go variable [default: Val]
  --template-path TEMPLATE-PATH
                         user supplied template path
  --help, -h             display this help and exit
```

## About Base Templates

`xo` provides a set of generic "base" [templates](templates/) for each of the
supported databases, but it is understood these templates are not suitable for
every organization or every schema out there. As such, you can author your own
custom templates, or modify the base templates available in the `xo` source
tree, and use those with `xo` by a passing a directory path via the `--template-path`
flag.

For non-trivial schemas, custom templates are the most practical, common, and
best way to use `xo` (see below quickstart and related example).

### Custom Template Quickstart

The following is a quick overview of copying the base templates contained in
the `xo` project's [`templates/`](templates) directory, editing to suit, and
using with `xo`:

```sh
# change to working project directory
$ cd $GOPATH/src/path/to/my/project

# create a template directory
$ mkdir -p templates

# copy xo templates for postgres
$ cp "$GOPATH/src/github.com/xo/xo/templates/*" templates/

# remove xo binary data
$ rm templates/*.go

# edit base postgres templates
$ vi templates/postgres.*.tpl.go

# use with xo
$ xo pgsql://user:pass@host/db -o models --template-path templates
```

See the Custom Template example below for more information on adapting the base
templates in the `xo` source tree for use within your own project.

### Storing Project Templates

Ideally, the custom templates for your project/schema should be stored
within your project, and used in conjunction with a build pipeline such as
`go generate`:

```sh
# add to custom xo command to go generate:
$ tee -a gen.go << ENDGO
package mypackage

//go:generate xo pgsql://user:pass@host/db -o models --template-path templates
ENDGO

# run go generate
$ go generate

# add custom templates and gen.go to project
$ git add templates gen.go && git commit -m 'Adding custom xo templates for models'
```

Note that `xo` only needs the templates for your specific database. You can
safely delete the templates for the other databases -- make sure, however, that
your templates are not symlinks to another database's templates before
deleting.

### Template Language/Syntax

`xo` templates are standard Go text templates. Please see the [documentation
for Go's standard `text/template` package](https://golang.org/pkg/text/template/)
for information concerning the syntax, logic, and variable use within Go
templates.

### Template Context and File Layout

The contexts (ie, the `.` identifier in templates) made available to custom
templates are instances of `xo/internal/$TYPE` (see below table on `$TYPE`
available `$TYPE`s), and are defined in [`internal/types.go`](internal/types.go).

Each database, `$DBNAME`, has its own set of templates for `$TYPE` and are
available in the [templates/](templates/) directory as `templates/$DBNAME.$TYPE.go.tpl`:

| Template File                         | `$TYPE`      | Description                                           |
|---------------------------------------|--------------|-------------------------------------------------------|
| `templates/$DBNAME.type.go.tpl`       | `Type`       | Template for schema tables/views/queries              |
| `templates/$DBNAME.enum.go.tpl`       | `Enum`       | Template for schema enum definitions                  |
| `templates/$DBNAME.proc.go.tpl`       | `Proc`       | Template for stored procedures/functions ("routines") |
| `templates/$DBNAME.foreignkey.go.tpl` | `ForeignKey` | Template for foreign keys relationships               |
| `templates/$DBNAME.index.go.tpl`      | `Index`      | Template for schema indexes                           |
| `templates/$DBNAME.querytype.go.tpl`  | `QueryType`  | Template for a custom query's generated type          |
| `templates/$DBNAME.query.go.tpl`      | `Query`      | Template for custom query execution                   |
| `templates/xo_db.go.tpl`              | `ArgType`    | Package level template generated once per package     |
| `templates/xo_package.go.tpl`         | `ArgType`    | File header template generated once per file          |

For example, PostgreSQL has [`templates/postgres.foreignkey.go.tpl`](templates/postgres.foreignkey.go.tpl)
which defines the template used by `xo` for PostgreSQL's foreign keys. This
template will be called once for every foreign key relationship that `xo` finds
in a PostgreSQL schema, and each time the template will be passed a different
`internal.ForeignKey` instance, populated fields for `Name`, `Schema`, etc.,
which are then available in the `templates/postgres.foreignkey.go.tpl` as
template variables, and used similar to the following: `{{ .Name }}`, `{{ .Schema }}`,
etc.

Since some of the templates are identical for the supported databases, the
templates are not duplicated, but are instead symlinks in the `xo` source tree.
For example, [`templates/oracle.querytype.go.tpl`](templates/oracle.querytype.go.tpl)
is a symlink to [`templates/postgres.querytype.go.tpl`](`templates/postgres.querytype.go.tpl`).

#### Template Helpers

There is a set of well defined template helpers in [`internal/funcs.go`](internal/funcs.go)
that can assist with writing templated Go code / SQL. Please review how the
base [`templates/`](templates/) make use of helpers, and/or see the inline
documentation for the respective helper func definitions.

#### Packing Templates

The base `xo` templates are bin packed so that they are always available to the
built `xo` binary using [`go-bindata`](https://github.com/jteeuwen/go-bindata) (via
the [`tpl.sh`](tpl.sh) script) and need to be regenerated/included in any
changeset when submitting any template changes to the `xo` project.

If you would like to distribute your own binary version of `xo` with the
included templates, simply modify the templates in the `xo` source tree, run
`tpl.sh`, and build as you normally would.

Alternatively, you can simply do the following:

```sh
$ go generate && go build
```

## Examples

### Example: End-to-End

Please see the [booktest example](examples/booktest) for a full end-to-end
example for each supported database, showcasing how to use a database schema
with `xo`, and the resulting code generated by `xo`.

Additionally, please see the [pokedex example](examples/pokedex) for a
demonstration of running `xo` against a large schema. Please note that this
example is a work in progress, and does not yet work properly with Microsoft
SQL Server and Oracle databases, and has no documentation (for now) -- however
it works very similarly to the booktest end-to-end example.

### Example: Ignoring Fields

Sometimes you may wish to have the database manage the values of columns
instead of having them managed by code generated by `xo`. As such, when you
need `xo` to ignore fields for a database schema, you can use the `--ignore-fields`
flag. For example, a common use case is to define a table with `created_at`
and/or `modified_at` timestamps, where the database is responsible for setting
column values on `INSERT` and `UPDATE`, respectively.

Consider the following PostgreSQL schema where a `users` table has a
`created_at` and `modified_at` field, where `created_at` has a default value of
`now()` and where `modified_at` is updated by a trigger on `UPDATE`:

```PLpgSQL
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name text NOT NULL DEFAULT '' UNIQUE,
  created_at timestamptz default now(),
  modified_at timestamptz default now(),
);

CREATE OR REPLACE FUNCTION update_modified_column() RETURNS TRIGGER AS $$
BEGIN
    NEW.modfified_at = now();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_modtime BEFORE UPDATE ON users FROM EACH ROW EXECUTE PROCEDURE update_modified_column();
```

We can ensure that these columns are managed by PostgreSQL and not by the Go
code generated with `xo` by passing the `--ignore-fields` option:

```sh
# ignore special fields
$ xo pgsql://user:pass@host/db -o models --ignore-fields created_at modified_at
```

### Example: Custom Template -- adding a `GetMostRecent` lookup for all tables

Often, a schema has a common layout/pattern, such as every table having a
`created_at` and `modified_at` field (as in the PostgreSQL schema in the
previous example). It is then a common use-case to have a `GetMostRecent`
lookup for each table type, retrieving the most recently modified rows for each
table (up to some limit, N).

To accomplish this with `xo`, we will need to create our own set of custom
templates, and then add a `GetMostRecent` lookup to the `$DBTYPE.type.go.tpl`
template.

First, we create a copy of the base `xo` templates:

```sh
$ cd $GOPATH/src/path/to/project

$ mkdir -p templates

$ cp $GOPATH/src/github.com/xo/xo/templates/* templates/
```

We can now modify the templates to suit our specific schema, adding lookups,
helpers, or anything else necessary for our schema.

To add a `GetMostRecent` lookup, we edit our copy of the `postgres.type.go.tpl`
template:

```sh
$ vi templates/postgres.type.go.tpl
```

And add the following templated `GetMostRecent` func at the end of the file:

```go
// GetMostRecent{{ .Name }} returns n most recent rows from '{{ .Schema }}.{{ .Table.TableName }}',
// ordered by "created_at" in descending order.
func GetMostRecent{{ .Name }}(db XODB, n int) ([]*{{ .Name }}, error) {
    const sqlstr = `SELECT ` +
        `{{ colnames .Fields "created_at" "modified_at" }}` +
        `FROM {{ $table }} ` +
        `ORDER BY created_at DESC LIMIT $1`

    q, err := db.Query(sqlstr, n)
    if err != nil {
        return nil, err
    }
    defer q.Close()

    // load results
    var res []*{{ .Name }}
    for q.Next() {
        {{ $short }} := {{ .Name }}{}

        // scan
        err = q.Scan({{ fieldnames .Fields (print "&" $short) }})
        if err != nil {
            return nil, err
        }

        res = append(res, &{{ $short }})
    }

    return res, nil
}
```

We can then use the templates in conjunction with `xo` to generate our "model"
code:

```sh
$ xo pgsql://user:pass@localhost/dbname -o models --template-path templates/
```

There will now be a `GetMostRecentUsers` func defined in `models/user.xo.go`,
which can be used as follows:

```go
db, err := dburl.Open("pgsql//user:pass@localhost/dbname")
if err != nil { /* ... */ }

// retrieve 15 most recent items
mostRecentUsers, err := models.GetMostRecentUsers(db, 15)
if err != nil { /* ... */ }
for _, user := range users {
    log.Printf("got user: %+v", user)
}
```

## Using SQL Drivers

Please note that the base `xo` templates do not import any SQL drivers. It is
left for the user of `xo`'s generated code to import the actual drivers. For
reference, these are the expected drivers to use with the code generated by
`xo`:

| Database (driver)            | Package                                                                      |
|------------------------------|------------------------------------------------------------------------------|
| Microsoft SQL Server (mssql) | [github.com/denisenkom/go-mssqldb](https://github.com/denisenkom/go-mssqldb) |
| MySQL (mysql)                | [github.com/go-sql-driver/mysql](https://github.com/go-sql-driver/mysql)     |
| Oracle (ora)                 | [gopkg.in/rana/ora.v4](https://gopkg.in/rana/ora.v4)                         |
| PostgreSQL (postgres)        | [github.com/lib/pq](https://github.com/lib/pq)                               |
| SQLite3 (sqlite3)            | [github.com/mattn/go-sqlite3](https://github.com/mattn/go-sqlite3)           |

Additionally, please see below for usage notes on specific SQL database
drivers.

### MySQL (mysql)

If your schema or custom query contains table or column names that need to be
escaped using any of the `--escape-*` options, you must pass the `sql_mode=ansi`
option to the MySQL driver:

```sh
$ xo --escape-all 'mysql://user:pass@host/?parseTime=true&sql_mode=ansi' -o models
```

And when opening a database connection:

```go
db, err := dburl.Open("mysql://user:pass@host/?parseTime=true&sql_mode=ansi")
```

Additionally, when working with date/time column types in MySQL, one should
pass the `parseTime=true` option to the MySQL driver:

```sh
$ xo 'mysql://user:pass@host/dbname?parseTime=true' -o models
```

And when opening a database connection:

```go
db, err := dburl.Open("mysql://user:pass@host/dbname?parseTime=true")
```

### Oracle (ora)

Oracle support is disabled by default as the [Go Oracle driver](https://github.com/rana/ora)
used by `xo` needs the Oracle `instantclient` libs to be installed/known by
`pkg-config`. If you have already [installed rana's Oracle driver](https://github.com/rana/ora#installation)
according to the installation instructions, you can simply pass `-tags oracle`
to `go get`, `go install` or `go build` to enable Oracle support:

```sh
$ go get -tags oracle -u github.com/xo/xo
```

#### Installing Oracle instantclient on Debian/Ubuntu

On Ubuntu/Debian, you may download the instantclient RPMs
[here](http://www.oracle.com/technetwork/topics/linuxx86-64soft-092277.html).

You should then be able to do the following:

```sh
# install alien, if not already installed
$ sudo aptitude install alien

# install the instantclient RPMs
$ sudo alien -i oracle-instantclient-12.1-basic-*.rpm
$ sudo alien -i oracle-instantclient-12.1-devel-*.rpm
$ sudo alien -i oracle-instantclient-12.1-sqlplus-*.rpm

# get xo
$ go get -u github.com/xo/xo

# copy oci8.pc from xo/contrib to system pkg-config directory
$ sudo cp $GOPATH/src/github.com/xo/xo/contrib/oci8.pc /usr/lib/pkgconfig/

# install rana's ora driver
$ go get -u gopkg.in/rana/ora.v4

# assuming the above succeeded, install xo with oracle support enabled
$ go install -tags oracle github.com/xo/xo
```

#### Contrib Scripts and Oracle Docker Image

It's of note that there are additional scripts available in the
[contrib](contrib/) directory that can help when working with Oracle databases
and `xo`.

For reference, the `xo` developers use the [sath89/oracle-12c](https://hub.docker.com/r/sath89/oracle-12c/) Docker image
for testing `xo`'s Oracle database support.

### SQLite3 (sqlite3)

While not required, one should specify the `loc=auto` option when using `xo`
with a SQLite3 database:

```sh
$ xo 'file:mydatabase.sqlite3?loc=auto' -o models
```

And when opening a database connection:

```go
db, err := dburl.Open("file:mydatabase.sqlite3?loc=auto")
```

## About Primary Keys
For row inserts `xo` determines whether the primary key is
automatically generated by the DB or must be provided by the application for the table row being inserted.
For example a table that has a primary key that is also a foreign key to another table, or a
table that has multiple primary keys in a many-to-many link table, it is desired that
the application provide the primary key(s) for the insert rather than the DB.

`xo` will query the schema to determine if the database provides an automatic primary key
and if the table does not provide one then it will require that the application provide the
primary key for the object passed to the the Insert method. Below is information on how
the logic works for each database type to determine if the DB automatically provides the PK.

### MySQL Auto PK Logic
* Checks for an autoincrement row in the information_schema for the table in question.

### PostgreSQL Auto PK Logic
* Checks for a sequence that is owned by the table in question.

### SQLite Auto PK Logic
* Checks the SQL that is used to generate the table contains
the *AUTOINCREMENT* keyword.
* Checks that the table was created with the primary key type of *INTEGER*.

If either of the above conditions are satisfied then the PK is determined to be automatically provided
by the DB. For the case of integer PK's when you want to override that the PK be manually provided
then you can define the key type as *INT* instead of *INTEGER*, for example as in the following many-to-many
link table:

```sql
  CREATE TABLE "SiteContacts" (
  "ContactId"	INT NOT NULL,
  "SiteId"	INT NOT NULL,
  PRIMARY KEY(ContactId,SiteId),
  FOREIGN KEY("ContactId") REFERENCES "Contacts" ( "ContactId" ),
  FOREIGN KEY("SiteId") REFERENCES "Sites" ( "SiteId" )
)
```

### SQL Server Auto PK Logic
* Checks for an identity associated with one of the columns for the table in question.

### Oracle Auto PK Logic
There is currently no method provided for Oracle as there is no programmatic way to query
for which sequences are associated with tables. All PK's will be assumed to be provided
by the database.

## About xo: Design, Origin, Philosophy, and History

`xo` can likely get you 99% "of the way there" on medium or large database
schemas and 100% of the way there for small or trivial database schemas. In
short, xo is a great launching point for developing standardized packages for
standard database abstractions/relationships, and xo's most common use-case is
indeed in a code generation pipeline, ala `stringer`.

### Design

`xo` is **NOT** designed to be an ORM or to generate an ORM. Instead, `xo` is
designed to vastly reduce the overhead/redundancy of (re-)writing types and
funcs for common database queries/relationships in Go -- it is not meant to be
a "silver bullet".

### History

`xo` was originally developed while migrating a large application written in
PHP to Go. The schema in use in the original app, while well designed, had
become inconsistent over multiple iterations/generations, mainly due to
different naming styles adopted by various developers/database admins over the
preceding years. Additionally, some components had been written in different
languages (Ruby, Java) and had also accumulated significant drift from the
original application and accompanying schema. Simultaneously, a large amount of
growth meant that the PHP/Ruby code could no longer efficiently serve the
traffic volumes.

In late 2014/early 2015, a decision was made to unify and strip out certain
backend services and to fully isolate the API from the original application,
allowing the various components to instead speak to a common API layer instead
of directly to the database, and to build that service layer in Go.

However, unraveling the old PHP/Ruby/Java code became a large headache, as the
code, the database, and the API, all had significant drift -- thus, underlying
function names, fields, and API methods no longer coincided with the actual
database schema, and were named differently in each language. As such, after a
round of standardizing names, dropping cruft, and adding a small number of
relationship changes to the schema, the various codebases were fixed to match
the schema changes. After that was determined to be a success, the next target
was to rewrite the backend services in Go.

In order to keep a similar and consistent workflow for the developers, the
previous code generator (written in PHP and Twig templates) was modified to
generate Go code. Additionally, at this time, but tangential to the story, the
API definitions were ported from JSON to Protobuf to make use of its code
generation abilities as well.

`xo` is the open source version of that code generation tool, and is the fruits
of those development efforts. It is hoped that others will be able to use and
expand `xo` to support other databases -- SQL or otherwise -- and that `xo` can
become a common tool in any Go developer's toolbox.

### Goals

Part of `xo`'s goals is to avoid writing an ORM, or an ORM-like in Go, and to
instead generate static, type-safe, fast, and idiomatic Go code across
languages and databases. Additionally, the `xo` developers are of the opinion
that relational databases should have proper, well-designed relationships and
all the related definitions should reside within the database schema itself:
ie, a "self-documenting" schema. `xo` is an end to that pursuit.

## Related Projects

* [dburl](https://github.com/xo/dburl) - a Go package providing a standard, URL style mechanism for parsing and opening database connection URLs
* [usql](https://github.com/xo/usql) - a universal command-line interface for SQL databases

### Other Projects

The following projects work with similar concepts as xo:

#### Go Generators
* [ModelQ](https://github.com/mijia/modelq)
* [sqlgen](https://github.com/drone/sqlgen)
* [squirrel](https://github.com/Masterminds/squirrel)
* [scaneo](https://github.com/variadico/scaneo)
* [acorn](https://github.com/willowtreeapps/acorn) and
  [rootx](https://github.com/willowtreeapps/rootx) \[[read overview
  here](http://willowtreeapps.com/blog/go-generate-your-database-code/)\]

#### Go ORM-likes
* [sqlc](https://github.com/relops/sqlc)

## TODO
* Completely refactor / fix code, templates, and other issues (PRIORITY #1)
* Add (finish) stored proc support for Oracle + Microsoft SQL Server
* Unit tests / code coverage / continuous builds for binary package releases
* Move database introspection to separate package for reuse by other Go packages
* Overhaul/standardize type parsing
* Finish support for --{incl, excl}[ude] types
* Write/publish template set for protobuf
* Add support for generating models for other languages
* Finish many-to-many and link table support
* Finish example and code for generated *Slice types (also, only generate for the databases its needed for)
* Add example for many-to-many relationships and link tables
* Add support for supplying a file (ie, *.sql) for query generation
* Add support for full text types (tsvector, tsquery on PostgreSQL)
* Finish COMMENT support for PostgreSQL/MySQL and update templates accordingly.
* Add support for JSON types (json, jsonb on PostgreSQL, json on MySQL)
* Add support for GIN index queries (PostgreSQL)
