# Platform Interface Specification

This document specifies the interface between a lifecycle and a platform.

A platform orchestrates a lifecycle to make buildpack functionality available to end-users such as application developers.

Examples of a platform might include:

1. A local CLI tool that uses buildpacks to create OCI images
1. A plugin for a continuous integration service that uses buildpacks to create OCI images
1. A cloud application platform that uses buildpacks to build source code before deployment

## Table of Contents

<!-- Using https://github.com/yzhang-gh/vscode-markdown to manage toc -->
- [Platform Interface Specification](#platform-interface-specification)
  - [Table of Contents](#table-of-contents)
  - [Platform API Version](#platform-api-version)
  - [Terminology](#terminology)
      - [CNB Terminology](#cnb-terminology)
      - [Additional Terminology](#additional-terminology)
    - [Build Image](#build-image)
    - [Run Image](#run-image)
    - [Target Data](#target-data)
    - [Compatibility Guarantees](#compatibility-guarantees)
  - [Lifecycle Interface](#lifecycle-interface)
    - [Platform API Compatibility](#platform-api-compatibility)
    - [Operations](#operations)
      - [Build](#build)
      - [Rebase](#rebase)
      - [Launch](#launch)
    - [Usage](#usage)
      - [`analyzer`](#analyzer)
        - [Inputs](#inputs)
        - [Outputs](#outputs)
      - [`detector`](#detector)
        - [Inputs](#inputs-1)
        - [Outputs](#outputs-1)
      - [`restorer`](#restorer)
        - [Inputs](#inputs-2)
        - [Outputs](#outputs-2)
        - [Layer Restoration](#layer-restoration)
      - [`extender` (optional and **experimental**)](#extender-optional-and-experimental)
        - [Inputs](#inputs-3)
        - [Outputs](#outputs-3)
      - [`builder`](#builder)
        - [Inputs](#inputs-4)
        - [Outputs](#outputs-4)
      - [`exporter`](#exporter)
        - [Inputs](#inputs-5)
        - [Outputs](#outputs-5)
      - [`creator`](#creator)
        - [Inputs](#inputs-6)
        - [Outputs](#outputs-6)
      - [`rebaser`](#rebaser)
        - [Inputs](#inputs-7)
        - [Outputs](#outputs-7)
      - [`launcher`](#launcher)
        - [Inputs](#inputs-8)
        - [Execution](#execution)
        - [Outputs](#outputs-8)
    - [Run Image Resolution](#run-image-resolution)
    - [Registry Authentication](#registry-authentication)
    - [Experimental Features](#experimental-features)
  - [Buildpacks](#buildpacks)
    - [Buildpacks Directory Layout](#buildpacks-directory-layout)
  - [Image Extensions](#image-extensions)
    - [Image Extensions Directory Layout](#image-extensions-directory-layout)
  - [Security Considerations](#security-considerations)
  - [Additional Guidance](#additional-guidance)
    - [Environment](#environment)
      - [Buildpack Environment](#buildpack-environment)
        - [Base Image-Provided Variables](#base-image-provided-variables)
        - [POSIX Path Variables](#posix-path-variables)
        - [User-Provided Variables](#user-provided-variables)
        - [Operator-Defined Variables](#operator-defined-variables)
      - [Launch Environment](#launch-environment)
    - [Caching](#caching)
    - [Build Reproducibility](#build-reproducibility)
  - [Data Format](#data-format)
    - [Files](#files)
      - [`analyzed.toml` (TOML)](#analyzedtoml-toml)
      - [`group.toml` (TOML)](#grouptoml-toml)
      - [`metadata.toml` (TOML)](#metadatatoml-toml)
      - [`order.toml` (TOML)](#ordertoml-toml)
      - [`plan.toml` (TOML)](#plantoml-toml)
      - [`project-metadata.toml` (TOML)](#project-metadatatoml-toml)
      - [`report.toml` (TOML)](#reporttoml-toml)
      - [`run.toml` (TOML)](#runtoml-toml)
    - [Labels](#labels)
      - [`io.buildpacks.build.metadata` (JSON)](#iobuildpacksbuildmetadata-json)
      - [`io.buildpacks.lifecycle.metadata` (JSON)](#iobuildpackslifecyclemetadata-json)
      - [`io.buildpacks.project.metadata` (JSON)](#iobuildpacksprojectmetadata-json)
  - [Deprecations](#deprecations)
    - [`io.buildpacks.stack.*` Labels](#iobuildpacksstack-labels)
    - [`io.buildpacks.lifecycle.metadata` (JSON) `stack` Key](#iobuildpackslifecyclemetadata-json-stack-key)

## Platform API Version

This document specifies Platform API version `0.12`.

Platform API versions:
 - MUST be in form `<major>.<minor>` or `<major>`, where `<major>` is equivalent to `<major>.0`
 - When `<major>` is greater than `0` increments to `<minor>` SHALL exclusively indicate additive changes

## Terminology

#### CNB Terminology

A **buildpack** refers to software compliant with the [Buildpack Interface Specification](buildpack.md).

A **base image** is an OCI image containing the base, or initial set of layers, for other images.

A **build image** is an OCI image that serves as the base image for the **build environment**.

A **run image** is an OCI image that serves as the base image for the **app image**.

The **build environment** refers to the containerized environment in which the lifecycle executes buildpacks.

An **app image** refers to an OCI image generated by the lifecycle by extending the **run image** with any or all of the following: **app layers**, **launch layers**, **launcher layers**, image configuration.

A **launch layer** refers to a layer in the app image created from a  `<layers>/<layer>` directory as specified in the [Buildpack Interface Specification](buildpack.md).

An **app layer** refers to a layer in the app image created from the `<app>` directory as specified in the [Buildpack Interface Specification](buildpack.md).

A **run image layer** refers to a layer in the **app image** originating from the **run image**.

A **launcher layer** refers to a layer in the app OCI image containing the **launcher** itself and/or launcher configuration.

The **launcher** refers to a lifecycle executable packaged in the **app image** for the purpose of executing processes at runtime.

An **image extension** refers to software compliant with the [Image Extension Interface Specification](image_extension.md). Image extensions participate in detection and execute before the buildpack build process.

A **stack** (deprecated, see [deprecations](#deprecations)) is a contract, implemented by a **build image** and **run image**, that guarantees properties of the **build environment** and **app image**.

#### Additional Terminology

An **image reference** refers to either a **tag reference** or **digest reference**.

A **tag reference** refers to an identifier of form `<registry>/<repo>:<tag>` which locates an image manifest in an [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/master/spec.md) compliant registry.

A **digest reference**  refers to a [content addressable](https://en.wikipedia.org/wiki/Content-addressable_storage) identifier of form `<registry>/<repo>@<digest>` which locates an image manifest in an [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/master/spec.md) compliant registry.

The following is a non-exhaustive list of terms defined in the [OCI Image Format Specification](https://github.com/opencontainers/image-spec) used throughout this document:
* **image manifest** provides an **image config** and a set of layers for a single container image for a specific architecture and operating system.
* **image config** - https://github.com/opencontainers/image-spec/blob/master/config.md#oci-image-configuration
* **imageID** - https://github.com/opencontainers/image-spec/blob/master/config.md#imageid
* **diffID** - https://github.com/opencontainers/image-spec/blob/master/config.md#layer-diffid
* **OCI Image Layout** format is the [directory structure](https://github.com/opencontainers/image-spec/blob/main/image-layout.md) for OCI content-addressable blobs and [location-addressable](https://en.wikipedia.org/wiki/Content-addressable_storage#Content-addressed_vs._location-addressed) references.

The following is a non-exhaustive list of terms defined in the [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/main/spec.md) used throughout this document:

* **registry** - https://github.com/opencontainers/distribution-spec/blob/main/spec.md#definitions

### Build Image

A typical build image might determine:
* The OS distro in the build environment.
* OS packages installed in the build environment.
* Trusted CA certificates in the build environment.
* The default user in the build environment.

The platform MUST ensure that:

- The image config's `User` field is set to a non-root user with a writable home directory.
- The image config's `Env` field has the environment variable `CNB_USER_ID` set to the user [†](README.md#operating-system-conventions)UID/[‡](README.md#operating-system-conventions)SID of the user specified in the `User` field.
- The image config's `Env` field has the environment variable `CNB_GROUP_ID` set to the primary group [†](README.md#operating-system-conventions)GID/[‡](README.md#operating-system-conventions)SID of the user specified in the `User` field.
- The image config's `Env` field has the environment variable `PATH` set to a valid set of paths or explicitly set to empty (`PATH=`).

The platform SHOULD ensure that:

- The image config's `Label` field has the label `io.buildpacks.base.maintainer` set to the name of the image maintainer.
- The image config's `Label` field has the label `io.buildpacks.base.homepage` set to the homepage of the image.
- The image config's `Label` field has the label `io.buildpacks.base.released` set to the release date of the image.
- The image config's `Label` field has the label `io.buildpacks.base.description` set to the description of the image.
- The image config's `Label` field has the label `io.buildpacks.base.metadata` set to additional metadata related to the image.

### Run Image

A typical run image might determine:
* The OS distro or distroless OS in the launch environment.
* OS packages installed in the launch environment.
* Trusted CA certificates in the launch environment.
* The default user in the run environment.

The platform MUST ensure that:

- The image config's `Env` field has the environment variable `PATH` set to a valid set of paths or explicitly set to empty (`PATH=`).

The platform SHOULD ensure that:

- The image config's `User` field is set to a user with a **DIFFERENT** user [†](README.md#operating-system-conventions)UID/[‡](README.md#operating-system-conventions)SID as the build image.
- The image config's `Label` field has the label `io.buildpacks.base.maintainer` set to the name of the image maintainer.
- The image config's `Label` field has the label `io.buildpacks.base.homepage` set to the homepage of the image.
- The image config's `Label` field has the label `io.buildpacks.base.released` set to the release date of the image.
- The image config's `Label` field has the label `io.buildpacks.base.description` set to the description of the image.
- The image config's `Label` field has the label `io.buildpacks.base.metadata` set to additional metadata related to the image.
- The image config's `Label` field has the label `io.buildpacks.rebasable` set to `true` to indicate that new run image versions maintain [ABI-compatibility](https://en.wikipedia.org/wiki/Application_binary_interface) with previous versions (see [Compatibility Guarantees](#compatibility-guarantees)).

### Target Data

For run images, the platform SHOULD ensure that:

- The image config's `Label` field has the label `io.buildpacks.base.id` set to the target ID of the run image.

**The target ID:**
- MUST only contain numbers, letters, and the characters `.`, `/`, and `-`.
- MUST NOT be identical to any other target ID when using a case-insensitive comparison.
- SHOULD use reverse domain notation to avoid name collisions - i.e. buildpacks.io will be `io.buildpacks`.

For both build images and run images, the platform MUST ensure that:

- The image config's `os` and `architecture` fields are set to valid identifiers as defined in the [OCI Image Specification](https://github.com/opencontainers/image-spec/blob/main/config.md).
- The build image config and the run image config both specify the same `os`, `architecture`, `variant` (if specified), `io.buildpacks.base.distro.name` (if specified), and `io.buildpacks.base.distro.version` (if specified).

The platform SHOULD ensure that:

- The image config's `variant` field is set to a valid identifier as defined in the [OCI Image Specification](https://github.com/opencontainers/image-spec/blob/main/config.md).
- The image config's `Label` field has the label `io.buildpacks.base.distro.name` set to the OS distribution and the label `io.buildpacks.base.distro.version` set to the OS distribution version.
  - For Linux-based images, each label should contain the values specified in `/etc/os-release` (`$ID` and `$VERSION_ID`), as the `os.version` field in an image config may contain combined distribution and version information.
  - For Windows-based images, `io.buildpacks.base.distro.name` should be empty; `io.buildpacks.base.distro.version` should contain the value of `os.version` in the image config (e.g., `10.0.14393.1066`).

### Compatibility Guarantees

- Base image authors SHOULD ensure that new build image versions maintain [ABI-compatibility](https://en.wikipedia.org/wiki/Application_binary_interface) with previous versions, although violating this requirement will not change the behavior of previously built images containing app and launch layers.
- Base image authors MUST ensure that new run image versions maintain [ABI-compatibility](https://en.wikipedia.org/wiki/Application_binary_interface) with previous versions.
- Base image authors MUST ensure that app and launch layers do not change behavior when the run image layers are upgraded to newer versions, unless those behavior changes are intended to fix security vulnerabilities.

## Lifecycle Interface

### Platform API Compatibility

The platform SHOULD set `CNB_PLATFORM_API=<platform API version>` in the lifecycle's execution environment.

If `CNB_PLATFORM_API` is set in the lifecycle's execution environment, the lifecycle:
  - MUST either conform to the matching version of this specification or
  - MUST fail if it does not support `<platform API version>`

### Operations

#### Build

A single app image build* consists of the following phases:

1. Analysis
2. Detection
3. Cache Restoration
4. (Optional and Experimental) Base Image Extension
5. Build*
6. Export

A platform MUST execute these phases either by invoking the following phase-specific lifecycle binaries in order:
1. `/cnb/lifecycle/analyzer`
2. `/cnb/lifecycle/detector`
3. `/cnb/lifecycle/restorer`
4. `/cnb/lifecycle/extender` (Optional and [Experimental](#experimental-features)) 
5. `/cnb/lifecycle/builder`
6. `/cnb/lifecycle/exporter`

or by executing `/cnb/lifecycle/creator`**.

> \* **Build** is an overloaded term that refers to both a single phase and the operation comprised of the above phases.
> The meaning of any particular instance of the word **build** must be assessed in context.

> **Does not perform image extension.

#### Rebase

When an updated run image is available, an updated app image SHOULD be generated from the existing app image config by replacing the run image layers in the existing app image with the layers from the new run image.
This is referred to as rebasing the app, launch, and launcher layers onto the new run image layers.
When layers are rebased, any app image metadata referencing to the original run image MUST be updated to reference to the new run image.
This entire operation is referred to as rebasing the app image.

Rebasing allows for fast runtime OS-level dependency updates for app images without requiring a rebuild. A rebase requires minimal data transfer when the app and run images are co-located on an OCI registry that supports [Cross Repository Blob Mounts](https://docs.docker.com/registry/spec/api/#cross-repository-blob-mount).

To rebase an app image a platform MUST execute the `/cnb/lifecycle/rebaser` or perform an equivalent operation.

If an SBOM is available, platforms MAY:
- Warn when a rebase operation would add OS packages.
- Fail if a rebase operation would remove OS packages.

#### Launch

`/cnb/lifecycle/launcher` is responsible for launching user-provided and buildpack-provided processes in the correct execution environment.
`/cnb/lifecycle/launcher`, or a symlink to it (see [exporter outputs](#outputs-4)), SHALL be the `ENTRYPOINT` for all app images.

### Usage

All lifecycle phases:

- MUST read `CNB_PLATFORM_API` from the execution environment and evaluate compatibility before attempting to parse other inputs (see [Platform API Compatibility](#platform-api-compatibility))
- MUST give command line inputs precedence over other inputs

#### `analyzer`

Usage:
```
/cnb/lifecycle/analyzer \
  [-analyzed <analyzed>] \
  [-cache-image <cache-image>] \
  [-daemon] \ # sets <daemon>
  [-gid <gid>] \
  [-launch-cache <launch-cache>] \
  [-layers <layers>] \
  [-layout] \ # sets <layout>
  [-layout-dir] \ # sets <layout-dir>
  [-log-level <log-level>] \
  [-previous-image <previous-image> ] \
  [-run <run> ] \
  [-run-image <run-image> ] \
  [-skip-layers <skip-layers> ] \
  [-tag <tag>...] \
  [-uid <uid>] \
  <image>
```

##### Inputs

| Input              | Environment Variable   | Default Value            | Description                                                                                                           |
|--------------------|------------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------|
| `<analyzed>`       | `CNB_ANALYZED_PATH`    | `<layers>/analyzed.toml` | Path to output analysis metadata (see [`analyzed.toml`](#analyzedtoml-toml)                                           |
| `<cache-image>`    | `CNB_CACHE_IMAGE`      |                          | Reference to a cache image in an OCI registry                                                                         |
| `<daemon>`         | `CNB_USE_DAEMON`       | `false`                  | Analyze image from docker daemon                                                                                      |
| `<gid>`            | `CNB_GROUP_ID`         |                          | Primary GID of the build image `User`                                                                                 |
| `<layers>`         | `CNB_LAYERS_DIR`       | `/layers`                | Path to layers directory                                                                                              |
| `<layout>`         | `CNB_USE_LAYOUT`       | false                    | (**[experimental](#experimental-features)**) Analyze image from disk in OCI layout format                             |
| `<layout-dir>`     | `CNB_LAYOUT_DIR`       |                          | (**[experimental](#experimental-features)**) Path to a root directory where the images are saved in OCI layout format |
| `<image>`          |                        |                          | Tag reference to which the app image will be written                                                                  |
| `<launch-cache>`   | `CNB_LAUNCH_CACHE_DIR` |                          | Path to a cache directory containing launch layers                                                                    |
| `<log-level>`      | `CNB_LOG_LEVEL`        | `info`                   | Log Level                                                                                                             |
| `<previous-image>` | `CNB_PREVIOUS_IMAGE`   | `<image>`                | Image reference to be analyzed (usually the result of the previous build)                                             |
| `<run>`            | `CNB_RUN_PATH`         | `/cnb/run.toml`          | Path to run file (see [`run.toml`](#runtoml-toml))                                                                    |
| `<run-image>`      | `CNB_RUN_IMAGE`        | resolved from `<run>`    | Run image reference                                                                                                   |
| `<skip-layers>`    | `CNB_SKIP_LAYERS`      | `false`                  | Do not restore SBOM layer from previous image                                                                         |
| `<tag>...`         |                        |                          | Additional tag to apply to exported image                                                                             |
| `<uid>`            | `CNB_USER_ID`          |                          | UID of the build image `User`                                                                                         |

-`<image>` MUST be a valid image reference
- **If** the platform provides one or more `<tag>` inputs, each `<tag>` MUST be a valid image reference.
- **If** `<daemon>` is `false` and the platform provides one or more `<tag>` inputs, each `<tag>` MUST refer to the same registry as `<image>`.
- **If** `<daemon>` is `false`, `<previous-image>`, if provided,  MUST be a valid image reference.
- **If** `<daemon>` is `true`, `<previous-image>`, if provided, MUST be either a valid image reference or an imageID.
- **If** `<skip-layers>` is `true` the lifecycle MUST NOT restore the SBOM layer (if any) from the previous image.
- **If** `<run-image>` is not provided by the platform the lifecycle MUST [resolve](#run-image-resolution) the run image from the contents of `run` or fail if `run` does not contain a valid run image.
- The lifecycle MUST accept valid references to non-existent `<previous-image>`, `<cache-image>`, and `<image>` without error.
- The lifecycle MUST ensure registry write access to `<image>`, `<cache-image>` and any provided `<tag>`s.
- The lifecycle MUST ensure registry read access to `<previous-image>`, `<cache-image>`, and `<run-image>`.
- The lifecycle MUST write [analysis metadata](#analyzedtoml-toml) to `<analyzed>`, where:
  - `image` MUST describe the `<previous-image>`, if accessible
  - `run-image` MUST describe the `<run-image>`
- **If** `<layout>` is `true`, `<layout-dir>` MUST be provided and the lifecycle MUST [resolve](#map-an-image-reference-to-a-path-in-the-layout-directory) `<run-image>` and `<previous-image>` following the rules to convert the reference to a path

##### Outputs

| Output             | Description
|--------------------|----------------------------------------------
| [exit status]      | (see Exit Code table below for values)
| `/dev/stdout`      | Logs (info)
| `/dev/stderr`      | Logs (warnings, errors)
| `<analyzed>`       | Analysis metadata (see [`analyzed.toml`](#analyzedtoml-toml)

| Exit Code       | Result|
|-----------------|-------|
| `0`             | Success
| `11`            | Platform API incompatibility error
| `12`            | Buildpack API incompatibility error
| `1-10`, `13-19` | Generic lifecycle errors
| `30-39`         | Analysis-specific lifecycle errors

#### `detector`

The platform MUST execute `detector` in the **build environment**

Usage:
```
/cnb/lifecycle/detector \
  [-analyzed <analyzed>] \
  [-app <app>] \
  [-buildpacks <buildpacks>] \
  [-build-config <build-config>] \
  [-extensions <extensions>] \
  [-generated <generated>] \
  [-group <group>] \
  [-layers <layers>] \
  [-log-level <log-level>] \
  [-order <order>] \
  [-plan <plan>] \
  [-platform <platform>] \
  [-run <run> ]
```

##### Inputs

| Input            | Environment Variable   | Default Value                                          | Description                                                                                                                                                  |
|------------------|------------------------|--------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `<analyzed>`     | `CNB_ANALYZED_PATH`    | `<layers>/analyzed.toml`                               | Path to output analysis metadata (see [`analyzed.toml`](#analyzedtoml-toml)                                                                                  |
| `<app>`          | `CNB_APP_DIR`          | `/workspace`                                           | Path to application directory                                                                                                                                |
| `<build-config>` | `CNB_BUILD_CONFIG_DIR` | `/cnb/build-config`                                    | Path to build config directory                                                                                                                               |
| `<buildpacks>`   | `CNB_BUILDPACKS_DIR`   | `/cnb/buildpacks`                                      | Path to buildpacks directory (see [Buildpacks Directory Layout](#buildpacks-directory-layout))                                                               |
| `<extensions>`^  | `CNB_EXTENSIONS_DIR`   | `/cnb/extensions`                                      | (**[experimental](#experimental-features)**) Path to image extensions directory (see [Image Extensions Directory Layout](#image-extensions-directory-layout) |
| `<generated>`^   | `CNB_GENERATED_DIR`    | `<layers>/generated`                                   | (**[experimental](#experimental-features)**) Path to output directory for generated Dockerfiles                                                              |
| `<group>`        | `CNB_GROUP_PATH`       | `<layers>/group.toml`                                  | Path to output group definition                                                                                                                              |
| `<layers>`       | `CNB_LAYERS_DIR`       | `/layers`                                              | Path to layers directory                                                                                                                                     |
| `<log-level>`    | `CNB_LOG_LEVEL`        | `info`                                                 | Log Level                                                                                                                                                    |
| `<order>`        | `CNB_ORDER_PATH`       | `<layers>/order.toml` if present, or `/cnb/order.toml` | Path resolution for order definition (see [`order.toml`](#ordertoml-toml))                                                                                   |
| `<plan>`         | `CNB_PLAN_PATH`        | `<layers>/plan.toml`                                   | Path to output resolved build plan                                                                                                                           |
| `<platform>`     | `CNB_PLATFORM_DIR`     | `/platform`                                            | Path to platform directory                                                                                                                                   |
| `<run>`^         | `CNB_RUN_PATH`         | `/cnb/run.toml`                                        | Path to run file (see [`run.toml`](#runtoml-toml))                                                                                                           |

> ^Only needed when using image extensions

##### Outputs

| Output                                                   | Description                                                                                   |
|----------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| [exit status]                                            | (see Exit Code table below for values)                                                        |
| `/dev/stdout`                                            | Logs (info)                                                                                   |
| `/dev/stderr`                                            | Logs (warnings, errors)                                                                       |
| `<group>`                                                | Detected buildpack group  (see [`group.toml`](#grouptoml-toml))                               |
| `<plan>`                                                 | Resolved Build Plan (see [`plan.toml`](#plantoml-toml))                                       |
| `<analyzed>`                                             | Updated to include the run image obtained from applying generated Dockerfiles                 |
| `<generated>/run/<image extension ID>/Dockerfile`        | Generated Dockerfiles (see [Image Extension Specfication](image-extension.md))                |
| `<generated>/build/<image extension ID>/Dockerfile`      | Generated Dockerfiles (see [Image Extension Specfication](image-extension.md))                |
| `<generated>/build/<image extension ID>/<extend-config>` | Configuration for the `extend` phase (see [Image Extension Specfication](image-extension.md)) |

| Exit Code       | Result                                                                            |
|-----------------|-----------------------------------------------------------------------------------|
| `0`             | Success                                                                           |
| `11`            | Platform API incompatibility error                                                |
| `12`            | Buildpack API incompatibility error                                               |
| `1-10`, `13-19` | Generic lifecycle errors                                                          |
| `20`            | All buildpacks groups have failed to detect w/o error                             |
| `21`            | All buildpack groups have failed to detect and at least one buildpack has errored |
| `22-29`         | Detection-specific lifecycle errors                                               |
| `91`            | Extension generate error                                                          |
| `92-99`         | Generation-specific lifecycle errors                                              |

The lifecycle:
- SHALL detect a single group from `<order>` and write it to `<group>` using the [detection process](buildpack.md#phase-1-detection) outlined in the Buildpack Interface Specification
- SHALL write the resolved build plan from the detected group to `<plan>`
- SHALL provide `run-image.target` data in `<analyzed>` to buildpacks according to the process outlined in the [Buildpack Interface Specification](buildpack.md).

When image extensions are present in the order (optional and **[experimental](#experimental-features)**), the lifecycle:
- SHALL execute all image extensions in the order defined in `<group>` according to the process outlined in the [Buildpack Interface Specification](buildpack.md).
- SHALL filter the build plan with dependencies provided by image extensions.
- SHALL copy any generated run.Dockerfiles to `<generated>/run/<image extension ID>/Dockerfile`.
- SHALL copy any generated build.Dockerfiles to `<generated>/build/<image extension ID>/Dockerfile`.
- SHALL copy any generated `<extend-config>` files to `<generated>/build/<image extension ID>/<extend-config>`.
- SHALL replace `run-image` in `<analyzed>` with the selected run image. To select the run image, the lifecycle SHALL inspect each `run.Dockerfile` output by image extensions, in the order defined in `<group>`:
  - **If** all `run.Dockerfile`s declare `FROM ${base_image}`, the selected run image SHALL be the original run image in `<analyzed>`, with `extend = true`
  - **Else** the selected run image SHALL be the last image referenced in the `FROM` statement of the last `run.Dockerfile` not to declare `FROM ${base_image}`
    - `run-image.image` SHALL be the name of the selected run image
    - `run-image.reference` and `run-image.target` SHALL be cleared (as they may no longer be accurate)
    - All preceding `run.Dockerfile`s SHALL be copied to `<generated>` with suffix `.ignore`
    - **If** there are no `run.Dockerfile`s following the Dockerfile with the selected run image:
      - `run-image.extend` SHALL be `false`
    - **Else**
      - `run-image.extend` SHALL be `true`
- SHALL warn if the selected run image is not found in `<run>`

#### `restorer`

Usage:
```
/cnb/lifecycle/restorer \
  [-analyzed <analyzed>] \
  [-build-image <build-image>] \
  [-cache-dir <cache-dir>] \
  [-cache-image <cache-image>] \
  [-daemon] \ # sets <daemon>
  [-gid <gid>] \
  [-group <group>] \
  [-layers <layers>] \
  [-log-level <log-level>] \
  [-skip-layers <skip-layers>] \
  [-uid <uid>]
```

##### Inputs

| Input            | Environment Variable | Default Value            | Description                                                                                       |
|------------------|----------------------|--------------------------|---------------------------------------------------------------------------------------------------|
| `<analyzed>`     | `CNB_ANALYZED_PATH`  | `<layers>/analyzed.toml` | Path to output analysis metadata (see [`analyzed.toml`](#analyzedtoml-toml)                       |
| `<build-image>`* | `CNB_BUILD_IMAGE`    |                          | Reference to the current build image in an OCI registry (if used `<kaniko-dir>` must be provided) |
| `<cache-dir>`    | `CNB_CACHE_DIR`      |                          | Path to a cache directory                                                                         |
| `<cache-image>`  | `CNB_CACHE_IMAGE`    |                          | Reference to a cache image in an OCI registry                                                     |
| `<daemon>`^      | `CNB_USE_DAEMON`     | `false`                  | Read additional target data for run image from docker daemon                                      |
| `<gid>`          | `CNB_GROUP_ID`       |                          | Primary GID of the build image `User`                                                             |
| `<group>`        | `CNB_GROUP_PATH`     | `<layers>/group.toml`    | Path to group definition (see [`group.toml`](#grouptoml-toml))                                    |
| `<kaniko-dir>`^  |                      |                          | Kaniko directory (must be `/kaniko`)                                                              |
| `<layers>`       | `CNB_LAYERS_DIR`     | `/layers`                | Path to layers directory                                                                          |
| `<log-level>`    | `CNB_LOG_LEVEL`      | `info`                   | Log Level                                                                                         |
| `<skip-layers>`  | `CNB_SKIP_LAYERS`    | `false`                  | Do not perform [layer restoration](#layer-restoration)                                            |
| `<uid>`          | `CNB_USER_ID`        |                          | UID of the build image `User`                                                                     |

> ^ Only needed when using image extensions

> \* Only needed when using image extensions to extend the build image

##### Outputs

| Output                                      | Description                                                                                                                               |
|---------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| [exit status]                               | (see Exit Code table below for values)                                                                                                    |
| `/dev/stdout`                               | Logs (info)                                                                                                                               |
| `/dev/stderr`                               | Logs (warnings, errors)                                                                                                                   |
| `<layers>/<buidpack-id>/store.toml`         | Persistent metadata (see data format in [Buildpack Interface Specification](buildpack.md))                                                |
| `<layers>/<buidpack-id>/<layer>.toml`       | Files containing the layer content metadata of each analyzed layer (see data format in [Buildpack Interface Specification](buildpack.md)) |
| `<layers>/<buidpack-id>/<layer>.sbom.<ext>` | Files containing the Software Bill of Materials for each analyzed layer (see [Buildpack Interface Specification](buildpack.md))           |
| `<layers>/<buidpack-id>/<layer>/*`.         | Restored layer contents                                                                                                                   |
| `<kaniko-dir>/cache`                        | Kaniko cache contents                                                                                                                     |


| Exit Code       | Result                                |
|-----------------|---------------------------------------|
| `0`             | Success                               |
| `11`            | Platform API incompatibility error    |
| `12`            | Buildpack API incompatibility error   |
| `1-10`, `13-19` | Generic lifecycle errors              |
| `40-49`         | Restoration-specific lifecycle errors |

- For each buildpack in `<group>`, if persistent metadata for that buildpack exists in the analysis metadata, lifecycle MUST write a toml representation of the persistent metadata to `<layers>/<buildpack-id>/store.toml`
- **If** `<skip-layers>` is `true` the lifecycle MUST NOT perform layer restoration.
- **Else** the lifecycle MUST perform [layer restoration](#layer-restoration) for any app image layers or cached layers created by any buildpack present in the provided `<group>`.
- When `<build-image>` is provided (optional and **[experimental](#experimental-features)**), the lifecycle:
  - MUST record the digest reference to the provided `<build-image>` in `<analyzed>`
  - MUST copy the OCI manifest and config file for `<build-image>` to `<kaniko-dir>/cache`
- The lifecycle:
  - MUST resolve `run-image.reference` to a digest reference in `<analyzed>` if not present
  - MUST populate `run-image.target` data in `<analyzed>` if not present
  - **If** `<analyzed>` has `run-image.extend = true`, the lifecycle:
    - MUST download from the registry and save in OCI layout format the `run-image` in `<analyzed>` to `<kaniko-dir>/cache`

##### Layer Restoration

lifeycle MUST use the provided `cache-dir` or `cache-image` to retrieve cache contents. The [rules](https://github.com/buildpacks/spec/blob/main/buildpack.md#layer-types) for restoration MUST be followed when determining how and when to store cache layers.

#### `extender` (optional and **[experimental](#experimental-features)**)

If using `extender`, the platform MUST execute `extender` in either or both of: the **build environment**, the **run environment**

Usage:
```
/cnb/lifecycle/extender \
  [-analyzed <analyzed>] \
  [-app <app>] \
  [-buildpacks <buildpacks>] \
  [-extended <extended>] \
  [-generated <generated>] \
  [-gid <gid>] \
  [-group <group>] \
  [-kaniko-cache-ttl <kaniko-cache-ttl>] \
  [-kind <kind>] \
  [-layers <layers>] \
  [-log-level <log-level>] \
  [-plan <plan>] \
  [-platform <platform>]
  [-uid <uid>] \
```

##### Inputs

| Input                | Env                    | Default Value            | Description                                                                                     |
|----------------------|------------------------|--------------------------|-------------------------------------------------------------------------------------------------|
| `<analyzed>`         | `CNB_ANALYZED_PATH`    | `<layers>/analyzed.toml` | Path to analysis metadata (see [`analyzed.toml`](#analyzedtoml-toml)                            |
| `<app>`              | `CNB_APP_DIR`          | `/workspace`             | Path to application directory                                                                   |
| `<build-config>`     | `CNB_BUILD_CONFIG_DIR` | `/cnb/build-config`      | Path to build config directory                                                                  |
| `<buildpacks>`*      | `CNB_BUILDPACKS_DIR`   | `/cnb/buildpacks`        | Path to buildpacks directory (see [Buildpacks Directory Layout](#buildpacks-directory-layout))  |
| `<extended>`**       | `CNB_EXTENDED_DIR`     | `<layers>/extended`      | Path to output directory for extended run image layers                                          |
| `<generated>`        | `CNB_GENERATED_DIR`    | `<layers>/generated`     | (**[experimental](#experimental-features)**) Path to directory containing generated Dockerfiles |
| `<gid>`*             | `CNB_GROUP_ID`         |                          | Primary GID of the build image `User`                                                           |
| `<group>`            | `CNB_GROUP_PATH`       | `<layers>/group.toml`    | Path to group definition (see [`group.toml`](#grouptoml-toml))                                  |
| `<kaniko-cache-ttl>` | `CNB_KANIKO_CACHE_TTL` | 2 weeks                  | Kaniko cache TTL                                                                                |
| `<kaniko-dir>`       |                        |                          | Kaniko directory (must be `/kaniko`)                                                            |
| `<kind>`             | `CNB_EXTEND_KIND`      | `build`                  | Type of image to extend (valid values: `build`, `run`)                                          |
| `<layers>`           | `CNB_LAYERS_DIR`       | `/layers`                | Path to layers directory                                                                        |
| `<log-level>`        | `CNB_LOG_LEVEL`        | `info`                   | Log Level                                                                                       |
| `<plan>`*            | `CNB_PLAN_PATH`        | `<layers>/plan.toml`     | Path to resolved build plan (see [`plan.toml`](#plantoml-toml))                                 |
| `<platform>`         | `CNB_PLATFORM_DIR`     | `/platform`              | Path to platform directory                                                                      |
| `<uid>`*             | `CNB_USER_ID`          |                          | UID of the build image `User`                                                                   |

> \* Only needed when extending the build image

> ** Only needed when extending the run image

##### Outputs

When extending the build image:

- In addition to the outputs enumerated below, outputs produced by `extender` include those produced by `builder` - as the lifecycle will run the `build` phase after extending the build image.
- Platforms MUST skip the `builder` and proceed to the `exporter`.

| Output               | Description                            |
|----------------------|----------------------------------------|
| [exit status]        | (see Exit Code table below for values) |
| `/dev/stdout`        | Logs (info)                            |
| `/dev/stderr`        | Logs (warnings, errors)                |
| `<kaniko-dir>/cache` | Kaniko cache contents                  |

| Exit Code       | Result                              |
|-----------------|-------------------------------------|
| `0`             | Success                             |
| `11`            | Platform API incompatibility error  |
| `12`            | Buildpack API incompatibility error |
| `1-10`, `13-19` | Generic lifecycle errors            |
| `100-109`       | Extension-specific lifecycle errors |

- For each extension in `<group>` in order, if a Dockerfile exists in `<generated>/<kind>/<buildpack-id>`, the lifecycle:
  - SHALL apply the Dockerfile to the environment according to the process outlined in the [Image Extension Specification](image-extension.md).
- The extended image MUST be an extension of:
  - The `build-image` in `<analyzed>` when `<kind>` is `build`, or
  - The `run-image` in `<analyzed>` when `<kind>` is `run`
- When extending the build image, after all `build.Dockefile`s are applied, the lifecycle:
  - SHALL proceed with the `build` phase using the provided `<gid>` and `<uid>`
- When extending the run image, after all `run.Dockefile`s are applied, the lifecycle:
  - **If** any `run.Dockerfile` set the label `io.buildpacks.rebasable` to `false` or left the label unset:
    - SHALL set the label `io.buildpacks.rebasable` to `false` on the extended run image
  - **If** after the final `run.Dockerfile` the run image user is `root`,
    - SHALL fail
  - SHALL copy the manifest and config for the extended run image, along with any new layers, to `<extended>`/run

#### `builder`

The platform MUST execute `builder` in the **build environment**

Usage:
```
/cnb/lifecycle/builder \
  [-analyzed <analyzed>] \
  [-app <app>] \
  [-buildpacks <buildpacks>] \
  [-build-config <build-config>] \
  [-group <group>] \
  [-layers <layers>] \
  [-log-level <log-level>] \
  [-plan <plan>] \
  [-platform <platform>]
```

##### Inputs

| Input            | Env                    | Default Value            | Description                                                                                    |
|------------------|------------------------|--------------------------|------------------------------------------------------------------------------------------------|
| `<analyzed>`     | `CNB_ANALYZED_PATH`    | `<layers>/analyzed.toml` | Path to analysis metadata (see [`analyzed.toml`](#analyzedtoml-toml)                           |
| `<app>`          | `CNB_APP_DIR`          | `/workspace`             | Path to application directory                                                                  |
| `<build-config>` | `CNB_BUILD_CONFIG_DIR` | `/cnb/build-config`      | Path to build config directory                                                                 |
| `<buildpacks>`   | `CNB_BUILDPACKS_DIR`   | `/cnb/buildpacks`        | Path to buildpacks directory (see [Buildpacks Directory Layout](#buildpacks-directory-layout)) |
| `<group>`        | `CNB_GROUP_PATH`       | `<layers>/group.toml`    | Path to group definition (see [`group.toml`](#grouptoml-toml))                                 |
| `<layers>`       | `CNB_LAYERS_DIR`       | `/layers`                | Path to layers directory                                                                       |
| `<log-level>`    | `CNB_LOG_LEVEL`        | `info`                   | Log Level                                                                                      |
| `<plan>`         | `CNB_PLAN_PATH`        | `<layers>/plan.toml`     | Path to resolved build plan (see [`plan.toml`](#plantoml-toml))                                |
| `<platform>`     | `CNB_PLATFORM_DIR`     | `/platform`              | Path to platform directory                                                                     |

##### Outputs

| Output                                     | Description
|--------------------------------------------|----------------------------------------------
| [exit status]                              | (see Exit Code table below for values)
| `/dev/stdout`                              | Logs (info)
| `/dev/stderr`                              | Logs (warnings, errors)
| `<layers>/<buildpack ID>/<layer>`          | Layer contents (see [Buildpack Interface Specfication](buildpack.md)
| `<layers>/<buildpack ID>/<layer>.toml`     | Layer metadata (see [Buildpack Interface Specfication](buildpack.md)
| `<layers>/config/metadata.toml`            | Build metadata (see [`metadata.toml`](#metadatatoml-toml))

| Exit Code | Result|
|-----------|-------|
| `0`       | Success
| `11`      | Platform API incompatibility error
| `12`      | Buildpack API incompatibility error
| `1-10`, `13-19` | Generic lifecycle errors
| `51`     | Buildpack build error
| `50`, `52-59`|  Build-specific lifecycle errors

- The lifecycle SHALL execute all buildpacks in the order defined in `<group>` according to the process outlined in the [Buildpack Interface Specification](buildpack.md).
- SHALL provide `run-image.target` data in `<analyzed>` to buildpacks according to the process outlined in the [Buildpack Interface Specification](buildpack.md).
- The lifecycle SHALL add all invoked buildpacks to`<layers>/config/metadata.toml`.
- The lifecycle SHALL aggregate all `processes` and `slices` returned by buildpacks in `<layers>/config/metadata.toml`.
- The lifecycle SHALL record the buildpack-provided default process type in `<layers>/config/metadata.toml`.
    - The lifecycle SHALL treat `web` processes defined by buildpacks implementing Buildpack API < 0.6 as `default = true`.

#### `exporter`

Usage:
```
/cnb/lifecycle/exporter \
  [-analyzed <analyzed>] \
  [-app <app>] \
  [-cache-dir <cache-dir>] \
  [-cache-image <cache-image>] \
  [-daemon] \ # sets <daemon>
  [-extended <extended>] \
  [-gid <gid>] \
  [-group <group>] \
  [-launch-cache <launch-cache> ] \
  [-launcher <launcher> ] \
  [-launcher-sbom <launcher-sbom> ] \
  [-layers <layers>] \
  [-layout] \ # sets <layout>
  [-layout-dir] \ # sets <layout-dir>
  [-log-level <log-level>] \
  [-process-type <process-type> ] \
  [-project-metadata <project-metadata> ] \
  [-report <report> ] \
  [-run <run>] \
  [-uid <uid> ] \
  <image> [<image>...]
```

##### Inputs

| Input                           | Environment Variable        | Default Value                    | Description                                                                                |
|---------------------------------|-----------------------------|----------------------------------|--------------------------------------------------------------------------------------------|
| `<analyzed>`                    | `CNB_ANALYZED_PATH`         | `<layers>/analyzed.toml`         | Path to analysis metadata (see [`analyzed.toml`](#analyzedtoml-toml)                       |
| `<app>`                         | `CNB_APP_DIR`               | `/workspace`                     | Path to application directory                                                              |
| `<cache-dir>`                   | `CNB_CACHE_DIR`             |                                  | Path to a cache directory                                                                  |
| `<cache-image>`                 | `CNB_CACHE_IMAGE`           |                                  | Reference to a cache image in an OCI registry                                              |
| `<daemon>`                      | `CNB_USE_DAEMON`            | `false`                          | Export image to docker daemon                                                              |
| `<extended>`**                  | `CNB_EXTENDED_DIR`          | `<layers>/extended`              | Path to directory containing extended run image layers                                     |
| `<gid>`                         | `CNB_GROUP_ID`              |                                  | Primary GID of the build image `User`                                                      |
| `<group>`                       | `CNB_GROUP_PATH`            | `<layers>/group.toml`            | Path to group file (see [`group.toml`](#grouptoml-toml))                                   |
| `<image>`                       |                             |                                  | Tag reference to which the app image will be written                                       |
| `<launch-cache>`                | `CNB_LAUNCH_CACHE_DIR`      |                                  | Path to a cache directory containing launch layers                                         |
| `<launcher-sbom>`               |                             | `/cnb/lifecycle`                 | Path to directory containing SBOM files describing the `launcher` executable               |
| `<launcher>`                    |                             | `/cnb/lifecycle/launcher`        | Path to the `launcher` executable                                                          |
| `<layers>/config/metadata.toml` |                             |                                  | Build metadata (see [`metadata.toml`](#metadatatoml-toml)                                  |
| `<layers>`                      | `CNB_LAYERS_DIR`            | `/layers`                        | Path to layer directory                                                                    |
| `<layout>`                      | `CNB_USE_LAYOUT`            | false                            | (**[experimental](#experimental-features)**) Export image to disk in OCI layout format     |
| `<layout-dir>`                  | `CNB_LAYOUT_DIR`            |                                  | (**[experimental](#experimental-features)**) Path to a root directory where the images are saved in OCI layout format |
| `<log-level>`                   | `CNB_LOG_LEVEL`             | `info`                           | Log Level                                                                                  |
| `<process-type>`                | `CNB_PROCESS_TYPE`          |                                  | Default process type to set in the exported image                                          |
| `<project-metadata>`            | `CNB_PROJECT_METADATA_PATH` | `<layers>/project-metadata.toml` | Path to a project metadata file (see [`project-metadata.toml`](#project-metadatatoml-toml) |
| `<report>`                      | `CNB_REPORT_PATH`           | `<layers>/report.toml`           | Path to report (see [`report.toml`](#reporttoml-toml)                                      |
| `<run>`                         | `CNB_RUN_PATH`              | `/cnb/run.toml`                  | Path to run file (see [`run.toml`](#runtoml-toml)                                          |
| `<uid>`                         | `CNB_USER_ID`               |                                  | UID of the build image `User`                                                              |
|                                 | `SOURCE_DATE_EPOCH`         |                                  | Timestamp for `created` time in app image config                                           |

> ** Only needed when extending the run image

- At least one `<image>` must be provided
- Each `<image>` MUST be a valid tag reference
- **If** `<daemon>` is `false` and more than one `<image>` is provided they MUST refer to the same registry
- The `<run-image>` will be read from [`analyzed.toml`](#analyzedtoml-toml)

##### Outputs

| Output             | Description
|--------------------|----------------------------------------------
| `[exit status]`    | Success (0), or error (1+)
| `/dev/stdout`      | Logs (info)
| `/dev/stderr`      | Logs (warnings, errors)
| `<image>`          | Exported app image (see [Buildpack Interface Specfication](buildpack.md)

| Exit Code | Result|
|-----------|-------|
| `0`       | Success
| `11`      | Platform API incompatibility error
| `12`      | Buildpack API incompatibility error
| `1-10`, `13-19` | Generic lifecycle errors
| `60-69`|  Export-specific lifecycle errors

- The lifecycle SHALL write the same app image to each `<image>` tag
- The app image:
    - **If** image extensions were used to extend the `run-image` in [`analyzed.toml`](#analyzedtoml-toml):
      - MUST be based on the image in `<extended>`/run
    - **Else**:
      - MUST be based on the `run-image` in [`analyzed.toml`](#analyzedtoml-toml)
    - All base image layers and any extension layers SHALL be preserved
    - All base image config values SHALL be preserved unless this conflicts with another requirement
    - MUST contain all buildpack-provided launch layers as determined by the [Buildpack Interface Specfication](buildpack.md)
    - MUST contain a layer containing all Software Bill of Materials (SBOM) files for `launch` as determined by the [Buildpack Interface Specfication](buildpack.md) if they are present
      - `<layers>/sbom/launch/<buildpack-id>/sbom.<ext>` MUST contain the buildpack-provided `launch` SBOM
      - `<layers>/sbom/launch/<buildpack-id>/<layer-id>/sbom.<ext>` MUST contain the buildpack-provided layer SBOM if `<layer-id>` is a `launch` layer
      - `<layers>/sbom/launch/sbom.legacy.json` MAY contain the legacy buildpack-provided non-standard Bill of Materials for `launch` (where [supported](buildpack.md))
      - `<layers>/sbom/launch/buildpacksio_lifecycle/launcher/sbom.<ext>` MUST contain the CNB-provided launcher SBOM if present in the `/cnb/lifecycle` directory
    - MUST contain one or more app layers as determined by the [Buildpack Interface Specfication](buildpack.md)
    - MUST contain one or more launcher layers that include:
        - A file with the contents of the `<launcher>` file at path `/cnb/lifecycle/launcher`
        - One symlink per buildpack-provided process type with name `/cnb/process/<type>` and target `/cnb/lifecycle/launcher`
    - MUST contain a layer that includes `<layers>/config/metadata.toml`
    - **If** `<process-type>` matches a buildpack-provided process:
      - MUST have `ENTRYPOINT=/cnb/process/<process-type>`
    - **Else if** `<process-type>` is provided and does not match a buildpack-provided process:
      - MUST fail
    - **Else if** there is a buildpack-provided default process type in `<layers>/config/metadata.toml`:
      - MUST have `ENTRYPOINT=/cnb/process/<buildpack-default-process-type>`
    - **Else**:
      - MUST have `ENTRYPOINT` set to `/cnb/lifecycle/launcher`
    - MUST contain the following `Env` entries
      - `CNB_LAYERS_DIR=<layers>`
      - `CNB_APP_DIR=<app>`
      - `PATH=/cnb/process:$PATH` where `$PATH` is the value of `$PATH` on the run-image.
    - MUST have the working directory set to the value of `<app>`.
    - MUST contain the following labels
        - `io.buildpacks.lifecycle.metadata`: see [lifecycle metadata label](#iobuildpackslifecyclemetadata-json)
        - `io.buildpacks.project.metadata`: the value of which SHALL be the json representation of `<project-metadata>`
        - `io.buildpacks.build.metadata`: see [build metadata](#iobuildpacksbuildmetadata-json)
        - **If** image extensions were used to extend the run image and `<extended>/run/<image config>` has the label `io.buildpacks.rebasable` set to `true`:
          - `io.buildpacks.rebasable` SHALL be `true`
        - **Else**
          - `io.buildpacks.rebasable` SHALL be `false`
- To ensure [build reproducibility](#build-reproducibility), the lifecycle:
    - SHOULD set the modification time of all files in newly created layers to a constant value
    - SHOULD set the `created` time in image config to `SOURCE_DATE_EPOCH`, or to a constant value if not defined

- The lifecycle SHALL write a [report](#reporttoml-toml) to `<report>` describing the exported app image

- The `<layers>` directory:
  - MUST include all Software Bill of Materials (SBOM) files for `build` as determined by the [Buildpack Interface Specfication](buildpack.md) if they are present
    - `<layers>/sbom/build/<buildpack-id>/sbom.<ext>` MUST contain the buildpack-provided `build` SBOM
    - `<layers>/sbom/build/<buildpack-id>/<layer-id>/sbom.<ext>` MUST contain the buildpack-provided layer SBOM if `<layer-id>` is not a `launch` layer.
    - `<layers>/sbom/build/sbom.legacy.json` MAY contain the legacy buildpack-provided non-standard Bill of Materials for `build` (where [supported](buildpack.md))
    - `<layers>/sbom/build/buildpacksio_lifecycle/sbom.<ext>` MUST contain the CNB-provided lifecycle SBOM if present in the `/cnb/lifecycle` directory
- *If* a cache is provided the lifecycle:
   - SHALL write the contents of all cached layers and any provided layer-associated SBOM files to the cache
   - SHALL record the diffID and layer content metadata of all cached layers in the cache

- **If** `<layout>` is `true` the lifecycle:
   - SHALL write the app image on disk following the [rules](#map-an-image-reference-to-a-path-in-the-layout-directory) to convert the reference to a path

#### `creator`

The platform MUST execute `creator` in the **build environment**

Usage:
```
/cnb/lifecycle/creator \
  [-app <app>] \
  [-buildpacks <buildpacks>] \
  [-cache-dir <cache-dir>] \
  [-cache-image <cache-image>] \
  [-daemon] \ # sets <daemon>
  [-gid <gid>] \
  [-launch-cache <launch-cache> ] \
  [-launcher <launcher> ] \
  [-layers <layers>] \
  [-layout] \ # sets <layout>
  [-layout-dir] \ # sets <layout-dir>
  [-log-level <log-level>] \
  [-order <order>] \
  [-platform <platform>] \
  [-previous-image <previous-image> ] \
  [-process-type <process-type> ] \
  [-project-metadata <project-metadata> ] \
  [-report <report> ] \
  [-run <run>] \
  [-run-image <run-image>] \
  [-skip-restore <skip-restore>] \
  [-tag <tag>...] \
  [-uid <uid> ] \
   <image>
```

##### Inputs

Running `creator` SHALL be equivalent to running `detector`, `analyzer`, `restorer`, `builder` and `exporter` in order with identical inputs where they are accepted, with the following exceptions.

| Input             | Environment Variable| Default Value| Description
|-------------------|---------------------|--------------|----------------------
| `<previous-image>`| `CNB_PREVIOUS_IMAGE`| `<image>`    | Image reference to be analyzed (usually the result of the previous build)
| `<skip-restore>`  | `CNB_SKIP_RESTORE`  | `false`      | Prevent buildpacks from reusing layers from previous builds, by skipping the restoration of any data to each buildpack's layers directory, with the exception of `store.toml`.
| `<tag>...`        |                     |              | Additional tag to apply to exported image

- **If** `<skip-restore>` is `true` the `creator` SHALL skip the restoration of any data to each buildpack's layers directory, with the exception of `store.toml`.
- **If** the platform provides one or more `<tag>` inputs they SHALL be treated as additional `<image>` inputs to the `exporter`

##### Outputs

Outputs produced by `creator` are identical to those produced by `exporter`, with the following additional expanded set of error codes.

| Exit Code | Result|
|-----------|-------|
| `0`       | Success
| `11`      | Platform API incompatibility error
| `12`      | Buildpack API incompatibility error
| `1-10`, `13-19` | Generic lifecycle errors
| `20-29`|  Detection-specific lifecycle errors
| `30-39`|  Analysis-specific lifecycle errors
| `40-49`|  Restoration-specific lifecycle errors
| `50-59`|  Build-specific lifecycle errors
| `60-69`|  Export-specific lifecycle errors

#### `rebaser`

Usage:
```
/cnb/lifecycle/rebaser \
  [-daemon] \ # sets <daemon>
  [-force] \
  [-gid <gid>] \
  [-log-level <log-level>] \
  [-previous-image <previous-image>] \
  [-report <report> ] \
  [-run-image <run-image> | -image <run-image> ] \ # -image is Deprecated
  [-uid <uid>] \
  <image> [<image>...]
```

##### Inputs

| Input              | Environment Variable | Default Value          | Description                                           |
|--------------------|----------------------|------------------------|-------------------------------------------------------|
| `<daemon>`         | `CNB_USE_DAEMON`     | `false`                | Export image to docker daemon                         |
| `<force>`          | `CNB_FORCE_REBASE`   | `false`                | Allow unsafe rebase                                   |
| `<gid>`            | `CNB_GROUP_ID`       |                        | Primary GID of the build image `User`                 |
| `<image>`          |                      |                        | App image to rebase                                   |
| `<log-level>`      | `CNB_LOG_LEVEL`      | `info`                 | Log Level                                             |
| `<previous-image>` |                      | derived from `<image>` | Previous image reference                              |
| `<report>`         | `CNB_REPORT_PATH`    | `<layers>/report.toml` | Path to report (see [`report.toml`](#reporttoml-toml) |
| `<run-image>`      | `CNB_RUN_IMAGE`      | derived from `<image>` | Run image reference                                   |
| `<uid>`            | `CNB_USER_ID`        |                        | UID of the build image `User`                         |

- At least one `<image>` must be provided
- **If** `<image>` has the label `io.buildpacks.rebasable` set to `false`, the lifecycle SHALL fail unless `<force>` is `true`
- Each `<image>` MUST be a valid tag reference
- **If** `<daemon>` is `false` and more than one `<image>` is provided, the images MUST refer to the same registry.
- **If** `<previous-image>` is provided by the platform, the value will be used as the app image to rebase. `<previous-image>` must NOT be modified unless specified again in `<image>`.
- **Else** `<previous-image>` value will be derived from the first `<image>`.
- **If** `<run-image>` is not provided by the platform, the value will be [resolved](#run-image-resolution) from the contents of the `runImage` key in the `io.buildpacks.lifecycle.metdata` label on `<image>`, or `stack.runImage` if not found (for compatibility with older platforms; see [deprecations](#deprecations)).

##### Outputs

| Output             | Description
|--------------------|----------------------------------------------
| [exit status]      | (see Exit Code table below for values)
| `/dev/stdout`      | Logs (info)
| `/dev/stderr`      | Logs (warnings, errors)
| `<image>`          | Rebased app image (see [Buildpack Interface Specfication](buildpack.md))

| Exit Code | Result|
|-----------|-------|
| `0`       | Success
| `11`      | Platform API incompatibility error
| `12`      | Buildpack API incompatibility error
| `1-10`, `13-19` | Generic lifecycle errors
| `70-79`|  Rebase-specific lifecycle errors

- The lifecycle SHALL write the same app image to each `<image>` tag
- The rebased app image SHALL be identical to `<image>`, with the following modifications:
    - Run image layers SHALL be defined as Layers in `<image>` up to and including the layer with diff ID matching the value of `run-image.top-layer` from the `io.buildpacks.lifecycle.metadata` label
    - Run image layers SHALL be replaced with the layers from the new `<run-image>`
    - The value of `io.buildpacks.lifecycle.metadata` SHALL be modified as follows
      - `run-image.reference` SHALL uniquely identify `<run-image>`
      - `run-image.top-layer` SHALL be set to the uncompressed digest of the top layer in `<run-image>`
    - The value of `io.buildpacks.base.*` labels and `io.buildpacks.stack.*` labels (if present) SHALL be modified to that of the new `run-image`
- **If** `<force>` is `true`, the following [target data](#target-data) values in the output `<image>` config MUST be derived from the new `<run-image>`:
  - `os`
  - `architecture`
  - `variant` (if specified)
  - `io.buildpacks.base.distro.name` (if specified)
  - `io.buildpacks.base.distro.version` (if specified)
- **Else** the target data above MUST match the old run image if `<force>` is `false`
- **If** `<force>` is `true` and the provided `<run-image>` is not found in `runImage.image` or `runImage.mirrors`:
  - `run-image.image` SHALL be the provided `<run-image>`
  - `run-image.mirrors` SHALL be omitted
- **Else if** `<force> is `false`, the provided `<run-image>` MUST be found in `runImage.image` or `runImage.mirrors`
- To ensure [build reproducibility](#build-reproducibility), the lifecycle:
    - SHOULD set the `created` time in image config to a constant
- The lifecycle SHALL write a [report](#reporttoml-toml) to `<report>` describing the rebased app image

#### `launcher`

Usage:
```
/cnb/process/<process-type> [<arg>...]
# OR
/cnb/lifecycle/launcher [--] [<cmd> <arg>...]
```
##### Inputs

| Input                              | Environment Variable | Default Value | Description                                               |
|------------------------------------|----------------------|---------------|-----------------------------------------------------------|
| `<app>`                            | `CNB_APP_DIR`        | `/workspace`  | Path to application directory                             |
| `<layers>`                         | `CNB_LAYERS_DIR`     | `/layers`     | Path to layer directory                                   |
| `<process-type>`                   |                      |               | `type` of process to launch                               |
| `<direct>`                         |                      |               | Process execution strategy                                |
| `<cmd>`                            |                      |               | Command to execute                                        |
| `<args>`                           |                      |               | Arguments to command                                      |
| `<layers>/config/metadata.toml`    |                      |               | Build metadata (see [`metadata.toml`](#metadatatoml-toml) |
| `<layers>/<buildpack-id>/<layer>/` |                      |               | Launch Layers                                             |

A command (`<cmd>`), arguments to that command (`<args>`), a working directory (`<working-dir>`), and an execution strategy (`<direct>`) comprise a process definition. Processes MAY be buildpack-defined or user-defined.

The launcher:

- MUST derive the values of `<cmd>`, `<args>`, `<working-dir>`, and `<direct>` as follows:
- **If** the final path element in `$0`, matches the type of any buildpack-provided process type
    - `<process-type>` SHALL be the final path element in `$0`
    - The lifecycle:
        - MUST select the process with type equal to `<process-type>` from `<layers>/config/metadata.toml`
        - MUST set `<working-dir>` to the value defined for the process in `<layers>/config/metadata.toml`, or to `<app>` if not defined
        - **If** the buildpack that provided the process supports default process args
          - `<direct>` SHALL be `true`
          - MUST replace any buildpack provided default `<args>` with any user-provided `<args>`
        - **Else**
          - `<direct>` value defined by buildpack SHALL be honored
          - MUST append any user-provided `<args>` to process arguments
- **Else**
    - **If** `$1` is `--`
        - `<direct>` SHALL be `true`
        - `<cmd>` SHALL be `$2`
        - `<args>` SHALL be `${@3:}`
        - `<working-dir>` SHALL be `<app>`
    - **Else**
        - `<direct>` SHALL be `false`
        - `<cmd>` SHALL be `$1`
        - `<args>` SHALL be `${@2:}`
        - `<working-dir` SHALL be `<app>`

##### Execution

Given the start command and execution strategy,

1. The launcher MUST set all buildpack-provided launch environment variables as described in the [Environment](#environment) section.

1. The launcher MUST
   1. [execute](#execd) each file in each `<layers>/<layer>/exec.d` directory in the launch environment, with working directory `<app>`, and set the [returned variables](#execd-output-toml) in the launch environment before continuing,
      1. Firstly, in order of `/bin/build` execution used to construct the OCI image.
      2. Secondly, in alphabetically ascending order by layer directory name.
      3. Thirdly, in alphabetically ascending order by file name.
   2. [execute](#execd) each file in each `<layers>/<layer>/exec.d/<process>` directory in the launch environment, with working directory `<app>`, and set the [returned variables](#execd-output-toml) in the launch environment before continuing,
      1. Firstly, in order of `/bin/build` execution used to construct the OCI image.
      2. Secondly, in alphabetically ascending order by layer directory name.
      3. Thirdly, in alphabetically ascending order by file name.

1. If using an execution strategy involving a shell, the launcher MUST use a single shell process, with working directory `<app>`, to
   1. source each file in each `<layers>/<layer>/profile.d` directory,
      1. Firstly, in order of `/bin/build` execution used to construct the OCI image.
      2. Secondly, in alphabetically ascending order by layer directory name.
      3. Thirdly, in alphabetically ascending order by file name.
   2. source each file in each `<layers>/<layer>/profile.d/<process>` directory,
      1. Firstly, in order of `/bin/build` execution used to construct the OCI image.
      2. Secondly, in alphabetically ascending order by layer directory name.
      3. Thirdly, in alphabetically ascending order by file name.
   3. source [†](README.md#linux-only)`<app>/.profile` or [‡](README.md#windows-only)`<app>/.profile.bat` if it is present.

1. The launcher MUST set the working directory for the start command to `<working-dir>`, or to `<app>` if `<working-dir>` is not specified.

1. The launcher MUST invoke the start command with the decided execution strategy.

[†](README.md#linux-only)When executing a process using any execution strategy, the launcher SHOULD replace the launcher process in memory without forking it.

[†](README.md#linux-only)When executing a process with Bash, the launcher SHOULD additionally replace the Bash process in memory without forking it.

[‡](README.md#windows-only)When executing a process with Command Prompt, the launcher SHOULD start a new process with the same security context, terminal, working directory, STDIN/STDOUT/STDERR handles and environment variables as the Command Prompt process.

##### Outputs

If the launcher errors before executing the process it will have one of the following error codes:

| Exit Code | Result                              |
|-----------|-------------------------------------|
| `11`      | Platform API incompatibility error  |
| `12`      | Buildpack API incompatibility error |
| `80-89`   | Launch-specific lifecycle errors    |

Otherwise, the exit code shall be the exit code of the launched process.

The launcher:
- MUST construct the process execution environment as described in [Launch Environment](#launch-environment)
- MUST execute the selected process as specified in the [Buildpack Interface Specfication](buildpack.md)
- SHOULD replace the lifecycle with the process in memory without forking it.

### Run Image Resolution

Given [run](#runtoml-toml) metadata shall be resolved as follows:
- By choosing the `<run-image>` for a given `<app-image>`:
  - **If** any of `image.image` or `image.mirrors` has a registry matching that of `<app-image>` and is accessible with read permissions:
    - This value will become the `<run-image>`
  - **If** none of `image.image` or `image.mirrors` has a registry matching that of `<app-image>`:
    - The first value of `image.image` or `image.mirrors` that is accessible with read permissions will become the `<run-image>`
- By choosing mirrors information for a given `<run-image>`:
  - The first image in `[[images]]` where `image.image` or one of `image.mirrors` matches `<run-image>`
  - **Else** the first image in `[[images]]`

### Registry Authentication

The platform MAY set `CNB_REGISTRY_AUTH`  in the lifecycle execution environment, where value of `CNB_REGISTRY_AUTH` MUST be valid JSON object and MAY contain any number of `<regsitry>` to `<auth-header>` mappings.
If `CNB_REGISTRY_AUTH` is set and `<registry>` matches the registry of an image reference, the lifecycle SHOULD set the value of the `Authorization` HTTP header to `<auth-header>` when attempting to read or write the image located at the given reference.

If `CNB_REGISTRY_AUTH` is unset and a docker [config.json](https://docs.docker.com/engine/reference/commandline/cli/#configjson-properties) file is present, the lifecycle SHOULD use the contents of this file to authenticate with any matching registry.
The lifecycle SHOULD adhere to established docker conventions when checking for the existence of or interpreting the contents of a `config.json` file.

The lifecycle MAY provide other mechanisms by which a platform can supply registry credentials.

The lifecycle MUST attempt to authenticate anonymously if no matching credentials are found.

### Experimental Features

Where noted, certain features are considered experimental and susceptible to change in a future API version.

The platform SHOULD set `CNB_EXPERIMENTAL_MODE=<warn|error|silent>` in the lifecycle's execution environment to control the behavior of experimental features.

When an experimental feature is invoked, the lifecycle:
- SHALL fail if `CNB_EXPERIMENTAL_MODE` is unset
- SHALL warn and continue if `CNB_EXPERIMENTAL_MODE=warn`
- SHALL fail if `CNB_EXPERIMENTAL_MODE=error`
- SHALL continue without warning if `CNB_EXPERIMENTAL_MODE=silent`

## Buildpacks

### Buildpacks Directory Layout

The buildpacks directory MUST contain unarchived buildpacks such that:

- Each top-level directory is a buildpack ID.
- Each second-level directory is a buildpack version.

## Image Extensions

### Image Extensions Directory Layout

The image extensions directory MUST contain unarchived image extensions such that:

- Each top-level directory is an image extension ID.
- Each second-level directory is an image extension version.

## Security Considerations

The platform SHOULD run each phase of the lifecycle in an isolated container to prevent untrusted app and buildpack code from accessing storage credentials needed during the export and analysis phases.
A more thorough explanation is provided in the [Buildpack Interface Specification](buildpack.md).

## Additional Guidance

### Environment

#### Buildpack Environment

##### Base Image-Provided Variables

The following variables SHOULD be set in the lifecycle execution environment and SHALL be directly inherited by the
buildpack without modification:

| Env Variable                | Description                    |
|-----------------------------|--------------------------------|
| `HOME`                      | Current user's home directory  |

##### POSIX Path Variables

The following variables SHOULD be set in the lifecycle execution environment and MAY be modified by prior buildpacks before they are provided to a given buildpack:

| Env Variable      | Layer Path   | Contents         |
|-------------------|--------------|------------------|
| `PATH`            | `/bin`       | binaries         |
| `LD_LIBRARY_PATH` | `/lib`       | shared libraries |
| `LIBRARY_PATH`    | `/lib`       | static libraries |
| `CPATH`           | `/include`   | header files     |
| `PKG_CONFIG_PATH` | `/pkgconfig` | pc files         |

The platform SHOULD NOT assume any other base-image-provided environment variables are inherited by the buildpack.

##### User-Provided Variables

User-provided environment variables MUST be supplied by the platform as files in the `<platform>/env/` directory.

Each file SHALL define a single environment variable, where the file name defines the key and the file contents define the value.

User-provided environment variables that are [POSIX path variables](#posix-path-variables) MAY be modified by prior buildpacks before they are provided to a given buildpack,
however the user-provided value is always prepended to the buildpack-provided value.

User-provided environment variables that are not POSIX path variables MAY NOT be modified by prior buildpacks before they are provided to a given buildpack.

The platform SHOULD NOT set user-provided environment variables directly in the lifecycle execution environment.

##### Operator-Defined Variables

Operator-provided environment variables MUST be supplied by the platform as files in the `<build-config>/env/` directory.

Each file SHALL define a single environment variable, where the file name defines the key and the file contents define the value.

The `<build-config>/env/` directory follows the [Environment Variable Modification Rules](https://github.com/buildpacks/spec/blob/main/buildpack.md#environment-variable-modification-rules)
outlined in the [Buildpack Interface Specification](buildpack.md), except for the modification behavior when no period-delimited suffix is provided; when no suffix is provided, the behavior is `default`.

Operator-defined environment variables MAY be modified by prior buildpacks before they are provided to a given buildpack,
however the operator-defined value is always applied after the buildpack-provided value.

The platform SHOULD NOT set operator-provided environment variables directly in the lifecycle execution environment.

#### Launch Environment

User-provided modifications to the process execution environment SHOULD be set directly in the lifecycle execution environment.

The process SHALL inherit both base-image-provided and user-provided variables from the lifecycle execution environment with the following exceptions:
* `CNB_APP_DIR`, `CNB_LAYERS_DIR` and `CNB_PROCESS_TYPE` SHALL NOT be set in the process execution environment.
* `/cnb/process` SHALL be removed from the beginning of `PATH`.
* The lifecycle SHALL apply buildpack-provided modifications to the environment as outlined in the [Buildpack Interface Specification](buildpack.md).

### Caching

If caching is enabled the platform is responsible for providing the lifecycle with access to the correct cache.
Whenever possible, the platform SHOULD provide the same cache to each rebuild of a given app image.
Cache locality and availability MAY vary between platforms.

### Build Reproducibility

When given identical inputs all build and rebase operations:
   - SHOULD produce app images with identical imageIDs
   - **If** exporting directly to a registry
       - SHOULD produce app images with identical manifest digests
   - MAY output other non-reproducible artifacts

To achieve reproducibility the lifecycle SHOULD set the following to a constant, rather than an accurate value:
- file modification times in generated layers
- image creation time

Because compressions algorithms and manifest whitespace affect the image digest, an app image exported to the docker daemon and subsequently pushed to a registry MAY have a different digest than an app image exported directly to a registry by the lifecycle, even when all other inputs are held constant.

If buildpacks do not generate layer contents or layer metadata reproducibly, builds MAY NOT be reproducibile even when identical source code and buildpacks are provided to the lifecycle.

All app image labels SHOULD contain only reproducible values.

For more information on build reproducibility see [https://reproducible-builds.org/](https://reproducible-builds.org/)

### Map an image reference to a path in the layout directory

An **image reference** refers to either a **tag reference** or a **digest reference**:
- A tag reference refers to an identifier of form `<registry>/<repo>/<image>:<tag>`
- A digest reference refers to a content addressable identifier of form `<registry>/<repo>/<image>@<algorithm>:<digest>`

The image reference will be mapped to a path in the layout directory following these rules:
  - **If** the image points to a tag reference:
    - The path MUST be `<layout-dir>/<registry>/<repo>/<image>/<tag>`
  - **Else if** the image points to a digest reference:
    - The path MUST be `<layout-dir>/<registry>/<repo>/<image>/<algorithm>/<digest>`

## Data Format

### Files

#### `analyzed.toml` (TOML)

```toml
[image]
  reference = "<image reference>"

[metadata]
# layer metadata

[run-image]
  image = "<image>"
  reference = "<image reference>"
  extend = false
  [target]
  os = "<OS name>"
  arch = "<architecture>"
  variant = "<architecture variant>"
  [target.distro]
  name = "<OS distribution name>"
  version = "<OS distribution version>"

[build-image]
  reference = "<image reference>"
```

Where:
- `previous-image.reference` MUST be either: 
  - A digest reference to an image in an OCI registry 
  - The ID of an image in a docker daemon
  - The path to an image in OCI layout format
- `previous-image.metadata` MUST be the TOML representation of the layer [metadata label](#iobuildpackslifecyclemetadata-json)
- `run-image.reference` MUST be either:
  - A digest reference to an image in an OCI registry
  - The ID of an image in a docker daemon
  - The path to an image in OCI layout format
- `run-image.image` MUST be the platform- or extension-provided image name
- `run-image.target` contains the [target data](#target-data) for the image
  - If target distribution data is missing, it will be inferred from `/etc/os-release` for Linux images; furthermore, if the image contains the label `io.buildpacks.stack.id` with value `io.buildpacks.stacks.bionic`, the lifecycle SHALL assume the following values:
    - `run-image.target.os = "linux"`
    - `run-image.target.arch = "arm64"`
    - `run-image.target.distro.name = "ubuntu"`
    - `run-image.target.distro.version = "18.04"`

#### `group.toml` (TOML)

```toml
[[group]]
id = "<buildpack ID>"
version = "<buildpack version>"
api = "<buildpack API version>"
homepage = "<buildpack homepage>"

[[group-extensions]]
id = "<image extension ID>"
version = "<image extension version>"
api = "<image extension API version>"
homepage = "<image extension homepage>"
```

Where:

- `id`, `version`, and `api` MUST be present for each buildpack object in a group.

#### `metadata.toml` (TOML)
```toml
buildpack-default-process-type = "<process type>"

[[buildpacks]]
id = "<buildpack ID>"
version = "<buildpack version>"
api = "<buildpack API version>"
optional = false

[[extensions]]
id = "<image extension ID>"
version = "<image extension version>"
api = "<image extension API version>"

[[processes]]
type = "<process type>"
command = ["<command>"]
args = ["<arguments>"]
direct = false
working-dir = "<working directory>"

[[slices]]
paths = ["<app sub-path glob>"]
```

Where:
- `id`, `version`, and `api` MUST be present for each buildpack
- `processes` contains the complete set of processes contributed by all buildpacks
- `slices` contains the complete set of slices defined by all buildpacks

#### `order.toml` (TOML)

```toml
[[order]]
[[order.group]]
id = "<buildpack ID>"
version = "<buildpack version>"
optional = false

[[order-extensions]]
[[order-extensions.group]]
id = "<image extension ID>"
version = "<image extension version>"
```

Where:

- Both `id` and `version` MUST be present for each buildpack object in a group.
- The value of `optional` MUST default to `false` if not specified.

#### `plan.toml` (TOML)

```toml
[[entries]]

  [[entries.providers]]
    id = "<buildpack or image extension ID>"
    version = "<buildpack or image extension version>"
    extension = false

  [[entries.requires]]
    name = "<dependency name>"
    [entries.requires.metadata]
      # arbitrary data describing the required dependency
```
Where:
- `entries` MAY be empty
- Each entry:
    - MUST contain at least one buildpack or image extension in `providers`
      - If the provider is an image extension (optional and **[experimental](#experimental-features)**), `extension` MUST be `true`; the value of `extension` MUST default to `false` if not specified
    - MUST contain at least one dependency requirement in `requires`
    - MUST exclusively contain dependency requirements with the same `<dependency name>`

#### `project-metadata.toml` (TOML)

```toml
[source]
type = "<source type>"

[source.version]
# arbitrary data

[source.metadata]
# arbitrary data
```

Where:
- All values are optional
- `type`, if present, SHOULD contain the type of location where the provided app source is stored (e.g `git`, `s3`)
- `version`, if present, SHOULD contain data uniquely identifying the particular version of the provided source
- `metadata` MAY contain additional arbitrary data about the provided source

#### `report.toml` (TOML)

```toml
[image]
tags = ["<tag reference>"]
digest = "<image digest>"
image-id = "<imageID>"
manifest-size = "<manifest size in bytes>"
```
Where:
- `tags` MUST contain all tag references to the exported app image
- **If** the app image was exported to an OCI registry
  - `digest` MUST contain the image digest
  - `manifest-size` MUST contain the manifest size in bytes
- **If** the app image was exported to a docker daemon
  - `imageID` MUST contain the imageID

#### `run.toml` (TOML)

```toml
[[images]]
 image = "<image>"
 mirrors = ["<mirror1>", "<mirror2>"]
```

Where:
- `image.image` MAY be a tag reference to a run image in an OCI registry
- `image.mirrors` MUST NOT be present if `image.image` is not present
- `image.mirrors` MAY contain one or more tag references to run images in OCI registries
- All `image.mirrors`:
  - SHOULD reference an image with ID identical to that of `image.image`
- `image.image` and `image.mirrors.[]` SHOULD each refer to a unique registry

### Labels

#### `io.buildpacks.build.metadata` (JSON)

```javascript
{
  "processes": [
    {
      "type": "<process-type>",
      "command": ["<command>"],
      "args": [
        "<args>"
      ],
      "direct": false,
      "working-dir": "<working-dir>",
      "buildpackID": "<buildpack ID>"
    }
  ],
  "buildpacks": [
    {
      "id": "<buildpack ID>",
      "version": "<buildpack version>",
      "homepage": "<buildpack homepage>",
      "api": "<buildpack API version>"
    }
  ],
  "extensions": [
    {
      "id": "<extension ID>",
      "version": "<extension version>",
      "homepage": "<extension homepage>",
      "api": "<buildpack API version>"
    }
  ],
 "launcher": {
    "version": "<launcher-version>",
    "source": {
      "git": {
        "repository": "<launcher-source-repository>",
        "commit": "<launcher-source-commit>"
      }
    }
  }
}
```
Where:
- `processes` MUST contain all buildpack contributed processes
- `buildpacks` MUST contain the detected group
- `launcher.version` SHOULD contain the version of the `launcher` binary included in the app
- `launcher.source.git.repository` SHOULD contain the git repository containing the `launcher` source code
- `launcher.source.git.commit` SHOULD contain the git commit from which the given `launcher` was built

#### `io.buildpacks.lifecycle.metadata` (JSON)

```javascript
{
  "app": [
    {"sha": "<slice-layer-diffID>"}
  ],
  "sbom": {
    "sha": "<BOM-layer-diffID>"
  },
  "config": {
    "sha": "<config-layer-diffID>"
  },
  "launcher": {
    "sha": "<launcher-layer-diffID>"
  },
  "buildpacks": [
    {
      "key": "<buildpack-id>",
      "version": "<buildpack-version>",
      "layers": {
        "<layer-name>": {
          "sha": "<layer-diffID>",
          "data": {},
          "build": false,
          "launch": false,
          "cache": false
        }
      }
    }
  ],
  "runImage": {
    "image": "cnbs/sample-stack-run:bionic",
    "mirrors": = ["<mirror1>", "<mirror2>"],
    "topLayer": "<run-image-top-layer-diffID>",
    "reference": "<run-image-reference>"
  }
}
```

Where:
- `app` MUST contain one entry per app slice layer where
  - `sha` MUST contain the digest of the uncompressed layer
- `sbom.sha` MUST contain the digest of the uncompressed layer containing buildpack-provided Software Bill of Materials
- `config.sha` MUST contain the digest of the uncompressed layer containing launcher config
- `launcher.sha` MUST contain the digest of the uncompressed layer containing the launcher binary
- `buildpacks` MUST contain one entry per buildpack that participated in the build where
  - `key` is required and MUST contain the buildpack ID
  - `version` is required and MUST contain the buidpack Version
  - `layers` is required and MUST contain one entry per launch layer contributed by the given buildpack.
  - For each entry in `layers`:
    - The key  MUST be the name of the layer
    - The value MUST contain JSON representation of the `layer.toml` with an additional `sha` key, containing the digest of the uncompressed layer
    - The value MUST contain an additional `sha` key, containing the digest of the uncompressed layer
- `runImage.image` and `runImage.mirrors` MUST be [resolved](#run-image-resolution) from `run.toml` from the given `<run-image>`
- `runImage.topLayer` MUST contain the uncompressed digest of the top layer of the run-image
- `runImage.reference` MUST uniquely identify the run image. It MAY contain one of the following
  - An image ID (the digest of the uncompressed config blob)
  - A digest reference to a manifest stored in an OCI registry

#### `io.buildpacks.project.metadata` (JSON)

```javascript
{
  "source": {
    "type": "<type>",
    "version": {
     // arbitrary version data
    },
    "metadata": {
    // arbitrary data
    }
  }
}
```

This label MUST contain the JSON representation of [`project-metadata.toml`](#project-metadatatoml-toml)


## Deprecations
This section describes all the features that are deprecated.

### `io.buildpacks.stack.*` Labels

_Deprecated in Platform API 0.12._

For compatibility with older platforms and older buildpacks, base image authors SHOULD ensure for build images and run images:

- The image config's `Env` field has the environment variable `CNB_STACK_ID` set to the stack ID.
- The image config's `Label` field has the label `io.buildpacks.stack.id` set to the stack ID.
- The image config's `Label` field has the label `io.buildpacks.stack.mixins` set to a JSON array containing mixin names for each mixin applied to the image.

Where `CNB_STACK_ID` SHALL be directly inherited by buildpacks without modification.

To upgrade, the platform SHOULD upgrade all buildpacks to use Buildpack API `0.10` or greater.

### `io.buildpacks.lifecycle.metadata` (JSON) `stack` Key

_Deprecated in Platform API 0.12._

The `stack` key is deprecated.

```json
  "stack": {
    "runImage": {
      "image": "cnbs/sample-stack-run:bionic",
      "mirrors": ["<mirror1>", "<mirror2>"]
    }
  }
```

Where `stack` MUST contain the same data as the top-level `runImage` key.

To upgrade, the platform SHOULD read the top-level `runImage` key instead.
