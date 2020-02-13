# Platform Interface Specification

This document specifies the interface between a lifecycle and a platform.

A platform orchestrates a lifecycle to make buildpack functionality available to end-users such as application developers.

Examples of a platform might include:

1. A local CLI tool that uses buildpacks to create OCI images
1. A plugin for a continuous integration service that uses buildpacks to create OCI images
1. A cloud application platform that uses buildpacks to build source code before deployment

## Table of Contents

1. [Platform API Version](#platform-api-version)
   1. [Compatibility Verification](#compatibility-verification)
1. [Stacks](#stacks)
   1. [Compatibility Guarantees](#compatibility-guarantees)
   1. [Build Image](#build-image)
   1. [Run Image](#run-image)
1. [Buildpacks](#buildpacks)
   1. [Buildpacks Directory Layout](#buildpacks-directory-layout)
1. [Security Considerations](#security-considerations)
1. [Additional Guidance](#additional-guidance)
   1. [Environment](#environment)
   1. [Run Image Rebasing](#run-image-rebasing)
   1. [Caching](#caching)
1. [Data Format](#data-format)
   1. [order.toml (TOML)](#order.toml-(toml))
   1. [group.toml (TOML)](#group.toml-(toml))
   1. [analyzed.toml (TOML)](#group.toml-(toml))
   1. [stack.toml (TOML)](#group.toml-(toml))


## Platform API Version

The Platform API version:
 - MUST be in form `<major>.<minor>` or `<major>`, where `<major>` is equivalent to `<major>.0`
 - MUST describe the implemented Platform API.
 - SHALL indicate compatibility with a given lifecycle according to the following rules:
    - When `<major>` is `0`, the platform is only compatible with lifecycles implementing that exact Platform API.
    - When `<major>` is greater than `0`, the platforms is only compatible with lifecycles implementing platform API
    `<major>.<minor>`, where `<major>` of the lifecycle equals `<major>` of the platform and `<minor>` of the lifecycle
    is greater than or equal to `<minor>` of the platform.

### Compatibility Verification

The lifecycle SHALL verify compatibility if the environment variable `CNB_PLATFORM_API` is set. The value of this
environment variable MUST be the version of the Platform API the platform implements. Compatibility verification SHALL
NOT occur if this environment variable is not set. Compatibility verification SHALL occur before any other validation.

## Stacks

A **stack** is a contract defined by a base run OCI image and a base build OCI image that are only updated with security patches.
Stack images can be modified with mixins in order to make additive changes to the contract.

A **mixin** is a named set of additions to a stack that avoid changing the behavior of buildpacks or apps that do not depend on the mixin.

A **launch layer** refers to a layer in the app OCI image created from a  `<layers>/<layer>` directory as specified in the [Buildpack Interface Specification](buildpack.md).

An **app layer** refers to a layer created from the `<app>` directory as specified in the [Buildpack Interface Specification](buildpack.md).

### Compatibility Guarantees

Stack image authors SHOULD ensure that build image versions maintain [ABI-compatibility](https://en.wikipedia.org/wiki/Application_binary_interface) with previous versions, although violating this requirement will not change the behavior of previously built images containing app and launch layers.

Stack image authors MUST ensure that new run image versions maintain [ABI-compatibility](https://en.wikipedia.org/wiki/Application_binary_interface) with previous versions.
Stack image authors MUST ensure that app and launch layers do not change behavior when the run image layers are upgraded to newer versions, unless those behavior changes are intended to fix security vulnerabilities.

Mixin authors MUST ensure that mixins do not affect the [ABI-compatibility](https://en.wikipedia.org/wiki/Application_binary_interface) of any object code compiled to run on the base stack images without mixins.

During build, platforms MUST use the same set of mixins for the run image as were used in the build image (excluding mixins that have a stage specifier).

### Build Image

The platform MUST ensure that:

- The image config's `User` field is set to a non-root user with a writable home directory.
- The image config's `Env` field has the environment variable `CNB_STACK_ID` set to the stack ID.
- The image config's `Env` field has the environment variable `CNB_USER_ID` set to the UID of the user specified in the `User` field.
- The image config's `Env` field has the environment variable `CNB_GROUP_ID` set to the primary GID of the user specified in the `User` field.
- The image config's `Label` field has the label `io.buildpacks.stack.id` set to the stack ID.
- The image config's `Label` field has the label `io.buildpacks.stack.mixins` set to a JSON array containing mixin names for each mixin applied to the image.

### Run Image

The platform MUST ensure that:

- The image config's `User` field is set to a user with the same UID and primary GID as in the build image.
- The image config's `Label` field has the label `io.buildpacks.stack.id` set to the stack ID.
- The image config's `Label` field has the label `io.buildpacks.stack.mixins` set to a JSON array containing mixin names for each mixin applied to the image.

### Mixins

A mixin name MUST only be defined by the author of its corresponding stack.
A mixin name MUST always be used to specify the same set of changes.
A mixin name MUST only contain a `:` character as part of an optional stage specifier.

A mixin prefixed with the `build:` stage specifier only affects the build image and does not need to be specified on the run image.
A mixin prefixed with the `run:` stage specifier only affects the run image and does not need to be specified on the build image.

A platform MAY support any number of mixins for a given stack in order to support application code or buildpacks that require those mixins.

Changes introduced by mixins SHOULD be restricted to the addition of operating system software packages that are regularly patched with strictly backwards-compatible security fixes.
However, mixins MAY consist of any changes that follow the [Compatibility Guarantees](#compatibility-guarantees).

## Lifecycle Interface

A lifecycle 

The following specifies the interface implemented by executables in each buildpack.

### Key

| Mark    | Meaning
|---------|-------------------------------------------
| O       | Path to which output should be written
| P       | Path from which input should be read
| I(T)    | Image (tag) reference
| R       | Read-only


This executable MUST resolve input values according to the following rules:
* When both a flag it's correspond environment variable are provided, the executable:
  - MUST assume the input to be the argument to the flag and ignore environment variable
* When neither a flag nor it's corresponding environment are provided, the executable:
  - MUST assume the input to be the default value

### Detection
Executable: 
```
/cnb/lifecycle/detector \
  [-buildpacks <buildpacksR>] \
  [-app <app>] \
  [-platform <platformR>] \
  [-order <orderP>] \
  [-group <orderO>] \
  [-plan <planO>] \
  [-log-level <planO>] \
```

| Input        | Flag           | Env                 | Default Value   | Description
|--------------|----------------|---------------------|-----------------|----------------------
| <buildpacks> | -buildpacks    | CNB_BUILDPACKS_DIR  | /cnb/buildpacks | Path to buildpacks directory (see [Buildpacks Directory Layout](#buildpacks-directory-layout))
| <app>        | -app           | CNB_APP_DIR         | /workspace      | Path to application directory
| <platform>   | -platform      | CNB_PLATFORM_DIR    | /platform       | Path to platform directory
| <order>      | -order         | CNB_ORDER_PATH      | ./order.toml    | Path to order definition ( see [order.toml (TOML)](#order.toml-(toml)))
| <group>      | -group         | CNB_GROUP_PATH      | ./group.toml    | Path to output group file ( see [group.toml (TOML)](#group.toml-(toml)))
| <plan>       | -plan          | CNB_PLAN_PATH       | ./plan.toml     | Path to output build plan file ( see  data format in [Buildpack Interface Specification](buildpack.md))

| Output             | Description
|--------------------|----------------------------------------------
| [exit status]      | Pass (0), fail (100), or error (1-99, 101+)
| `/dev/stdout`      | Logs (info)
| `/dev/stderr`      | Logs (warnings, errors)
| `<group>`          | Detected buildpack group
| `<plan>`           | Build Plan of detector group

### Analysis
Executable: `/cnb/lifecycle/analyzer <image>`

This executable MUST resolve the input values according to the following rules:
* When both a flag it's correspond environment variable are provided, the executable:
  - MUST assume the input to be the argument to the flag and ignore environment variable
* When neither a flag nor it's corresponding environment are provided, the executable:
  - MUST assume the input to be the default value
  
`/cnb/lifecycle/analyzer` MUST accept the following inputs

| Input          | Flag           | Env                 | Default Value   | Description
|----------------|----------------|---------------------|-----------------|----------------------
| `<group>`      | -group         | CNB_GROUP_PATH      | ./group.toml    | Path to group definition ( see [group.toml (TOML)](#group.toml-(toml)))
| `<layers>`     | -layers        | CNB_LAYERS_DIR      | /layers         | Path to layer directory
| `<skip-layers>`| -skip-layers   | CNB_SKIP_LAYERS     | false           | Do not write layer metadata
| `<analyzed>`   | -analyzed      | CNB_ANALYZED_PATH   | ./analyzed.toml | Path to output analysis metadata ( see [analyzed.toml (TOML)](#analyzed.toml-(toml))
| `<use-daemon>` | -daemon        | CNB_USE_DAEMON      | false           | Read image config blob from docker daemon
| `<cache-dir>`  | -daemon        | CNB_CACHE_DIR       | /cache          | Location of cache directory
| `<uid>`        | -uid           | CNB_USER_ID         | -               | User that build phase will run as
| `<gid>`        | -gid           | CNB_GROUP_ID        | -               | Group of user that build phase will run as

`/cnb/lifecycle/analyzer` MAY accept additional inputs to support other cache implementations

### Cache Restoration
Executable: `/cnb/lifecycle/restorer`

This executable MUST resolve the input values according to the following rules:
* When both a flag it's correspond environment variable are provided, the executable:
  - MUST assume the input to be the argument to the flag and ignore environment variable
* When neither a flag nor it's corresponding environment are provided, the executable:
  - MUST assume the input to be the default value
  
`/cnb/lifecycle/restorer` MUST accept the following inputs

| Input        | Flag           | Env                 | Default Value   | Description
|--------------|----------------|---------------------|-----------------|----------------------
| <group>      | -group         | CNB_GROUP_PATH      | ./group.toml    | Path to group definition ( see [group.toml (TOML)](#group.toml-(toml)))
| <layers>     | -layers        | CNB_LAYERS_DIR      | /layers         | Path to layer directory
| <cache-dir>  | -daemon        | CNB_CACHE_DIR       | /cache          | Location of cache directory
| <uid>        | -uid           | CNB_USER_ID         | -               | User that build phase will run as
| <gid>        | -gid           | CNB_GROUP_ID        | -               | Group of user that build phase will run as

`/cnb/lifecycle/restorer` MAY accept additional inputs to support other cache implementations

### Build
Executable: `/cnb/lifecycle/builder`

This executable MUST resolve input values according to the following rules:
* When both a flag it's correspond environment variable are provided, the executable:
  - MUST assume the input to be the argument to the flag and ignore environment variable
* When neither a flag nor it's corresponding environment are provided, the executable:
  - MUST assume the input to be the default value

| Input        | Flag           | Env                 | Default Value   | Description
|--------------|----------------|---------------------|-----------------|----------------------
| <buildpacks> | -buildpacks    | CNB_BUILDPACKS_DIR  | /cnb/buildpacks | Path to buildpacks directory (see [Buildpacks Directory Layout](#buildpacks-directory-layout))
| <app>        | -app           | CNB_APP_DIR         | /workspace      | Path to application directory
| <layers>     | -layers        | CNB_LAYERS_DIR      | /layers         | Path to layer directory
| <platform>   | -platform      | CNB_PLATFORM_DIR    | /platform       | Path to platform directory
| <group>      | -group         | CNB_GROUP_PATH      | ./group.toml    | Path to group file ( see [group.toml (TOML)](#group.toml-(toml)))
| <plan>       | -plan          | CNB_PLAN_PATH       | ./plan.toml     | Path to output build plan file ( see  data format in [Buildpack Interface Specification](buildpack.md))


### Export
Executable: `/cnb/lifecycle/exporter <image>`

This executable MUST resolve input values according to the following rules:
* When both a flag it's correspond environment variable are provided, the executable:
  - MUST assume the input to be the argument to the flag and ignore environment variable
* When neither a flag nor it's corresponding environment are provided, the executable:
  - MUST assume the input to be the default value

| Input        | Flag           | Env                 | Default Value   | Description
|--------------|----------------|---------------------|-----------------|----------------------
| <group>      | -group         | CNB_GROUP_PATH      | ./group.toml    | Path to group definition ( see [group.toml (TOML)](#group.toml-(toml)))
| <layers>     | -layers        | CNB_LAYERS_DIR      | /layers         | Path to layer directory
| <skip-layers>| -skip-layers   | CNB_SKIP_LAYERS     | false           | Do not write layer metadata
| <analyzed>   | -analyzed      | CNB_ANALYZED_PATH   | ./analyzed.toml | Path to output analysis metadata ( see [analyzed.toml (TOML)](#analyzed.toml-(toml))
| <use-daemon> | -daemon        | CNB_USE_DAEMON      | false           | Read image config blob from docker daemon
| <cache-dir>  | -daemon        | CNB_CACHE_DIR       | /cache          | Location of cache directory
| <uid>        | -uid           | CNB_USER_ID         | -               | User that build phase will run as
| <gid>        | -gid           | CNB_GROUP_ID        | -               | Group of user that build phase will run as

## Build

To create an app OCI image a platform MUST invoke the following executables in order in the build environment

1. `/cnb/lifecycle/detector`
1. `/cnb/lifecycle/analyzer`
1. `/cnb/lifecycle/restorer`
1. `/cnb/lifecycle/builder`
1. `/cnb/lifecycle/exporter`


## Buildpacks

### Buildpacks Directory Layout

The buildpacks directory MUST contain unarchived buildpacks such that:

- Each top-level directory is a buildpack ID.
- Each second-level directory is a buildpack version.

## Security Considerations

The platform SHOULD run each phase of the lifecycle in an isolated container to prevent untrusted app and buildpack code from accessing storage credentials needed during the export and analysis phases.
A more thorough explanation is provided in the [Buildpack Interface Specification](buildpack.md).

## Additional Guidance

### Environment

User-provided environment variables intended for build and launch SHOULD NOT come from the same list.
The end-user SHOULD be encouraged to define them separately.
The platform MAY determine the initial environment of the build phase, detection phase, and launch.
The lifecycle MUST NOT assume that all platforms provide an identical environment.

### Run Image Rebasing

Run image rebasing allows for fast stack updates for already-exported OCI images with minimal data transfer when those images are stored on a Docker registry.
When a new stack version with the same stack ID is available, the app layers and launch layers SHOULD be rebased on the new run image by updating the image's configuration to point at the new run image.
Once the new run image is present on the registry, filesystem layers SHOULD NOT be uploaded or downloaded.

The new run image MUST have an identical stack ID and MUST include the exact same set of mixins.

![Launch](img/launch.svg)

### Caching

Each platform SHOULD implement caching so as to appropriately optimize performance.
Cache locality and availability MAY vary between platforms.

## Data Format

### order.toml (TOML)

```toml
[[order]]
[[order.group]]
id = "<buildpack ID>"
version = "<buildpack version>"
optional = false
```

Where:

- Both `id` and `version` MUST be present for each buildpack object in a group.
- The value of `optional` MUST default to false if not specified.

### group.toml (TOML)

```toml
group = [
  { id = "<buildpack ID>", version = "<buildpack version>" }
]
```

Where:

- Both `id` and `version` MUST be present for each buildpack object in a group.

### analyzed.toml (TOML)

```toml
[image]
  reference = "<image reference>"

[metadata]
# layer metadata
```

Where:
- `image.reference` MUST be EITHER a digest reference to an image in a docker registry or the ID of an image in a docker daemon
- `metadata` MUST be the TOML representation fo the layer [metadata label](#layer-metadata-label-json)

### stack.toml (TOML)

```toml
[run-image]
 image = "<image>"

[run-image.mirrors] = ["<mirror>", "mirror"]
```

Where:
- `run-image.image` MAY be a tag reference to an run image in a docker registry
- `run-image.mirrors` MUST NOT be present IF `run-image.image` is not present
- `run-image.mirrors` MAY contain one or more tag references to run images in docker registries
- all present `run-image-mirrors`:
  * MUST resolve to a digest reference identical to that which `run-image.image` resolves
  * MUST NOT refer to the same registry as does `run-image.image` or any other entries `run-image.mirrors`

### layer metadata label (JSON)
```json
{
  "app": [
    {"sha": "<slice-layer-diffID>"}
  ],
  "config": {
    "sha": "<config-layer-diffID>"
  },
  "launcher": {
    "sha": "<launcher-layer-diffID>"
  },
  "buildpacks": [
    {
      "key": "<buldpack-id>",
      "version": "<buildpack-version>",
      "layers": {
        "<layer-name>": {
          "sha": "<layer-diffID>",
          "data": <layer-metadata>,
          "build": false,
          "launch": false,
          "cache": false
        }
      }
    }
  ],
  "runImage": {
    "topLayer": "<run-image-top-layer-diffID>",
    "reference": "<run-image-reference>"
  },
  "stack": {
    "runImage": {
      "image": "cnbs/sample-stack-run:bionic"
    }
  }
}
```


