---
title: Data Values as Plain YAML
authors:
  - John Ryan <ryanjo@vmware.com>
  - Steven Locke <slocke@vmware.com>
status: in-review
approvers:
  - Dmitriy Kalinin <dkalinin@vmware.com>
  - Eli Wrenn <ewrenn@vmware.com>
---

# Data Values as Plain YAML

- [Problem Statement](#problem-statement)
- [Terminology / Concepts](#terminology--concepts)
- [Proposal](#proposal)
  - [Specification](#specification)
  - [Examples](#examples)
  
## Problem Statement

Today, users primarily supply Data Values using Data Values Overlays.

Configuration Consumers providing configuration data to a `ytt` invocation is required to do so by capturing those as a YAML document and annotating it _as_ a Data Values Overlay (i.e. `@data/values`). Further, they must resolve any merge ambiguities by annotating the document with `@overlay/...`.

Examples:
- https://github.com/vmware-tanzu/carvel-ytt/issues/81
- https://github.com/vmware-tanzu/carvel-ytt/issues/51

These requirements also make integrating `ytt` into automation tooling less than desirable: the `ytt` concepts of "data values" and "overlays" are foisted on the hapless end user of that automation tooling when all they wanted to do was customize their use of some higher-level feature.

Examples:
- the `values` section of an `InstalledPackage` custom resource: https://carvel.dev/kapp-controller/docs/latest/package-consumption/#installing-a-package
- in GitLab, [generating child pipelines](https://docs.gitlab.com/ee/ci/parent_child_pipelines.html#dynamic-child-pipelines) based on plain YAML configuration input.

Likewise, some Configuration Consumers (i.e. direct users of `ytt`) may be supplying Data Values via upstream tooling. These users shoulder the glue work required to integrate `ytt` into their toolchain.

**What's needed is a mode of accepting Data Values into a `ytt` invocation stripped of any requirements to understand overlay syntax or concepts.**

## Terminology / Concepts

These terms are used in this proposal:

- **`data` module** : the [`ytt` module](https://carvel.dev/ytt/docs/latest/lang-ref-ytt/#data) that supplies Data Values to templates (typically via `load("@ytt:data", "data")`).
- **Data Value** : an input into `ytt` (typically a key/value pair in a YAML document) that ultimately is presented to templates via the `data` module.
- **Data Values** : an overloaded term, loosely meaning a bunch of "Data Value"s. In practice it can mean:
  1. the final set of values resulting from merging all the Data Value inputs of a given `ytt` run (i.e. the output of `--data-values-inspect`).
  2. a YAML file containing a set of "Data Value"s.
- **Data Values File** : a plain YAML file that contains Data Values.
  - this is the primary object of this proposal.
- **Data Values Overlay** : a YAML document annotated with `@data/values`. 
  - optionally contains `@overlay` annotations to resolve matching ambiguities and indicate the action/edit desired. 
  - separated from all _other_ overlays (e.g. those applied on top the output of templates) and evaluated as a group to calculate the Data Values" for the given `ytt` invocation.
- "**plain YAML file**" : short-hand for a file whose contents are in YAML format and contain no `ytt` annotations.
- **private library** : a `ytt` library acting as a dependency on the primary (aka "Root") library. (see also [`ytt` docs > Library Module](https://carvel.dev/ytt/docs/latest/lang-ref-ytt-library/#library-module))


## Proposal

In short:
- introduce a new flag: `--data-values-file`;
- that flag accepts only plain YAML;
- it behaves like a set of `--data-value-yaml` parameters;
- continue to support Data Values Overlays, but background them as an "advanced feature."

### Goals

- describe a mechanism by which plain YAML can be supplied as Data Values.
- preserve the full functionality provided by (now known as) Data Values Overlays.

### Non-Goals

- achieve feature parity with Data Values Overlays.

### Specification

`ytt` has a family of flags that provide an interface by which Data Values can be supplied.

Today, Data Values...
- can be plucked from OS environment variables (`--data-values-env[-yaml]`) or provided explicitly on the command-line (`--data-value[-yaml]`).
- can be accepted as strings `--data-values{-env,}` or parsed as YAML expressions `--data-values{-env,}-yaml`

We propose to extend this family of flags to be _the_ first-class means of supplying Data Values, including a full set of values via a plain YAML file. This will be done via a new flag: `--data-values-file`

Syntax:

```console
ytt ... --data-values-file [<libref>][+:]<filepath>
```
where:
- `libref` directs the values to the specified private library
  - format: `@(<library-name> | ~<alias>):` (i.e. identical to the syntax for other `--data-value...` flags)
- `+:` if present indicates that the Data Values in the document(s) should be set whether or not they have been previously declared. By default, it is an error to set a Data Value that has not been declared (either in a prior Data Value or Schema).
- `filepath` refers to a file that contains plain YAML. Data Values will be extracted and set from its contents.
  - there are no restrictions on the name of the file, but the `.yaml` or `.yml` extension is recommended.

Given how common this flag will be used, it also has a shorthand form:

```console
ytt ... -d [<libref>][+:]<filepath>
```

(see [Example: Giving a Single YAML Document](#example-giving-a-single-yaml-document) to illustrate the simplest case.)

We consider a number of factors in turn:

- [Multiple Documents](#multiple-documents)
- [Interaction with Schema](#interaction-with-schema)
- [Interactions with Data Values Overlays](#interactions-with-data-values-overlays)
- [Interactions with Private Libraries](#interactions-with-private-libraries)
- [Clear Consistent Messaging Around Data Values](#clear-consistent-messaging-around-data-values)
- [Miscellaneous](#miscellaneous)
- [Consideration: Confusingly similar to `--data-value-file`](#consideration-confusingly-similar-to---data-value-file)
- [Consideration: Does not support merging arrays](#consideration-does-not-support-merging-arrays)

#### Multiple Documents

It is no uncommon that users will want to supply Data Values in not just one YAML document, but many.

This use-case is supported:

- the `--data-values-file` flag can be specified multiple times.
  - left-most instance applied first.
- the file given to the flag can contain zero or more documents.
  - top-most document applied first.
- each instance of a Data Value _replaces_ the previous value (this is already the behavior of the `--data-value[-yaml]` flag.

See also:
- [Example: Giving Multiple YAML Files](#example-giving-multiple-yaml-files)
- [Example: Giving Multiple YAML Documents in a Single File](#example-giving-multiple-yaml-documents-in-a-single-file)


#### Interaction with Schema

There are no new interactions between Data Values supplied with this new flag and ytt Schema. 

Data Values supplied with this new flag are typed, checked, and validated against schema in exactly the same way as Data Values specified via the other `--data-value` flags.


#### Interactions with Data Values Overlays

The file specified with the `--data-values-file` flag are referred to as "Data Values Files"

There are no new interactions between Data Values supplied with this new flag and Data Values Overlays:

- just as values supplied via other `--data-value...` flags have higher precedence over any Data Values Overlays, so too with the values specified by this new flag.
- users will not be _required_ to supply data values via the `--data-values-file` flag; users are free to continue specifying these values via Data Values Overlays.
- documentation and examples need adjusting to clearly message that _this_ mechanism is preferred for most use-cases


#### Interactions with Private Libraries

A user may optionally indicate the exact library that should receive the Data Values. This is done using the `libref` notation described at the start of [Specification](#specification).

See also: [Example: Targeting a Data Values to a Library](#example-targeting-a-data-values-to-a-library).

#### Clear Consistent Messaging Around Data Values

For the features described in this proposal to be well adopted and not create confusion, clear and consistent messaging needs to be crafted.

This includes (but is not limited to):
- the terms "Data Values File" and "Data Values Overlay" be consistently used throughout documentation, examples, and output from the tool itself
- when Data Values are mentioned, the "Data Values File" feature is referenced (rather than the "Data Values Overlay" feature)
- examples use Data Values Files (instead of Data Values Overlays), unless overlay-specific features are being employed


#### Contents of Data Values Files

Data Values Files are plain YAML files that happen to contain Data Values.

As such:
- YAML comments (i.e. strings prefixed with `#`) are permitted (as well as other features enjoyed by plain YAML)
- `ytt` annotations (i.e. strings prefixed with `#@`) are _not_ permitted
  - this prevents end-users from including executable bits, making it possible to better secure integration with `ytt`
    - for example, a `#@ while True:` never returns.
  - when such features are desired, users can employ / integrators can allow Data Values Overlays


#### Consideration: Confusingly similar to `--data-value-file`

One observation is that `--data-values-file` is easily confused or even mistyped to `--data-value-file`.

We believe this similarity is acceptable because:
1. it is relatively easy to detect when the argument given was intended for the other flag; and
2. the actual difference (i.e. the former is the plural of "data value") can help in remembering which flag is applicable when.

Examining the "signature" of each flag:
- `--data-values-file` — has an argument in the format: `<filepath>` (e.g. `../dev/values.yml`)
- `--data-value-file` — has an argument in the format: `<keypath>=<filepath>` (e.g. `server.certificate=../server.cert`)

We note that `--data-value-file` _requires_ the equal sign in the parameter — a character that is highly unlikely to appear in a `filepath`. We expect it trivial to detect when one flag was used with the parameter intended for the other and can provide the needed hint to get the user back on track.

Further, that the newly proposed flag refers to `data-values` (i.e. the plural), it's a mnemonic that the flag is specifying _multiple_ values. In contrast, the existing flag refers to `data-value` (i.e. the singular), a reminder that the value specified is headed for the data value at `keypath`.


#### Consideration: Does not support merging arrays

The careful reader will note that values specified through the `--data-values-file` flag _replaces_ any previously set value. This is most noticeable for values that are arrays.

We observe:
- in the community managing Kubernetes configuration, almost all libraries that declare a configuration value as an array set the default to an empty array (`[]`).
- most users think of supplying data values as "setting" the value (rather than merging).
- in the rare case where the user intends to append values to an array, they can accomplish this through a Data Value Overlay.

Given those conditions, we conclude that the most common cases are best served by _replacing_ the specified values.


### Examples 

1. [Giving a Single YAML Document](#example-giving-a-single-yaml-document)
1. [Giving Multiple YAML Files](#example-giving-multiple-yaml-files)
1. [Giving Multiple YAML Documents in a Single File](#example-giving-multiple-yaml-documents-in-a-single-file)
1. [Giving an Empty File Does Nothing](#example-giving-an-empty-file-does-nothing)
1. [Declaring Additional Data Values](#example-declaring-additional-data-values)
1. [Using Process Substitution to Supply Data Values](#example-using-process-substitution-to-supply-data-values)
1. [Targeting a Data Values to a Library](#example-targeting-a-data-values-to-a-library)
1. [Specifying a Scalar Value](#example-specifying-a-scalar-value)


#### Example: Giving a Single YAML Document

_Absent schema, the first file declares the set of Data Values._

`values.yml`
```yaml
foo: 13
bar:
- name: alpha
- name: beta
```

```console
$ ytt --data-values-file values.yml --data-values-inspect
foo: 13
bar:
- name: alpha
- name: beta
```

#### Example: Giving Multiple YAML Files

_Each file overwrites previously specified values._

`values.yml`
```yaml
foo: 13
bar:
- alpha
- beta
```

`values2.yml`
```yaml
bar:
- first
- second
```

```console
$ ytt --data-values-file values.yml --data-values-file values2.yml --data-values-inspect
foo: 13
bar:
- first
- second
```

#### Example: Giving Multiple YAML Documents in a Single File

_Each document overwrites previously specified values._

`values.yml`
```yaml
foo: 13
bar:
- alpha
- beta
---
bar:
- first
- second
```

```console
$ ytt --data-values-file values.yml --data-values-file values2.yml --data-values-inspect
foo: 13
bar:
- first
- second
```

#### Example: Giving an Empty File Does Nothing

_An empty file is accepted; and is a no-op._

`values.yml`
```yaml
foo: 13
bar:
- alpha
- beta
```

`values2.yml`
```yaml
```

```console
$ ytt --data-values-file values.yml --data-values-file values2.yml --data-values-inspect
foo: 13
bar:
- alpha
- beta
```

#### Example: Declaring Additional Data Values

_Absent schema, additional data values can be declared (if schema is present, new data values must be declared by overlaying schema)._

`values.yml`
```yaml
foo: 13
bar:
- alpha
- beta
```

`values2.yml`
```yaml
bar:
- first
- second
ree: true
```

```console
$ ytt --data-values-file values.yml --data-values-file +:values2.yml --data-values-inspect
foo: 13
bar:
- first
- second
ree: true
```

_Without the `+:` prefix, `ytt` would complain that `ree` was not found._

#### Example: Using Process Substitution to Supply Data Values

_In Unix shells, input can be provided via process substitution. Since there are no restrictions on the name of the file, no custom filename is required._

```console
$ ytt --data-values-file <(echo -e "foo: 13\nbar: [first, second]") --data-values-inspect
foo: 13
bar:
- first
- second
```

_or, given a set of files..._

`values.yml`
```yaml
foo: 13
bar:
- alpha
- beta
```

`values2.yml`
```yaml
bar:
- first
- second
```

```console
$ ytt --data-values-file <(cat values.yml values2.yml) --data-values-inspect
foo: 13
bar:
- first
- second
```

_or, further could be used to merge a set files (as is the case when a large set of Data Values is split into several files)..._

`values.yml`
```yaml
foo: 13
bar:
- alpha
- beta
```

`values2.yml`
```yaml
ree:
- first
- second
```

```console
$ ytt --data-values-file +:<(cat values.yml values2.yml) --data-values-inspect
foo: 13
bar:
- alpha
- beta
ree:
- first
- second
```

#### Example: Targeting a Data Values to a Library

_Configuration Consumers can direct a set of Data Values to a specific library._

```console
$ tree .
.
├── config
│   ├── _ytt_lib
│   │   └── libby
│   │       ├── defaults.yml
│   │       └── template.yml
│   └── main.yml
└── values.yml
```

`libby/defaults.yml`
```yaml
#@data/values
---
foo: 0
```

`libby/template.yml`
```yaml
#@ load("@ytt:data", "data")
---
foo_in_lib: #@ data.values.foo
```

`main.yml`
```yaml
#@ load("@ytt:library", "library")
#@ load("@ytt:template", "template")
--- #@ template.replace(library.get("libby").eval())
```

`values.yml`
```yaml
---
foo: 42
```

```console
ytt -f config/ --data-values-file @libby:values.yml
foo_in_lib: 42
```

#### Example: Specifying a Scalar Value

_While rare, it is valid to specify a single data value that is a scalar._

`values.yml`
```yaml
42
```

`template.yml`
```yaml
#@ load("@ytt:data", "data")
---
answer: #@ data.values
```

```console
$ ytt -f template --data-values-file values.yml
answer: 42
```
