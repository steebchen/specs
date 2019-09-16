- Start Date: 2019-08-28
- RFC PR: (leave this empty)
- Prisma Issue: (leave this empty)

---

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Motivation](#motivation)
- [Error Causes and Handling Strategies](#error-causes-and-handling-strategies)
  - [Validation Errors](#validation-errors)
  - [Data Error](#data-error)
  - [Runtime Error](#runtime-error)
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
- [Unknown Errors](#unknown-errors)
  - [Unknown Error Template](#unknown-error-template)
  - [Unknown Error Handling](#unknown-error-handling)
    - [Photon.js](#photonjs-1)
    - [Studio](#studio)
    - [CLI](#cli)
- [Error Log Masking](#error-log-masking)
- [Error Character Encoding](#error-character-encoding)
- [Appendix](#appendix)
  - [Common Database Errors](#common-database-errors)
    - [MySQL](#mysql)
    - [PostgreSQL](#postgresql)
    - [SQLite](#sqlite)
- [Open Questions?](#open-questions)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Motivation

![Prisma 2 architecture diagram with errors overlay](./errors-spec.png)

// TODO: Move image to figma

| Component   | Description                                                                                        |
| ----------- | -------------------------------------------------------------------------------------------------- |
| SDK Users   | Photon.js, Studio, Prisma CLI: dev, lift, generate commands, etc.                                  |
| Prisma SDK  | Helps tools interact with binaries. Spec [here](../sdk-js/Readme.md)                               |
| Core        | Binary artifacts of Prisma 2 compilation and a part of the SDK. Spec [here](../binaries/Readme.md) |
| Data source | A database or any other data source supported by Prisma                                            |

Prisma 2 ecosystem provides various layers of tools that communicate with each other to provide desired outcome.

Broadly, the life-cycle of a request (Query request or CLI command) can be seen in this diagram.

| SDK User | →   | Prisma SDK | →   | Binaries | →   | Data source |
| -------- | --- | ---------- | --- | -------- | --- | ----------- |


# Error Causes and Handling Strategies

There can be various reasons for a request/operation to fail. This section broadly classifies potential error causes and handling strategies.

## Validation Errors

These are usually caused by faulty user input. For example,

- Incorrect database credentials
- Invalid data source URL
- Malformed schema syntax

Handling strategy: Any user input must be validated and user should be notified about the validation error details.

## Data Error

These are usually caused when a database constraint fails. For example,

- Record not found
- Unique constraint violation
- Foreign key constraint violation
- Custom constraint violation

Handling strategy: Domain errors would usually originate from the data source and the underlying message should be relayed to the user with some context.

## Runtime Error

// TODO: Seeking feedback on this terminology. Something that encapsulates Database errors, OS errors.

These are caused due to an error in Prisma runtime. For example,

- The available binary is not compiled for this platform
- SDK failed to bind a port for query engine
- Database is not reachable
- Database timeout

Handling strategy: Notify the user by relaying the message from OS/Database and suggesting them to retry. They might need to free up resources or do something at the OS level.

In certain cases, like when a port collision, Prisma SDK can try to retry gracefully as well with a different port.

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

## Prisma SDK

SDK acts as the interface between the binaries and the tools. This section covers errors from SDK, binaries and the network between SDK ⇆ Binary and Binary ⇆ Data source.

// TODO: Update description

### Common

| Title                                | Description                                             | Code  |
| ------------------------------------ | ------------------------------------------------------- | ----- |
| Incorrect database credentials       | User inputted wrong database credentials                | P1000 |
| Database not reachable               | The URL is not reachable (bad URL or network too)       | P1001 |
| Database errors - corruption         | The database is reachable but corrupted                 | P1002 |
| Database timeout                     | Credentials are correct but request timed out           | P1003 |
| Database does not exist              | A database with the provided name does not exist        | P1004 |
| Binary for the wrong platform        | The downloaded binary is not compiled for this platform | P1005 |
| Unable to bind binary to a free port | Port already in use                                     | P1006 |

### Query Engine

| Title                               | Description                                                                       | Code  |
| ----------------------------------- | --------------------------------------------------------------------------------- | ----- |
| Value too long                      | The value is too long for the column                                              | P2000 |
| Record not found error              | The record specified by where condition does not exist                            | P2001 |
| Unique key violation error          | `UniqueConstraintViolation: Unique constraint failed: ${field_name}`              | P2002 |
| Foreign key violation error         | Database level constraint violation                                               | P2003 |
| Custom constraint violation         | Database level constraint violation                                               | P2004 |
| Stored value is invalid             | Example: https://github.com/prisma/photonjs/issues/214                            | P2005 |
| Type mismatch - invalid ID          | Invalid UUID in String (TODO: Open question: shim)                                | P2006 |
| Type mismatch - invalid Date        | https://github.com/prisma/photonjs/issues/212 (TODO: Open question: shim)         | P2007 |
| Type mismatch - invalid JSON        | https://github.com/prisma/photonjs/issues/60 (TODO: Open question: shim/coercion) | P2008 |
| Type mismatch - invalid custom type | https://github.com/prisma/specs/issues/119 (TODO: Open question: shim/coercion)   | P2009 |

### Migration Engine

| Title                                                     | Description                                                                  | Code  |
| --------------------------------------------------------- | ---------------------------------------------------------------------------- | ----- |
| Database creation failed                                  | `DatabaseCreationError: Database creation error: ${error}`                   | P3000 |
| Destructive migration                                     | Migration possible with destructive changes i.e. data loss                   | P3001 |
| Incorrect migration generated (multiple things, rollback) | The generated migration is incorrect due to invalid query or existing tables | P3002 |

### Schema Parser

| Title                                             | Description                                            | Code  |
| ------------------------------------------------- | ------------------------------------------------------ | ----- |
| Schema parsing error (multiple things, syntax)    | User input schema is invalid                           | P4000 |
| Schema semantic error (multiple things, semantic) | Parses but incorrect "meaning" like relation ambiguity | P4001 |
| Schema string input validation                    | Database URL in Prisma schema fails to parse           | P4002 |

### Introspection

| Title                 | Description                                   | Code  |
| --------------------- | --------------------------------------------- | ----- |
| Introspection failure | Introspection failed to produce a schema file | P5000 |

### Unclassified SDK errors

## Photon.js

| Title                         | Description                                           | Code |
| ----------------------------- | ----------------------------------------------------- | ---- |
| Invalid Photon call           | Querying non-existent fields or wrong arguments, etc. | ???  |
| Query engine connection error | Query engine was started but the process died         | ???  |

Photon relays the following errors from the SDK: `P1000`, `P1001` , `P1002`, `P1003`, `P1004`, `P1005`, `P1006`, `P2000`, `P2001` , `P2002`, `P2003`, `P2004`, `P2005`, `P2006`, `P2007`, `P2008`, `P2009`

Since Photon results are consumed in user-land, it should have a parsable error structure:

// TODO: Come back to this section and expand it more
Error object:

```json
{
  code: "<ERROR_CODE>"
  message: "<ERROR_MESSAGE>"
}
```

Serialization of the error message (default `toString`) will have the following template:

```
${ERROR_CODE}: ${ERROR_MESSAGE}
```

## Prisma Studio

Note: Studio has two workflows:

Electron app: Credentials from the UI → Introspection → Prisma schema → Valid Prisma project
Web app: `prisma2 dev` → Provides Prisma schema i.e a Valid Prisma project

Since studio uses Photon for query building. It relays the same error messages as Photon.
git

// TODO: Introspection failure, database creation failure

## Prisma CLI

### Init

| Title                                  | Description                             | Code |
| -------------------------------------- | --------------------------------------- | ---- |
| Directory already contains schema file | Directory is an existing Prisma project | ???  |
| Starter kit                            | Directory is not empty                  | ???  |

Init command relays the following errors from the SDK: `P3000`, `P5000`

More issues for init command failures are covered here: https://prisma-specs.netlify.com/cli/init/errors/

### Generate

Generate command relays the following errors from the SDK: `P4000`, `P4001`, `P4002`

### Dev

Init command relays the following errors from the SDK: `P1000`, `P1001` , `P1002`, `P1003`, `P1004`, `P1005`, `P1006`, `P3000`, `P3001`, `P4000`, `P4001`, `P4002`

### Lift

// TODO: Programmatic lift access

Init command relays the following errors from the SDK: `P1000`, `P1001` , `P1002`, `P1003`, `P1004`, `P1005`, `P1006`, `P3000`, `P3001`, `P4000`, `P4001`, `P4002`

### Introspect

Init command relays the following errors from the SDK: `P1000`, `P1001` , `P1002`, `P1003`, `P1004`, `P1005`, `P1006`, `P5000`

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

File name: `prisma-error-TIMESTAMP.md` where `TIMESTAMP` is a placeholder for the current timestamp.

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

File name: `prisma-error-TIMESTAMP.md` where `TIMESTAMP` is a placeholder for the current timestamp.

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

// TODO: Replace this with actual Studio telemetry spec once that is available.

### CLI

On encountering an unexpected error, CLI should inform the user and prepare an error report with context of the issue and masked sensitive information to be shared manually or via telemetry.

This is covered in the [CLI telemetry spec](https://prisma-specs.netlify.com/cli/telemetry/).

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

- Do we need error codes for anything other than the SDK?
- Batch API and errors?
- Single errors or error arrays? (in the GraphQL layer for example?)
- Error slugs in place of error codes like: https://github.com/typescript-eslint/typescript-eslint/tree/master/packages/eslint-plugin/src/rules
