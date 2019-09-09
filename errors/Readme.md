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
  * [Prisma Error](#prisma-error)
  * [OS Error](#os-error)
  * [Data Source Error](#data-source-error)
  * [Domain Error](#domain-error)
- [Error Codes](#error-codes)
  * [Prefix](#prefix)
  * [Error Code Numbers](#error-code-numbers)
- [Known Errors](#known-errors)
  * [Prisma SDK](#prisma-sdk)
    + [Unclassified SDK errors](#unclassified-sdk-errors)
  * [Photon.js](#photonjs)
  * [Prisma Studio](#prisma-studio)
  * [Prisma CLI](#prisma-cli)
    + [Init](#init)
    + [Dev](#dev)
    + [Generate](#generate)
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

<!-- tocstop -->

# Motivation

![Prisma 2 architecture diagram with errors overlay](./errors-spec.png)

// TODO: Move image to figma

| Component   | Description                                                                                 |
| ----------- | ------------------------------------------------------------------------------------------- |
| SDK Users   | Photon.js, Studio, Prisma CLI: dev, lift, generate commands, etc.                           |
| Prisma SDK  | Helps tools interact with binaries. Spec [here](../sdk-js/Readme.md)                        |
| Binaries    | Artifacts of Prisma 2 compilation and a part of the SDK. Spec [here](../binaries/Readme.md) |
| Data source | A database or any other data source supported by Prisma                                     |

Prisma 2 ecosystem provides various layers of tools that communicate with each other to provide desired outcome.

Broadly, the life-cycle of a request (Query request or CLI command) can be seen in this architecture diagram.

| SDK User | →   | Prisma SDK | →   | Binaries | →   | Data source |
| -------- | --- | ---------- | --- | -------- | --- | ----------- |


// TODO: Request for feedback, an alternative is to let the lower level bubble up as is. Which is also an easier implementation. Right now there is no concrete use case for having separate error codes for the same error across layers (except attempting to save people from underlying component knowledge)
**Note**: An error can happen in any parts of this stack and it bubbles up from that part to the surface area. For example, a unique constraint violation happens in the data source and the first programmatically parsable occurrence happens in the binary. However the same error is handled by Prisma SDK and eventually by the consumer of the SDK (say Photon). We have to document the same error in all these different specs as it bubbles up, thus defining a clear errors interface at each layer. This spec will eventually contain links to this distributed spec work.

This document covers our error handling strategy and covers the following:

1. Known/unknown errors
2. Error log masking
3. Error character encoding

As the Prisma SDK acts as the interface between binaries and SDK clients. A lot of error handling is managed in SDK layer.

# Error Causes, Codes, and Handling Strategies

// TODO: This is just an approach to try and list errors based on their classification (irrespective of the layers in Prisma stack). This is useful to show errors to people who are /// not aware of underlying components. An alternative can be to base prefixes on components.

There can be various reasons for a request/operation to fail. This section broadly classifies potential error causes and handling strategies.

## Validation Errors

These are usually caused by faulty user input. For example,

- Incorrect database credentials
- Invalid data source URL
- Malformed schema syntax
- Invalid type input to Photon (`String` in place of `Int` for example)

Handling strategy: Any user input must be validated and user should be notified about the validation error details.

## Prisma Error

Note: This is actually similar to an unknown error and wrapping this with an error code might not be possible from an implementation perspective.

These are caused due to a bug in a Prisma component. For example,

- Photon builds incorrect query for the query engine
- Lift generates incorrect migration steps

This occurrence of these should be rare.

Handling strategy: User should be notified about the Prisma bug and should be prompted to send the error report to Prisma team manually or via telemetry.

TODO: These are maybe just bugs and maybe do not belong to this spec.

## OS Error

These are caused due to an OS system call failure. For example,

- The available binary is not compiled for this platform
- SDK failed to bind a port for query engine

Handling strategy: Notify the user by relaying the message from OS and suggesting them to retry. They might need to free up resources or do something at the OS level.

In certain cases, like when a port collision, Prisma SDK can try to retry gracefully as well with a different port.

## Data Source Error

These are caused when a data source lacks availability, For example,

- Database is not reachable
- Database timeout

Handling strategy: Notify the user by relaying the error message from data source.

## Domain Error

TODO: This needs better naming based on additional feedback. Maybe "Constraint Error" or "Data Error" is a better term.

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

| Error Cause        | Prefix | Range |
| ------------------ | ------ | ----- |
| Validation Error   | VE     | xx0x  |
| Prisma Error (Bug) | PE     | xx1x  |
| Domain Error       | DE     | xx2x  |
| Data Source Error  | DSE    | xx3x  |
| OS Error           | OSE    | xx4x  |

## Error Code Numbers

An error code should provide users context of what might be wrong and where. This requires product wise error code numbering alongside an error category prefix defined in the last section.

| Component                      | Range     |
| ------------------------------ | --------- |
| Binaries                       | xxxx      |
| SDK                            | 10xx      |
| Photon.js                      | 20xx      |
| Studio                         | 30xx      |
| CLI                            | 40xx      |
| Custom Generators, Nexus, etc. | 50xx-90xx |

Note: Binaries are only consumed through the SDK. All known binary errors will have a defined error interface in SDK. Hence, there is no special range/code for binary only errors.

Example of this wrt how a single error would propagate:

| Component | Example                   | Category | Range | Code   |
| --------- | ------------------------- | -------- | ----- | ------ |
| SDK       | Null constraint exception | DE       | 20xx  | DE10xx |
| Photon    | Null constraint exception | DE       | 30xx  | DE20xx |

This means that a single error will have separate error codes based on where is it encountered. This is important though because multiple tools might consume the SDK and have variation in error handling.

# Known Errors

## Prisma SDK

SDK acts as the interface between the binaries and the tools. This section covers errors from SDK, binaries and the network between SDK ⇆ Binary and Binary ⇆ Data source.

| Title                                                     | Description                                                                  | Prefix | Code    |
| --------------------------------------------------------- | ---------------------------------------------------------------------------- | ------ | ------- |
| Schema parsing error (multiple things, syntax)            | User input schema is invalid                                                 | VE     | VE1000  |
| Schema semantic error (multiple things, semantic)         | Parses but incorrect "meaning" like relation ambiguity                       | VE     | VE1001  |
| Schema string input validation                            | Database URL in Prisma schema fails to parse                                 | VE     | VE1002  |
| Incorrect database credentials                            | User inputted wrong database credentials                                     | VE     | VE1003  |
| Database not reachable                                    | The URL is not reachable (bad URL or network too)                            | VE     | VE1004  |
| Incorrect Prisma query generated                          | Query built by query engine is incorrect                                     | PE     | PE1010  |
| Incorrect migration generated (multiple things, rollback) | The generated migration is incorrect due to invalid query or existing tables | PE     | PE1011  |
| Destructive migration                                     | Migration possible with destructive changes i.e. data loss                   | PE     | PE1012  |
| Introspection failure                                     | Introspection failed to produce a schema file                                | PE     | PE1013  |
| Incorrect DMMF generated                                  | Schema is parsed but DMMF is incorrect                                       | PE     | PE1014  |
| Value too long                                            | The value is too long for the column                                         | DE     | DE1020  |
| Record not found error                                    | The record specified by where condition does not exist                       | DE     | DE1021  |
| Unique key violation error                                | `UniqueConstraintViolation: Unique constraint failed: ${field_name}`         | DE     | DE1022  |
| Foreign key violation error                               | Database level constraint violation                                          | DE     | DE1023  |
| Custom constraint violation                               | Database level constraint violation                                          | DE     | DE1024  |
| Stored value is invalid                                   | Example: https://github.com/prisma/photonjs/issues/214                       | DE     | DE1025  |
| Database errors - corruption                              | The database is reachable but corrupted                                      | DSE    | DSE1030 |
| Database timeout                                          | Credentials are correct but request timed out                                | DSE    | DSE1031 |
| Database does not exist                                   | A database with the provided name does not exist                             | DSE    | DSE1032 |
| Database creation failed                                  | `DatabaseCreationError: Database creation error: ${error}`                   | DSE    | DSE1033 |
| Binary for the wrong platform                             | The downloaded binary is not compiled for this platform                      | OSE    | DSE1040 |
| Unable to bind binary to a free port                      | Port already in use                                                          | OSE    | DSE1041 |

### Unclassified SDK errors

| Title                               | Description                                                                       | Prefix | Code |
| ----------------------------------- | --------------------------------------------------------------------------------- | ------ | ---- |
| Type mismatch - invalid ID          | Invalid UUID in String (TODO: Open question: shim)                                | ???    | ???  |
| Type mismatch - invalid Date        | https://github.com/prisma/photonjs/issues/212 (TODO: Open question: shim)         | ???    | ???  |
| Type mismatch - invalid JSON        | https://github.com/prisma/photonjs/issues/60 (TODO: Open question: shim/coercion) | ???    | ???  |
| Type mismatch - invalid custom type | https://github.com/prisma/specs/issues/119 (TODO: Open question: shim/coercion)   | ???    | ???  |

## Photon.js

| Title                         | Description                                                                                                  | Prefix | Code    | Base Code |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------ | ------ | ------- | --------- |
| Invalid Photon call           | In JS or TS with "any" cast. This could be querying non-existent fields or wrong arguments or incorrect null | VE     | VE2000  |           |
| Photon query building error   | Photon fails to build the query correctly                                                                    | PE     | PE2010  |           |
| Data source Connection error  | Query engine is reachable but database is not responding                                                     | PE     | PE2011  | VE1004    |
| Value too long                | The value is too long for the column                                                                         | DE     | DE2020  | VE1020    |
| Record not found error        | The record specified by where condition does not exist                                                       | DE     | DE2021  | DE1021    |
| Unique key violation error    | `UniqueConstraintViolation: Unique constraint failed: ${field_name}`                                         | DE     | DE2022  | DE1022    |
| Foreign key violation error   | Database level constraint violation on foreign key                                                           | DE     | DE2023  | DE1023    |
| Custom constraint violation   | Database level constraint violation                                                                          | DE     | DE2024  | DE1024    |
| Stored value is invalid       | Example: https://github.com/prisma/photonjs/issues/214                                                       | DE     | DE2025  | DE1025    |
| Query engine connection error | Query engine was started but the process died                                                                | OSE    | OSE2040 |           |

Since Photon results are consumed in user-land, it should have a parsable error structure:

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

Studio has two workflows:

Electron app: Credentials from the UI → Introspection → Prisma schema → Valid Prisma project
Web app: `prisma2 dev` → Provides Prisma schema i.e a Valid Prisma project

| Title                           | Description                                   | Prefix | Code   | Base Code |
| ------------------------------- | --------------------------------------------- | ------ | ------ | --------- |
| Invalid Photon call generated   | Studio builds invalid Photon call             | PE     | PE3000 |           |
| Electron: Introspection failure | Introspection failed to produce a schema file | PE     | PE3001 | PE1013    |
| Incorrect DMMF                  | DMMF generated from the schema has a bug      | PE     | PE3010 | PE1013    |

## Prisma CLI

### Init

| Title                                  | Description                                                | Prefix | Code    | Base Code |
| -------------------------------------- | ---------------------------------------------------------- | ------ | ------- | --------- |
| Directory already contains schema file | Directory is an existing Prisma project                    | VE     | VE4000  |           |
| Starter kit                            | Directory is not empty                                     | VE     | VE4001  |           |
| Introspection failure                  | Introspection failed to produce a schema file              | PE     | PE4010  | PE1013    |
| Database creation failed               | `DatabaseCreationError: Database creation error: ${error}` | DSE    | DSE4030 | DSE1033   |

More issues for init command failures are covered here: https://prisma-specs.netlify.com/cli/init/errors/

### Dev

| Title                    | Description                                                | Prefix | Code    | Base Code |
| ------------------------ | ---------------------------------------------------------- | ------ | ------- | --------- |
| Database creation failed | `DatabaseCreationError: Database creation error: ${error}` | DSE    | DSE4031 | DSE1033   |

### Generate

| Title          | Description                              | Prefix | Code   | Base Code |
| -------------- | ---------------------------------------- | ------ | ------ | --------- |
| Incorrect DMMF | DMMF generated from the schema has a bug | PE     | PE4010 | PE1013    |

### Lift

| Title                      | Description                                                                  | Prefix | Code   | Base Code |
| -------------------------- | ---------------------------------------------------------------------------- | ------ | ------ | --------- |
| Migration failure          | The generated migration is incorrect due to invalid query or existing tables | PE     | PE4010 | PE1011    |
| Destructive steps detected | Migration possible with destructive changes i.e. data loss                   | PE     | PE4011 | PE1012    |

### Introspect

| Title                 | Description                                   | Prefix | Code   | Base Code |
| --------------------- | --------------------------------------------- | ------ | ------ | --------- |
| Introspection failure | Introspection failed to produce a schema file | PE     | PE4010 | PE1013    |

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
