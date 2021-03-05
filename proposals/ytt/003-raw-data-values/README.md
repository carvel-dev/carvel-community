---
title: Accept Unannotated YAML as Data Values
authors: 
- John Ryan <ryanjo@vmware.com>
- Steven Locke <slocke@vmware.com>
status: draft
approvers: 
- Dmitriy Kalinin <dkalinin@vmware.com>
- Eli Wrenn <ewrenn@vmware.com>
---

# Accept Unannotated YAML as Data Values

## Problem Statement

Today, Configuration Consumers providing configuration data to a `ytt` invocation is required to do so by capturing those as a YAML document and annotating it _as_ a Data Values document. Further -- given that all Data Values documents are overlays, they must resolve any merge ambiguities by annotating the document with `@overlay/...`.

These requirements make integrating `ytt` into automation tooling less than desirable: the `ytt` concepts of "data values" and "overlays" are foisted on the happless end user of that automation tooling when all they wanted to do was customize their use of some higher-level feature (e.g. the `values` portion of an `InstalledPackage`).

Likewise, some Configuration Consumers (i.e. direct users of `ytt`) may be supplying Data Values via upstream tooling. These users shoulder the glue work required to integrate `ytt` into their toolchain.

**What's needed is a mode of accepting Data Values into a `ytt` invocation stripped of any requirements to understand `ytt` concepts or syntax.**

That said, doing so must not dimish the reliability and safety that `ytt` aims to bring.  Specifically, `ytt` provides Configuration Consumers with fast feedback on likely errors in specifying Data Values.

There are two (2) broad scenarios considered:
- [Scenario: Consumer Configuration is Additive](#scenario-consumer-configuration-is-additive)
- [Scenario: Consumer Configuration (of arrays) To Overwrite Defaults](#scenario-consumer-configuration-(of-arrays)-to-overwrite-defaults)

**See Also:**

- https://carvel.dev/kapp-controller/docs/latest/package-consumption/#installing-a-package
- https://github.com/vmware-tanzu/carvel-ytt/issues/81
- https://github.com/vmware-tanzu/carvel-ytt/issues/51

### Scenario: Consumer Configuration is Additive

Configuration Consumer finds the library's defaults an acceptable basis on which they can meet their needs by merging in their own values.

In this case:
- schema defines the full set of Data Values with reasonable defaults.
- Configuration Consumer: overrides specific scalars and _merge_ values for arrays

**Author Inputs:**

```yaml
#! schema.yml
#@schema/definition data_values=True
---
foo: ""
bars:
- ""
rees: 1
```

```yaml
#! default-values.yml
#@data/values
---
bars:
- barA
- barB
- barC
```

```yaml
#! template.yml
#@ load("@ytt:data", "data")
---
values: #@ data.values
```

**Consumer Input:**

`values.yml`
```yaml
foo: fooy
bars:
- bar1
- bar2
```

**Desired Output:**

```yaml
values:
  foo: fooy
  bars:
  - barA
  - barB
  - barC
  - bar1
  - bar2
  rees: 1
```

### Scenario: Consumer Configuration (of arrays) To Overwrite Defaults

The Configuration Author intends to provide a simple Out of The Box experience by providing defaults, even for configuration that are arrays.
The Configuration Consumer needs to _overwrite_ any existing config, rather than merge.

In this case:
- schema defines the full set of Data Values with reasonable defaults.
- one or more arrays in the schema is non-empty
- Consumer-supplied Data values includes values for one or more of those non-empty arrays.

**Authored Inputs:**

```yaml
#! schema.yml
#@schema/definition data_values=True
---
foo: ""
bars:
- ""
rees: 1
```

```yaml
#! default-values.yml
#@data/values
---
bars:
- barA
- barB
- barC
```

```yaml
#! template.yml
#@ load("@ytt:data", "data")
---
values: #@ data.values
```

**Consumer Input:**

`values.yml`
```yaml
foo: fooy
bars:
- bar1
- bar2
```

**Desired Output:**

```yaml
values:
  foo: fooy
  bars:
  - bar1
  - bar2
  rees: 1
```


## Proposal

### Goals and Non-goals

**Goals**
- Make it possible for user-supplied configuration devoid of any `ytt`-specific syntax to flow to `ytt` (typically _through_ other tooling) and affect data values without any special handling for the majority of use cases.
- Avoid introducing situation-specific mechanisms.

**Non-Goals**
- Provide a way to edit data values from the command-line with feature-parity of overlay annotations.


### Specification

`ytt` has an existing mechanism for indicating what type of file a given input is: [file marks](https://carvel.dev/ytt/docs/latest/file-marks/).

Meet the specified needs by introducing another file type: `yaml-data-values`.

This type extends the "family" of file types: `yaml-plain` and `yaml-template` which also provide the Consumer with the ability to override `ytt`'s inferred typing.

Marking a file as `type:yaml-data-values` is equivalent to annotating it with `@data/values`.

From the CLI:
```console
$ ytt -f . --file-mark='values.yml:type=yaml-data-values'
```

Within a Starlark program: _(this already exists, today)_
```python
library.with_data_values(...)
```

Through `kapp-controller`:
```yaml
---
apiVersion: package.carvel.dev/v1alpha1
kind: Package
metadata:
  name: simple-app.corp.com.1.0.0
spec:
  publicName: simple-app.corp.com
  version: 1.0.0
  displayName: "simple-app v1.0.0"
  description: "Package for simple-app version 1.0.0"
  template:
    spec:
      fetch:
      - imgpkgBundle:
          image: k8slt/kctrl-example-pkg:v1.0.0
      template:
      - ytt:
          paths:
          - "config.yml"
          - "values.yml"
          fileMarks:
          - path: "values.yml"
            type: "yaml-data-values"
      - kbld:
          paths:
          - "-"
          - ".imgpkg/images.yml"
      deploy:
      - kapp: {}
```

Specifically, note the `/spec/template/spec/template[0]/fileMarks` section.

#### Consideration: Imposed Restrictions on Marked Files

`@data/values` _technically_ is a YAML Document-level annotation. Marking an entire file as "data values" means that _all_ YAML documents within it would be considered as such.  Notably, doing so conflicts with any future feature that aims to support Data Values co-existing with the templates that use them.

- currently, `ytt` disallows any other kind of document in a given file. So, this restriction is -- in practice -- not new.
- this "constraint" is limited to _only_ files explicitly marked in this fashion.
- as Schema matures, it would more likely be the kind of YAML document mixed with templates should such a structure truly want to be supported.


#### Consideration: Array Items Requiring an @overlay/match

Array items have traditionally required _some_ kind of matching as there was no default operation defined for them. This meant that overlays on documents that contained arrays could not be written without including such annotations.

At the time of writing, a feature that extends the default merge operation to array items has _just_ been merged into `develop`. Therefore, for the base use-case, there are no annotations required.


#### Consideration: Does not Support Replace Use-Case

This option does not answer directly for other kinds of overlay operations besides merge. Of most notable concern is [Use Case: Array Replace](#use-case-array-replace): replacing the contents of an array that had a non-empty default value.

Given:
- in practice, few Configuration Authors provide default values for arrays. Typically, such values source from the Consumer's context rather than the authors.
- in the rare case where a Configuration Consumer _does_ wish to customize the processing of their data values, they have the full power of `ytt`'s Overlay features available to them.
- in the specific case of replacing the contents of an array, this amounts to understanding one (1) annotation: `@overlay/replace`.

We believe there is no need, at this time, to provide a convenience mechanism to affect any other than the default overlay operations on data values.


### Other Approaches Considered

There was one other approach seriously considered: introducing a variety of the `--file` flag group.

#### Introduce Files Argument for Data Values

`ytt` also sports the `--file...` family of command-line flags.

Another option would be to extend that group of flags with `--file-data-values`.

This syntax is more succinct and conceptually lighter than the file mark approach.

This approach two concepts: 
1. one indicates a file for input using `-f`; and 
2. one can customize that introduction using the `--file-data-values` so that _that_ file will be handled as a data values input.

The file mark approach lights-up X concepts: 
1. one indicates a file for input using `-f`; and 
2. individual files implied by `--file` can be explicitly typed with "file marks"
3. one kind of "file mark" is "type"; of those types, one of which is `yaml-data-values`
4. to include an input as explicit a data values, it _both_ must be included in the set of files implied by the union of the `-f` inputs _and_ be explicitly marked with the `--file-mark '(path):type=yaml-data-values'` flag.

That's a significant difference in cognitive load on the user.


## Open Questions

1. 

## Answered Questions

