---
title: Data Values as Plain YAML
authors:
  - John Ryan <ryanjo@vmware.com>
  - Steven Locke <slocke@vmware.com>
status: draft
approvers:
  - Dmitriy Kalinin <dkalinin@vmware.com>
  - Eli Wrenn <ewrenn@vmware.com>
---

# Data Values as Plain YAML

## Problem Statement

Today, users primarily supply Data Values using Data Values Overlays.

Configuration Consumers providing configuration data to a `ytt` invocation is required to do so by capturing those as a YAML document and annotating it _as_ a Data Values Overlay (i.e. `@data/values`). Further, they must resolve any merge ambiguities by annotating the document with `@overlay/...`.

Examples:
- https://github.com/vmware-tanzu/carvel-ytt/issues/81
- https://github.com/vmware-tanzu/carvel-ytt/issues/51

These requirements also make integrating `ytt` into automation tooling less than desirable: the `ytt` concepts of "data values" and "overlays" are foisted on the hapless end user of that automation tooling when all they wanted to do was customize their use of some higher-level feature.

Example:
- the `values` section of an `InstalledPackage` custom resource: https://carvel.dev/kapp-controller/docs/latest/package-consumption/#installing-a-package

Likewise, some Configuration Consumers (i.e. direct users of `ytt`) may be supplying Data Values via upstream tooling. These users shoulder the glue work required to integrate `ytt` into their toolchain.

**What's needed is a mode of accepting Data Values into a `ytt` invocation stripped of any requirements to understand overlay concepts or syntax.**

## Terminology / Concepts

- **`data` module** : the `ytt` module that supplies Data Values to templates (typically via `load("@ytt:data", "data")`).
- **Data Value** : an input into `ytt` (typically a key/value pair in a YAML document) that ultimately is presented to templates via the `data` module.
- **Data Values** : a colloquial phrase referring to the full set of Data Values.
- **Data Values Overlay** : a YAML document annotated with `@data/values`. It optionally contains `@overlay` annotations to resolve matching ambiguities and indicate the action/edit desired. These overlays are plucked from the set of files provided in an `ytt` invocation and processed first, before any templates are evaluated.
- "**plain YAML file**" : short-hand for a file whose contents are in YAML format and contain no `ytt` annotations.
- **private library** : a `ytt` library acting as a dependency on the primary (aka "Root") library. (see also [`ytt` docs > Library Module](https://carvel.dev/ytt/docs/latest/lang-ref-ytt-library/#library-module))

## Proposal

In short:
- introduce a new flag: `--data-values-file`;
- accept only plain YAML;
- behaves like a set of `--data-value-yaml` parameters;
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

Extend this family of flags to be _the_ first-class means of supplying Data Values, including a full set of values via a plain YAML file. This will be done via a new flag: `--data-values-file`

Syntax:

```console
ytt ... --data-values-file [<libref>][+:]<filepath>
```
where:
- `libref` directs the values to the specified private library
  - format: `@(<library-name> | ~<alias>):` (i.e. identical to the syntax for other `--data-value...` flags)
- `+:` if present indicates that the Data Values in the document(s) should be set whether or not they have been previously declared.
- `filepath` refers to a file that contains plain YAML. Data Values will be extracted and set from its contents.
  - there are no restrictions on the name of the file, but the `.yaml` or `.yml` extension is recommended.

Given how common this flag will be used, it also has a shorthand form:

```console
ytt ... -d [<libref>][+:]<filepath>
```


#### Multiple Documents

`--data-values-file` can be specified multiple times. The file supplied in as a parameter can contain zero or more documents.

Regardless the means, each instance of a Data Value overwrites the previous value, as if it were specified via the `--data-value-yaml` flag.


#### Interaction with Schema

There are no new interactions between Data Values supplied with this new flag and schema. 
- Data Values supplied with this new flag are typed, checked, and validated against schema in exactly the same way as Data Values specified via the other `--data-value` flags.


#### Interactions with Data Values Overlays

There are no new interactions between Data Values supplied with this new flag and Data Values Overlays:

- just as values supplied via other `--data-value...` flags have higher precedence over any Data Values Overlays, so too with the values specified by this new flag.
- users will not be _required_ to supply data values via the `--data-values-file` flag; users are free to continue specifying these values via Data Values Overlays.
- documentation and examples need adjusting to clearly message that _this_ mechanism is preferred for most use-cases


#### Interactions with Private Libraries

A user may optionally indicate the exact library that should receive the Data Values. This is done using the `libref` notation described at the start of [Specification](#specification).


#### Miscellaneous

- YAML comments (i.e. strings prefixed with `#`) are permitted in Data Values files given they are treated as plain YAML and not a `ytt` template.


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




### Examples 

#### Example: Single document

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

#### Example: Multiple instances of `--data-values-file`

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

#### Example: Multiple documents in a file

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

#### Example: Empty file has no effect

_Empty files are accepted but are, in effect, a no-op._

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

#### Example: Declaring additional data values

_Absent schema, additional data values can be declared._

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
$ ytt --data-values-file values.yml --data-values-file +values2.yml --data-values-inspect
foo: 13
bar:
- first
- second
ree: true
```

#### Example: Process substitution
#### Example: Target a library
#### Example: Target a library alias

#### Example: Scalar Data Value

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

#### Example: 

**Example:**

Given a file:

`values.yml`
```yaml
---
foo:
  bar: 42
ree:
- 1
- 2
- 3
```

```console
$ ytt -f config/ --data-values-file ../dev/values.yml --data-values-inspect
foo:
  bar: 42
ree:
- 1
- 2
- 3
```
Given the following files:

`alpha.yaml`
```yaml
foo:
  bar: 13
```

`beta.yaml`
```yaml
foo:
  bar: 42
```

`omega.yaml`
```yaml
---
foo:
  bar: 13
---
foo:
  bar: 42
```

All of the following are equivalent:

```console
$ ytt ... --data-values-file alpha.yaml --data-values-file beta.yaml
$ ytt ... --data-values-file omega.yaml
$ ytt ... -v foo.bar=13 -v foo.bar=42
```
