# CLI spec

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [View Online](#view-online)
- [Usage](#usage)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Functional Spec

Prisma CLI offers essential functionality to Prisma users:

### Setup Prisma

`prisma2 init`

### Create datamodel based on existing database 

`prisma2 introspect`

=> Introspection

### Iterate on datamodel (watch mode)

`prisma2 dev`

=> Studio

### Migrate datamodel 

`prisma2 lift`

=> Lift

### Generate Photon client 

`prisma2 generate`

=> Photon

### Fill data with test data 

`prisma2 seed`

=> ???



## Technical Design Spec

### View Online

https://prisma-specs.netlify.com/cli/

### Usage

Edit [`src/pages/index.mdx`](src/pages/index.mdx) and run

```
yarn start
```
