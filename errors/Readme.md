# Errors

- Owner: @divyenduz
- Stakeholders: @mavilein @timsuchanek @nikolasburk
- State:
  - Spec: In Progress 🚧
  - Implementation: Future 👽

Definition of errors in Prisma Framework. (In this document we make the distinction between [Unknown Errors](https://github.com/prisma/specs/tree/master/errors#unknown-errors) and [Known Errors](https://github.com/prisma/specs/tree/master/errors#known-errors).)

---

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Motivation](#motivation)
- [Error Causes and Handling Strategies](#error-causes-and-handling-strategies)
- [Error Codes](#error-codes)
- [Known Errors](#known-errors)
  - [Prisma SDK](#prisma-sdk)
    - [Common](#common)
    - [Query Engine](#query-engine)
    - [Migration Engine](#migration-engine)
    - [Schema Parser](#schema-parser)
    - [Introspection](#introspection)
    - [Unclassified SDK errors](#unclassified-sdk-errors)
  - [Photon.js](#photonjs)
  - [Prisma Studio](#prisma-studio)
  - [Prisma CLI](#prisma-cli)
    - [Init](#init)
    - [Generate](#generate)
    - [Dev](#dev)
    - [Lift](#lift)
    - [Introspect](#introspect)
  - [Programmatic access](#programmatic-access)
- [Unknown Errors](#unknown-errors)
  - [Unknown Error Template](#unknown-error-template)
  - [Unknown Error Handling](#unknown-error-handling)
    - [Photon.js](#photonjs-1)
    - [Studio](#studio)
    - [CLI](#cli)
- [Error Log Masking](#error-log-masking)
- [Appendix](#appendix)
  - [Common Database Errors](#common-database-errors)
    - [MySQL](#mysql)
    - [PostgreSQL](#postgresql)
    - [SQLite](#sqlite)
- [Open Questions?](#open-questions)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Motivation

![Prisma 2 architecture diagram with errors overlay](./errors-spec.png)

| Component            | Description                                                                                        |
| -------------------- | -------------------------------------------------------------------------------------------------- |
| Systems that use SDK | Photon.js, Studio, Prisma CLI: dev, lift, generate commands, etc.                                  |
| Prisma SDK           | Helps tools interact with binaries. Spec [here](../sdk-js/Readme.md)                               |
| Core                 | Binary artifacts of Prisma 2 compilation and a part of the SDK. Spec [here](../binaries/Readme.md) |
| Data source          | A database or any other data source supported by Prisma                                            |

Prisma 2 ecosystem provides various layers of tools that communicate with each other to provide desired outcome.

Broadly, the life-cycle of a request (Query request or CLI command) can be seen in this diagram.

| System that uses SDK | →   | Prisma SDK | →   | Binaries | →   | Data source |
| -------------------- | --- | ---------- | --- | -------- | --- | ----------- |


# Error Causes and Handling Strategies

There can be various reasons for a request/operation to fail. This section broadly classifies potential error causes and handling strategies.

<details>
<summary>Validation Errors</summary>
<p>
These are usually caused by faulty user input. For example,

- Incorrect database credentials
- Invalid data source URL
- Malformed schema syntax

Handling strategy: Any user input must be validated and user should be notified about the validation error details.

</p>
</details>

<details>
<summary>Data Error</summary>
<p>These are usually caused when a database constraint fails. For example,

- Record not found
- Unique constraint violation
- Foreign key constraint violation
- Custom constraint violation

Handling strategy: Domain errors would usually originate from the data source and the underlying message should be relayed to the user with some context.

</p>
</details>

<details>
<summary>Runtime Error</summary>
<p>
These are caused due to an error in Prisma runtime. For example,

- The available binary is not compiled for this platform
- SDK failed to bind a port for query engine
- Database is not reachable
- Database timeout

Handling strategy: Notify the user by relaying the message from OS/Database and suggesting them to retry. They might need to free up resources or do something at the OS level.

In certain cases, like when a port collision, Prisma SDK can try to retry gracefully as well with a different port.

</p>
</details>

# Error Codes

Error codes make identification/classification of error easier. Moreover, we can have internal range for different system components

| Tool (Binary)    | Range |
| ---------------- | ----- |
| Common           | 1000  |
| Query Engine     | 2000  |
| Migration Engine | 3000  |
| Schema Parser    | 4000  |
| Introspection    | 5000  |
| Prisma Format    | 6000  |

# Known Errors

## Known Errors Template

When we encounter a known error, we should try to convey enough information to the user so they get a good understanding of the error and can possibly unblock themselves.

The format of our error should be the following:

```
{error_code}: {error_message}

{serialized_meta_schema}

Read more: {code_link}
```

| Template Field         | Description                                                                          |
| ---------------------- | ------------------------------------------------------------------------------------ |
| error_code             | Code identifier of the encountered error                                             |
| error_message          | Error message description that also contains a one liner for "how to fix" suggestion |
| serialized_meta_schema | Serialization of the meta schema that might different for each error                 |
| code_link              | Each error code has a permalink with detailed information                            |

If we encounter a Rust panic, that is covered in the [Unknown Errors](#unknown-errors) section.

## Prisma SDK

SDK acts as the interface between the binaries and the tools. This section covers errors from SDK, binaries and the network between SDK ⇆ Binary and Binary ⇆ Data source.

### Common

| Title                            | Message                                                                                                                                                                                                                              | Code  |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----- |
| Incorrect database credentials   | Authentication failed against database server at `${host}`, the provided database credentials for `${user}` are not valid. <br /> <br /> Please make sure to provide valid database credentials for the database server at \${host}. | P1000 |
| Database not reachable           | Can't reach database server at `${host}`:`${port}` <br /> <br /> Please make sure your database server is running at \${host:port}.                                                                                                  | P1001 |
| Database timeout                 | The database server at `${host}`:`${port}` was reached but timed out. <br /> <br /> Please try again.                                                                                                                                | P1002 |
| Database does not exist          | Database `${databaseName}` does not exist on the database server at `${host}`. <br /> <br />                                                                                                                                         | P1003 |
| Incompatible binary              | The downloaded/provided binary is not compiled for this platform                                                                                                                                                                     | P1004 |
| Unable to start the query engine | The binary process crashed or failed to spawn                                                                                                                                                                                        | P1005 |

Note on P1003: Different consumers of the SDK might handle that differently. Lift save, for example, shows a interactive dialog for user to create it.

### Query Engine

| Title                   | Message                                                                                        | Code  |
| ----------------------- | ---------------------------------------------------------------------------------------------- | ----- |
| Input value too long    | The value for the field `${field_name}` on the is too long for the field's type                | P2000 |
| Record not found        | The record searched for in the where condition does not exist                                  | P2001 |
| Unique key violation    | Unique constraint failed on the field: `${field_name}`                                         | P2002 |
| Foreign key violation   | Foreign key constraint failed on the field: `${field_name}`                                    | P2003 |
| Constraint violation    | A constraint failed on the database: `${error}`                                                | P2004 |
| Stored value is invalid | The value stored in the database for the field `${field_name}` is invalid for the field's type | P2005 |

TODO: Open questions, classify these issues:

| Title                                 | Issue                                              | Open Questions                                              |
| ------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------- |
| Type mismatch: invalid (ID/Date/Json) | The provided value for (ID/Date/Json) is not valid | What known types do we support besides Prisma scalar types? |
| Type mismatch: invalid custom type    | https://github.com/prisma/specs/issues/119         | Do we validate custom types or just pass them through?      |

### Migration Engine

| Title                          | Message                                                            | Code  |
| ------------------------------ | ------------------------------------------------------------------ | ----- |
| Database creation failed       | Failed to create database: `${error}`                              | P3000 |
| Destructive migration detected | Migration possible with destructive changes and possible data loss | P3001 |
| Migration rollback             | The attempted migration was rolled back: `${error}`                | P3002 |

### Schema Parser

| Title                                 | Message                                                                                  | Code  |
| ------------------------------------- | ---------------------------------------------------------------------------------------- | ----- |
| Schema parsing failed                 | Failed to parse schema file: `${error}`                                                  | P4000 |
| Schema relational ambiguity           | There is a relational ambiguity in the schema file between the models `${A}` and `${B}`. | P4001 |
| Schema string input validation errors | Database URL provided in the schema failed to parse                                      | P4002 |

TODO: Are there more known semantic errors like Relation ambiguity

### Introspection

| Title                | Message                                                             | Code  |
| -------------------- | ------------------------------------------------------------------- | ----- |
| Introspection failed | Introspection operation failed to produce a schema file: `${error}` | P5000 |

### Unclassified SDK errors

## Photon.js

| Title                         | Message                                               | Code |
| ----------------------------- | ----------------------------------------------------- | ---- |
| Invalid Photon call           | Querying non-existent fields or wrong arguments, etc. | ???  |
| Query engine connection error | Query engine was started but the process died         | ???  |

Photon relays the following errors from the SDK: `P1000`, `P1001` , `P1002`, `P1003`, `P1004`, `P1005`, `P2000`, `P2001` , `P2002`, `P2003`, `P2004`, `P2005`.

TODO: See open items from Prisma SDK → Query Engine section.

## Prisma Studio

Note: Studio has two workflows:

Electron app: Credentials from the UI → Introspection → Prisma schema → Valid Prisma project
Web app: `prisma2 dev` → Provides Prisma schema i.e a Valid Prisma project

Since studio uses Photon for query building. It relays the same error messages as Photon. Additionally, it relays the following errors from the SDK: `P3000`, `P5000`

## Prisma CLI

### Init

| Title                                  | Message                                 | Code |
| -------------------------------------- | --------------------------------------- | ---- |
| Directory already contains schema file | Directory is an existing Prisma project | ???  |
| Starter kit                            | Directory is not empty                  | ???  |

Init command relays the following errors from the SDK: `P3000`, `P5000`

More issues for init command failures are covered here: https://prisma-specs.netlify.com/cli/init/errors/

### Generate

Generate command relays the following errors from the SDK: `P4000`, `P4001`, `P4002`

### Dev

Init command relays the following errors from the SDK: `P1000`, `P1001` , `P1002`, `P1003`, `P1004`, `P1005`, `P3000`, `P3001`, `P4000`, `P4001`, `P4002`

### Lift

Init command relays the following errors from the SDK: `P1000`, `P1001` , `P1002`, `P1003`, `P1004`, `P1005`, `P3000`, `P3001`, `P4000`, `P4001`, `P4002`

### Introspect

Init command relays the following errors from the SDK: `P1000`, `P1001` , `P1002`, `P1003`, `P1004`, `P1005`, `P5000`

## Programmatic access

Many of these errors from the previous section are expected to be consumed programmatically.

`Photon.js`: In user's code base
`Prisma SDK`: Lift etc, in the tools that use Prisma SDK

Therefore, they should be consumable programmatically and have an error structure:

TODO: Come back to this section and expand it more
Error object:

```json
{
  code: "<ERROR_CODE>"
  message: "<ERROR_MESSAGE>",
  meta: <meta-schema-object>
}
```

Serialization of the error message (default `toString`) will have the following template:

```
${ERROR_CODE}: ${ERROR_MESSAGE}
```

# Unknown Errors

As Prisma 2 is still early, we're not yet aware of all error cases that can occur. This section explains what should happen when Prisma encounters an _unknown_ error.

An error can occur in any of the following tools that currently make up Prisma 2's developer surface area:

- Photon.js
- Studio
- CLI

When an unknown error occurs, **our primary goal** is to get the user to report it by creating a new GitHub issue or send the error report to us via telemetry.

Error messages should include clear guidelines of where to report the issue and what information to include. The following sections provide the templates for these error message per tool.

## Unknown Error Template

Additionally to showing the the error message directly to the user by printing it to the console, we also want to provide rich error reports that users can use to report the issue manually via Github issue or automatically via telemetry. These error reports are stored as markdown files or zip files on the file system. Therefore, each tool has two templates:

- **Logging output** directly shown to the user
- **Error report** (markdown or a zip file) stored on the file system

The error report generally is more exhaustive than the logging output (e.g. it also contains the Prisma schema which would be overkill if printed to the terminal as well). It is also written in Markdown enabling the user to copy and paste the report as a GitHub issue directly.

## Unknown Error Handling

### Photon.js

On encountering an unexpected error, Photon should inform the user and prepare an error report with context of the issue and masked sensitive information to be shared manually or via telemetry.

<details><Summary>Logging output</Summary>

```
Oops, an unexpected error occurred.

Find more info in the error report:
**/path/to/dir/prisma-error-TIMESTAMP.md**

Please help us fix the problem!

Copy the error report and paste it as a GitHub issue here:
**https://www.github.com/prisma/photonjs/issues**

Thanks for helping us making Photon.js more stable! 🙏

An internal error occurred during invocation of **photon.users.create()** in **/path/to/dir/src/.../file.ts**

  ${userStackTrace}
```

> Note: Text enclosed by the double-asterisk `**` means the text should be printed in **bold**.

</details>

<details><Summary>Error report</Summary>

File name: `prisma-error.md` is created inside the project directory on first error and is appended to on subsequent errors.

```
# Error report (Photon JS | July 23, 2019 | 14:42:23 h)

This is an exhaustive report containing all relevant information about the error.

**Please post this report as a GitHub issue so we can fix the problem: https://github.com/prisma/prisma2/issues** 🙏

## Stack trace

${internalStackTrace}

## System info

${uname -a}

## Prisma 2 CLI version

${prisma2 -v}

## Prisma schema file

${schema.prisma}

## Generated Photon JS code

${index.d.ts}
```

> **Note**: The connection strings for the data sources in the Prisma schema file must be obscured!

</details>

### Studio

On encountering an unexpected error, Studio should inform the user and prepare an error report with context of the issue and masked sensitive information to be shared manually or via telemetry.

<details><Summary>Logging output</Summary>

```
Oops, an unexpected error occurred! Find more info in the error report:
**/path/to/dir/prisma-error-TIMESTAMP.md**

Please help us fix the problem!

Copy the error report and paste it as a GitHub issue here:
**https://www.github.com/prisma/prisma2/issues**

Thanks for helping us making Prisma 2 more stable! 🙏
```

> Note: Text enclosed by the double-asterisk `**` means the text should be printed in **bold**.

</details>

<details><Summary>Error report</Summary>

File name: `studio-error-TIMESTAMP.zip` where `TIMESTAMP` is a placeholder for the current timestamp. It would contain the migrations and schema files with sensitive information redacted (see [Error Log Masking](#error-log-masking)) and an information file containing the error report:

```
# Error report (Prisma Studio | July 23, 2019 | 14:42:23 h)

This is an exhaustive report containing all relevant information about the error.

**Please post this report as a GitHub issue so we can fix the problem: https://github.com/prisma/prisma2/issues** 🙏

## Stack trace

${stacktrace}

## System info

${uname -a}

## Browser info

${browserInfo}

## Prisma 2 CLI version

${prisma2 -v}

## Prisma schema file

${schema.prisma}
```

> **Note**: The connection strings for the data sources in the Prisma schema file must be obscured!

</details>

Note that studio can also yield Photon errors as it uses Photon internally. The error log generation in that case would be done by Photon but the UI to prompt user to create a Github issue or send it to us would be handled by Studio.

### CLI

On encountering an unexpected error, CLI should inform the user and prepare an error report with context of the issue.

File name: `prisma-error-TIMESTAMP.zip` where `TIMESTAMP` is a placeholder for the current timestamp. It would contain the migrations and schema files with sensitive information redacted (see [Error Log Masking](#error-log-masking)).

This is covered in the [CLI error handling spec](https://prisma-specs.netlify.com/cli/error-handling/).

# Error Log Masking

Both logging output, error report might contain logs with sensitive information like database URL. Prisma 2 should mask the sensitive information (with asterisks `********`) before dumping the data on the file system.

The error report to be sent back automatically might also contain some proprietary information like the database schema via Prisma schema file.

We must ask the user before collecting such information. This is covered in the [telemetry spec](https://prisma-specs.netlify.com/cli/telemetry/).

# Appendix

## Common Database Errors

### MySQL

- https://www.fromdual.com/mysql-error-codes-and-messages
- https://docs.oracle.com/cd/E19078-01/mysql/mysql-refman-5.0/error-handling.html
- https://www.oreilly.com/library/view/mysql-reference-manual/0596002653/apas02.html
- https://dev.mysql.com/doc/refman/5.5/en/server-error-reference.html

### PostgreSQL

- https://www.postgresql.org/docs/10/errcodes-appendix.html
- https://www.postgresql.org/docs/8.1/errcodes-appendix.html
- https://www.enterprisedb.com/docs/en/9.2/pg/errcodes-appendix.html

### SQLite

- https://www.sqlite.org/rescode.html
- https://www.sqlite.org/c3ref/intro.html
- https://www.sqlite.org/c3ref/c_abort.html
- https://www.sqlite.org/c3ref/errcode.html

# Open Questions?

- Batch API and errors? Discussion https://www.notion.so/prismaio/Errors-Spec-Error-Arrays-4160085305444374a74f6a81b785e57a

- Single errors or error arrays? (in the GraphQL layer for example?) Discussion https://www.notion.so/prismaio/Errors-Spec-Error-Arrays-4160085305444374a74f6a81b785e57a

- Error slugs in place of error codes like: https://github.com/typescript-eslint/typescript-eslint/tree/master/packages/eslint-plugin/src/rules

- Should known errors have a CTA? To create a GH issue? That might help funnel user input for better developer experience. This also teaches users about multiple repositories.
