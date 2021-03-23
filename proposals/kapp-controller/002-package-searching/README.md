---
title: "Package Searching"
authors: [ Eli Wrenn <ewrenn@vmware.com> ]
status: "draft"
approvers: [ ]
---

# Package Searching

## Problem Statement

The only existing way to search for packages available on a cluster is through
the use of label selectors and field selectors. Although it exists, the
searching supported by these params is extremely limited out of the box; label
selectors only support equality, inequality, and set membership, and field
selectors are only supported for the `metadata.name` and `metadata.namespace`
fields and can use fewer operators than label selectors.  This results in a
diminished package searching experience for users.

## Terminology / Concepts
For clarification of any packaging concepts, please see the [packaging apis docs](https://carvel.dev/kapp-controller/docs/latest/packaging/).

## Proposal

### Goals and Non-goals
Goals:
* Increase searchability of packages

Non-goals:
* Introduce a way of searching that is not already exposed in common k8s clients

### Specification / Use Cases
In order to enhance searchability, we will extend the behavior of the field
selector parameter to allow for searches beyond the currently supported
operators and fields.

The new operators introduced will be regex searching for names and constraint
strings for versions.

Along with introducing these new operators, the aggregated server implementation
will also introduce support for the nonexistent fields, `name`, `version`, and
`repo`. These "virtual fields" will act as stand-ins for `spec.publicName`,
`spec.version`, and the yet-to-be-introduced repository field or annotation in
order to decouple search params from the underlying CR structure.

#### Use Case: Searching for Package by Name Using Existing Operators

Existing Operators:
- =   (equality)
- ==  (equality; same as =)
- !=  (not equal)

 The behavior of these operators will remain unchanged. For example,

`kubectl get packages --field-selector name="search-for-me"`

Will match all packages that have publicName equal to `search-for-me`.

#### Use Case: Searching for Package by Name Using Regex

To support this use case, we will add a qualifier to the beginning of the field
selector string. The `regex:` will instruct the server to interpret the
following string as a regex and search for package's with names that match. For
example,

`kubectl get package --field-selector 'name="regex:search-for-*"'`

will match any packages that have a name which matches the `search-for-*` regex.

#### Use Case: Searching for Package by Repo

This option will be introduced to search for packages based on what repository
they came from. This will be exposed using the field name repository. It will
only support the native operators.

`kubectl get package --field-selector repository="Carvel"`

Note: Repository information is not yet stored on packages

#### Use Case: Searching for Package by Version Using Constraint String

Version searching will support constraint strings to filter versions. More about
version constraints can be found [here](https://github.com/blang/semver#ranges)

`kubectl get packages --field-selector 'version=">=1.3.0"'`

Will match all packages that have version greater than or equal to `1.3.0`.

Because of the expressiveness of constraint strings, the supported operators
will be reduced to only equality operators (=, ==), removing the not equals. To
search by not equals, users should specify that in the constraint string.

#### Use Case: Searching for Package using Compound Logic

Kubectl currently support specifying multiple selectors by using a comma
separated list as the argument to the `--field-selector` option. Matching
default kubectl behavior, these selectors will be and'ed together to determine
the filter criteria.

`kubectl get package --field-selector 'name.regex="foobar.*",version.selector=">2.0.0"'`

### Other Approaches Considered

#### Using Label Selectors
The allowed character sets for label selectors is too limited for adding any
search functionality.

## Open Questions
1. Should field selector for name be name or publicName?
1. Is regex too complex an interface for users?
1. Should we support a way to or constraints?
1. Should version search field be renamed to something like version-constraint?

## Answered Questions

