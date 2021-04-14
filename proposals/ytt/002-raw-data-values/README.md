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


##### Syntax

```console
ytt ... --data-values-file [<libref>]<filepath>
```
where:
- `filepath` refers to a file that contains plain YAML. Data Values will be extracted and set from its contents.
  - there are no restrictions on the name of the file, but the `.yaml` or `.yml` extension is recommended.
- `libref` directs the values to the specified private library
  - format: `@(<library-name> | ~<alias>):` (i.e. identical to the syntax for other `--data-value...` flags)

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


##### Multiple Documents

`--data-values-file` can be specified multiple times. The file supplied in as a parameter can contain zero or more documents.

Regardless the means, each instance of a Data Value overwrites the previous value, as if it were specified via the `--data-value-yaml` flag.

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


##### Interaction with Schema

There are no new interactions between Data Values supplied with this new flag and schema. Data Values supplied with this new flag are checked and validated against schema in exactly the same way.


##### Interactions with Data Values Overlays

There are no new interactions between Data Values supplied with this new flag and Data Values Overlays:

- just as values supplied via other `--data-value...` flags have higher precedence over any Data Values Overlays, so too with the values specified by this new flag.
- users will not be _required_ to supply data values via the `--data-values-file` flag; users are free to continue specifying these values via Data Values Overlays.


##### Interactions with Private Libraries

As noted above in [Syntax](#syntax), above, a user may optionally specify the exact library that should receive the Data Values.


##### Miscellaneous

- YAML comments (i.e. strings prefixed with `#`) are permitted in Data Values files given they are treated as plain YAML and not a `ytt` template.


##### Consideration: Confusingly similar to `--data-value-file`

_(the inputs to these two flags are in a different format: it will be relatively easy to detect, and redirect the user, when one flag is used when the other was intended.)_

It will likely be crucial for a possible user experience that the messaging around these two flags takes into account this possible confusion or simple mistype.


##### Consideration: Does not support merging arrays

_(Yup. Our experience shows that typically more advanced users have this need. It's still available via Data Values Overlays.)_

