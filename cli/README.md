# CLI spec

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Functional Spec](#functional-spec)
  - [Setup Prisma](#setup-prisma)
  - [Create datamodel based on existing database](#create-datamodel-based-on-existing-database)
  - [Iterate on datamodel (watch mode)](#iterate-on-datamodel-watch-mode)
    - [Interactive](#interactive)
    - [Non-Interactive](#non-interactive)
  - [Migrate datamodel](#migrate-datamodel)
  - [Generate Photon client](#generate-photon-client)
  - [Fill data with test data](#fill-data-with-test-data)
  - [Convert  Prisma 1.x service configuration to Prisma 2 schema file.](#convert--prisma-1x-service-configuration-to-prisma-2-schema-file)
- [Technical Design Spec](#technical-design-spec)
  - [View Online](#view-online)
  - [Usage](#usage)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Problem, Idea, Concept

Prisma CLI offers essential functionality to Prisma users.

- Interactive commands
- Non Interactive commands (scriptable, automation, CI)

## Use Cases

1. Install Prisma
1. Set up Prisma in existing project with existing database
1. Create new project with Prisma
1. Play with and experience Prisma
1. Iterate on Datamodel
1. Migrate with Lift
1. Generate Photon.js
1. Help Prisma 1 users to upgrade

## Installation

TODO

- npm/yarn default
- All other options as fallback: Homebrew, curl, chocolatey, ...


## Commands

### Setup Prisma

`prisma2 init`

???

### Create datamodel based on existing database 

`prisma2 introspect`

=> Introspection

### Iterate on datamodel (watch mode)

`prisma2 dev`

???

#### Interactive

???

#### Non-Interactive

???

Also:
=> Studio

### Migrate datamodel 

`prisma2 lift x`

=> Lift

### Generate Photon client 

`prisma2 generate`

=> Photon

### Fill data with test data 

`prisma2 seed`

=> ???

### Convert  Prisma 1.x service configuration to Prisma 2 schema file.

`prisma2 convert`

???


## Technical Design Spec

### View Online

https://prisma-specs.netlify.com/cli/

### Usage

Edit [`src/pages/index.mdx`](src/pages/index.mdx) and run

```
yarn start
```
