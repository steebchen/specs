- Start Date: 2019-08-28
- RFC PR: (leave this empty)
- Prisma Issue: (leave this empty)

---

# Errors

In this document we make the distinction between [Unknown Errors](#unknown-errors) and [Known Errors](#known-errors).

<!-- toc -->

- [Motivation](#motivation)
- [Error Causes, Codes, and Handling Strategies](#error-causes-codes-and-handling-strategies)
  * [Validation Errors](#validation-errors)
  * [Runtime Error](#runtime-error)
  * [Data Error](#data-error)
- [Error Codes](#error-codes)
  * [Prefix](#prefix)
- [Known Errors](#known-errors)
  * [Prisma SDK](#prisma-sdk)
    + [Unclassified SDK errors](#unclassified-sdk-errors)
  * [Photon.js](#photonjs)
  * [Prisma Studio](#prisma-studio)
  * [Prisma CLI](#prisma-cli)
    + [Init](#init)
    + [Generate](#generate)
    + [Dev](#dev)
    + [Lift](#lift)
    + [Introspect](#introspect)
- [Unknown Errors](#unknown-errors)
  * [Unknown Error Template](#unknown-error-template)
  * [Unknown Error Handling](#unknown-error-handling)
    + [Photon.js](#photonjs-1)
    + [Studio](#studio)
    + [CLI](#cli)
- [Error Log Masking](#error-log-masking)
- [Error Character Encoding](#error-character-encoding)
- [Appendix](#appendix)
  * [Common Database Errors](#common-database-errors)
    + [MySQL](#mysql)
    + [PostgreSQL](#postgresql)
    + [SQLite](#sqlite)
- [Open Questions?](#open-questions)

<!-- tocstop -->

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

Broadly, the life-cycle of a request (Query request or CLI command) can be seen in this architecture diagram.

| SDK User | →   | Prisma SDK | →   | Binaries | →   | Data source |
| -------- | --- | ---------- | --- | -------- | --- | ----------- |


**Note**: An error can happen in any parts of this stack and it bubbles up from that part to the surface area. For example, a unique constraint violation happens in the data source and the first programmatically parsable occurrence happens in the binary. However the same error is handled by Prisma SDK and eventually by the consumer of the SDK (say Photon). We have to document the same error in all these different specs as it bubbles up, thus defining a clear errors interface at each layer. This spec will eventually contain links to this distributed spec work.

This document covers our error handling strategy and covers the following:

1. Known/unknown errors
2. Error log masking
3. Error character encoding

# Error Causes, Codes, and Handling Strategies

There can be various reasons for a request/operation to fail. This section broadly classifies potential error causes and handling strategies.

## Validation Errors

These are usually caused by faulty user input. For example,

- Incorrect database credentials
- Invalid data source URL
- Malformed schema syntax

Handling strategy: Any user input must be validated and user should be notified about the validation error details.

## Runtime Error

// TODO: Seeking feedback on this terminology. Something that encapsulates Database errors, OS errors.

These are caused due to an error in Prisma runtime. For example,

- The available binary is not compiled for this platform
- SDK failed to bind a port for query engine
- Database is not reachable
- Database timeout

Handling strategy: Notify the user by relaying the message from OS/Database and suggesting them to retry. They might need to free up resources or do something at the OS level.

In certain cases, like when a port collision, Prisma SDK can try to retry gracefully as well with a different port.

## Data Error

These are usually caused when a database constraint fails. For example,

- Record not found
- Unique constraint violation
- Foreign key constraint violation
- Custom constraint violation

Handling strategy: Domain errors would usually originate from the data source and the underlying message should be relayed to the user with some context.

# Error Codes

Error codes made identification/classification of error easier. Moreover, for tools like Photon, they also make programmatically handling error conditions easier.

This section describes our error prefix and error numbering strategy.

## Prefix

Marking every known error with a structured error code prefix and a number would make error identification and parsing easier for the users. We can also recommend third party generators/tools to follow the same error code convention.

| Error Cause      | Prefix |
| ---------------- | ------ |
| Validation Error | VE     |
| Data Error       | DE     |
| Runtime Error    | RE     |

# Known Errors

## Prisma SDK

SDK acts as the interface between the binaries and the tools. This section covers errors from SDK, binaries and the network between SDK ⇆ Binary and Binary ⇆ Data source.

| Title                                             | Description                                                          | Prefix | Code  |
| ------------------------------------------------- | -------------------------------------------------------------------- | ------ | ----- |
| Schema parsing error (multiple things, syntax)    | User input schema is invalid                                         | VE     | P1000 |
| Schema semantic error (multiple things, semantic) | Parses but incorrect "meaning" like relation ambiguity               | VE     | P1001 |
| Schema string input validation                    | Database URL in Prisma schema fails to parse                         | VE     | P1002 |
| Incorrect database credentials                    | User inputted wrong database credentials                             | VE     | P1003 |
| Database not reachable                            | The URL is not reachable (bad URL or network too)                    | VE     | P1004 |
| Value too long                                    | The value is too long for the column                                 | DE     | P1010 |
| Record not found error                            | The record specified by where condition does not exist               | DE     | P1011 |
| Unique key violation error                        | `UniqueConstraintViolation: Unique constraint failed: ${field_name}` | DE     | P1012 |
| Foreign key violation error                       | Database level constraint violation                                  | DE     | P1013 |
| Custom constraint violation                       | Database level constraint violation                                  | DE     | P1014 |
| Stored value is invalid                           | Example: https://github.com/prisma/photonjs/issues/214               | DE     | P1015 |
| Database errors - corruption                      | The database is reachable but corrupted                              | RE     | P1020 |
| Database timeout                                  | Credentials are correct but request timed out                        | RE     | P1021 |
| Database does not exist                           | A database with the provided name does not exist                     | RE     | P1022 |
| Database creation failed                          | `DatabaseCreationError: Database creation error: ${error}`           | RE     | P1023 |
| Binary for the wrong platform                     | The downloaded binary is not compiled for this platform              | RE     | P1030 |
| Unable to bind binary to a free port              | Port already in use                                                  | RE     | P1031 |

### Unclassified SDK errors

| Title                                                     | Description                                                                       | Prefix | Code |
| --------------------------------------------------------- | --------------------------------------------------------------------------------- | ------ | ---- |
| Type mismatch - invalid ID                                | Invalid UUID in String (TODO: Open question: shim)                                | ???    | ???  |
| Type mismatch - invalid Date                              | https://github.com/prisma/photonjs/issues/212 (TODO: Open question: shim)         | ???    | ???  |
| Type mismatch - invalid JSON                              | https://github.com/prisma/photonjs/issues/60 (TODO: Open question: shim/coercion) | ???    | ???  |
| Type mismatch - invalid custom type                       | https://github.com/prisma/specs/issues/119 (TODO: Open question: shim/coercion)   | ???    | ???  |
| Incorrect migration generated (multiple things, rollback) | The generated migration is incorrect due to invalid query or existing tables      | ??     | ???  |
| Destructive migration                                     | Migration possible with destructive changes i.e. data loss                        | ??     | ???  |
| Introspection failure                                     | Introspection failed to produce a schema file                                     | ??     | ???  |

## Photon.js

| Title                         | Description                                           | Prefix | Code |
| ----------------------------- | ----------------------------------------------------- | ------ | ---- |
| Invalid Photon call           | Querying non-existent fields or wrong arguments, etc. | VE     | ???  |
| Query engine connection error | Query engine was started but the process died         | RE     | ???  |

Photon relays the following errors from the SDK: `P1004`, `P1010` , `P1010`, `P1011`, `P1012`, `P1013`, `P1014`, `P1015`,

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

TODO: Introspection failure, database creation failure

## Prisma CLI

### Init

| Title                                  | Description                             | Prefix | Code |
| -------------------------------------- | --------------------------------------- | ------ | ---- |
| Directory already contains schema file | Directory is an existing Prisma project | VE     | ???  |
| Starter kit                            | Directory is not empty                  | VE     | ???  |

More issues for init command failures are covered here: https://prisma-specs.netlify.com/cli/init/errors/

TODO: Introspection failure, database creation failure (P1023)

### Generate

TODO: NA

### Dev

TODO: database creation failure (P1023)

### Lift

TODO: Destructive migration

### Introspect

TODO: NA

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

// TODO: Replace this with actual CLI telemetry spec once that is available.

# Error Log Masking

Both logging output, error report might contain logs with sensitive information like database URL. Prisma 2 should mask the sensitive information (with asterisks `********`) before dumping the data on the file system.

The error report to be sent back automatically might also contain some proprietary information like the database schema via Prisma schema file.

We must ask the user before collecting such information. This is covered in the [telemetry spec](https://prisma-specs.netlify.com/cli/telemetry/).

// TODO: Replace this CLI telemetry spec link with general telemetry spec link once that is available.

# Error Character Encoding

// TODO: This probably belongs to Photon.js error handling spec but parts of it like making it parsable and error codes belong to this spec.

Photon generates pretty error messages with colors that are very useful for development as they usually pin point the issue.

However, when this data is serialized it contains a lot of unicode characters.

<details><summary>Serialized Photon error</summary>
<p>
```
Error: ^[[31mInvalid ^[[1m`const data = await photon.users.findMany()`^[[22m invocation in ^[[4m/Users/divyendusingh/Documents/prisma/p2-studio/index.js:8:37^[[24m^[[39m ^[[2m ^[[90m 4 ^[[39m^[[36mconst^[[39m photon = ^[[36mnew^[[39m Photon^[[38;2;107;139;140m(^[[39m^[[38;2;107;139;140m)^[[39m^[[22m ^[[2m ^[[90m 5 ^[[39m^[[22m ^[[2m ^[[90m 6 ^[[39m^[[36masync^[[39m ^[[36mfunction^[[39m ^[[36mmain^[[39m^[[38;2;107;139;140m(^[[39m^[[38;2;107;139;140m)^[[39m ^[[38;2;107;139;140m{^[[39m^[[22m ^[[2m ^[[90m 7 ^[[39m ^[[36mtry^[[39m ^[[38;2;107;139;140m{^[[39m^[[22m ^[[31m^[[1m→^[[22m^[[39m ^[[90m 8 ^[[39m ^[[36mconst^[[39m data = ^[[36mawait^[[39m photon^[[38;2;107;139;140m.^[[39musers^[[38;2;107;139;140m.^[[39m^[[36mfindMany^[[39m^[[38;2;107;139;140m(^[[39m{ ^[[91munknown^[[39m: ^[[2m'1'^[[22m^[[2m^[[22m ^[[91m~~~~~~~^[[39m ^[[2m^[[22m}^[[2m)^[[22m Unknown arg ^[[91m`unknown`^[[39m in ^[[1munknown^[[22m for type ^[[1mUser^[[22m. Did you mean `^[[92mskip^[[39m`? ^[[2mAvailable args:^[[22m ^[[2mtype^[[22m ^[[1m^[[2mfindManyUser^[[1m^[[22m ^[[2m{^[[22m ^[[2m^[[32mwhere^[[39m?: ^[[37mUserWhereInput^[[39m^[[22m ^[[2m^[[32morderBy^[[39m?: ^[[37mUserOrderByInput^[[39m^[[22m ^[[2m^[[32mskip^[[39m?: ^[[37mInt^[[39m^[[22m ^[[2m^[[32mafter^[[39m?: ^[[37mString^[[39m^[[22m ^[[2m^[[32mbefore^[[39m?: ^[[37mString^[[39m^[[22m ^[[2m^[[32mfirst^[[39m?: ^[[37mInt^[[39m^[[22m ^[[2m^[[32mlast^[[39m?: ^[[37mInt^[[39m^[[22m ^[[2m}^[[22m
```
</p>
</details>

There are two use cases for disabling/better structuring the error logs:

1. In Production logs, one might want to read the error messages thrown by Photon.

2. In tools like Studio, it currently strips the ANSI characters (like [this](https://codesandbox.io/s/photon-pretty-errors-m4l77)) and displays the output as the error message.

To solve these two use case, Photon can do the following:

1. Respect `NODE_ENV`

This would solve the production logs use case. When `NODE_ENV` is set to `development` or is unset, Photon can emit logs with colors. However, when `NODE_ENV` is set to `production` Photon can omit colors from the logs.

2. While Point 1 might already be enough for tools like Studio as well, Photon can additionally offer `prettyLogs` as a constructor argument to switch off the pretty error logging.

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

- Do we need the prefixes (Like VR, DE, RE) for error classification?
- Do we need error codes for anything other than the SDK?
