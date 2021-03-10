---
title: "PackageVersions CR"
authors: [ "Eli Wrenn ewrenn@vmware.com" ]
status: "draft"
approvers: [ ]
---

# Add a PackageVersions CR

## Problem Statement

In the current alpha version of the packaging APIs, the experience for listing
available packages degrades quickly as the number of available packages
increases. One major cause of this degradation is the fact that, when
listing packages in a table format using kubectl, there will be an entry for
every version of every package. On large clusters, this could result in
thousands of entries being listed, reducing a users ability to easily parse the
packages available to them. For example, there could be multiple screens worth of the same
package just different versions, making it harder for a user to find an
unrelated package.

## Terminology / Concepts
For clarification of any packaging concepts, please see the [packaging apis docs](https://carvel.dev/kapp-controller/docs/latest/packaging/).

## Proposal

### Goals and Non-goals
Goals
* Reduce the amount of information a user needs to parse when trying to list
  available packages.


### Specification / Use Cases
A new PackageVersions CR will be introduced in to the packaging APIs. The new CR
will have the following structure:

```yaml
apiVersion: package.carvel.dev/v1alpha1
kind: PackageVersions
metadata:
   name: <package-name>
   # non-namespaced since packages aren't namespaced
spec:
   availableVersions:
   - 1.0.0
   - 1.1.0
   ...
```

This CR won't be stored in any persistent storage and will instead be created at
request time based on the current list of available packages in the system. It
will also be served by the packages aggregated api server. Additionally, the
only supported API methods for the CR will be GET and LIST.

The table output for this CR will show the package name as well as a small
preview of available versions. To see the full list of available versions, users
can describe the resource, or use the various output formats available via
kubectl get.

As part of the introduction of this CR, the LIST method for Package CRs will
be removed and replaced with an error message suggesting the use of the
PackageVersions list.

Table Format:
```
Name              Available Versions
<package-name>    1.0.0, 1.1.0
<package-name-2>  1.0.0, 1.1.0, 1.2.0, ... # if there are too many versions cut the list short
```

### Other Approaches Considered

## Open Questions

## Answered Questions
