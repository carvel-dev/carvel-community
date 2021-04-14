---
title: "Rename images when copying bundles"
authors: [ "Joao Pereira <joaod@vmware.com>"]
status: "in-review"
approvers: [ "Dmitriy Kalinin <dkalinin@vmware.com>", "Eli Wrenn <ewrenn@vmware.com>", "John Ryan <ryanjo@vmware.com>" ]
---

# Rename images when copying bundles

## Problem Statement

When users use `imgpkg` to copy a bundle between repositories, `imgpkg` concentrates all the copied OCI Images into the
destination repository. This behavior is unexpected by some users and is cause the following problems

@braunsonm
> For instance if I transfer a bundle to a private registry, all of my images are stored as sha
> tags on the bundle image. This makes it impossible to run these images in any context other
> than Kubernetes such as docker run without first attaining the lock file and searching
> for the image you would like to run.

@jorgemoralespou
> Definitely the original tag is the value, as the goal is for trackability. I don't really think
> retagging would provide true value.
> Trackability is being able to know where it came from without much thinking, hence keeping the
> original tag is important.

@jorgemoralespou
> When authoring a bundle, the author can't take decisions on how the user will need/want to have
> the images organized, so he can only provide with a "meaningful" default but allow the user to
> override that default

@jimconner
> Working in heavily locked-down, audited, air-gapped environments with onerous security requirements
> means having to keep the security team happy, and that means working with an off-the-shelf container
> image vulnerability scanning solution which generates reports that the security team can understand.
> The same air-gapped environment gives us a private docker registry where we have full control over
> our namespaces and can avoid conflicts pretty easily.
> A mechnaism for preserving names when relocating from public to private registry is a highly
> desirable feature for me.

The above are some relevant quotes from https://github.com/vmware-tanzu/carvel-imgpkg/issues/60
and https://github.com/vmware-tanzu/carvel-kbld/issues/79 that bootstrapped the conversation that led to this proposal.

In this proposal, we will address some of these concerns by creating a mechanism that will allow the users that are
executing the copy between registries to have more control over the location where the images are copied to.

## Terminology / Concepts

![](OCITerminology.png)

- **OCI**: Open Container Initiative, [the official website](https://opencontainers.org/)

- **Image**: content stored within OCI registry
- **API**: Application Programming Interface, in this proposal case we will refer to YAML configuration as API
- **Bundle**: OCI Image that contains configuration and OCI Images, for more information
  the [original proposal](https://github.com/vmware-tanzu/carvel/blob/develop/proposals/imgpkg/001-bundles/README.md)
- **kebab-case**: Naming Case Style that consists on replacing spaces with `-`. (ex: `some name` -> `some-name`)
  . [More information](https://betterprogramming.pub/string-case-styles-camel-pascal-snake-and-kebab-case-981407998841)
- **snake_case**: Naming Case Style that consists on replacing spaces with `_`. (ex: `some name` -> `some_name`)
  . [More information](https://betterprogramming.pub/string-case-styles-camel-pascal-snake-and-kebab-case-981407998841)
- **MVP**: Minimum Viable Product
- **Tag**: Mutable identifier of a particular OCI Image
- **Digest**: Immutable identifier of a particular OCI Image, usually in the form `@sha256:...`

References:

- **Image Reference**: Location pointing at some OCI registry where image is stored
- **Full Qualified Image Reference**: The combination of the unique Registry, Namespace, Image Name and Tag or Digest.

Reference parts:

`corp.gcr.io/projects/project-a/bank-app:v1` explodes into:

- **Registry** (`corp.gcr.io`): Server that will be used to store and retrieve OCI Images
    - other examples: goharbor.io, gcr.io, index.docker.io
- **Namespace** (`projects/project-a`): Directory hierarchy that includes Image
- **Image Name** (`bank-app`): Name of an Image
- **Repository** (`projects/project-a/bank-app`): Full location of Image inside a Registry. Combination of Namespace and
  Image name.
- **Tag** (`v1`): Mutable identifier of a particular OCI Image

Some above references where inspired
by [this blog post](https://stevelasker.blog/2020/02/17/registry-namespace-repo-names/), because the OCI Specification
does not clearly define all the above concepts.

## Proposal

When a user copies a bundle between registries `imgpkg` copies all the OCI Images that are part of the bundle to the
same repository. This solution was adopted to protect copy functionality from the following problems:

- Multiple images that have the same Image Name but are not the same. Assuming we have 2 bundles one that contains the
  image `my.registry.io/controller@sha256:aaaa` and another that contains the
  image `other.registry.io/controller@sha256:bbbb`, when we copy both images to the registry `third.registry.io` they
  would be copied to `third.registry.io/controller@sha256:aaaa` and `third.registry.io/controller@sha256:bbbb`
  respectively. This can confuse because even though now they share the same Repository they are completely different
  Images from 2 completely different Source Codes. This might cause problems for Registry Administrators when they try
  to understand what each Repository contains.
- Finding if an Image is already present in the destination repository is complicated.
- Registries like `gcr.io` support paths in their Repositories while `hub.docker.io` does not. Copying an OCI Image
  from `gcr.io/my/specific/path/controller@sha256:aaaa` to `index.docker.io` is a challenge because we would lose
  information about the original OCI Image that can be significant.
- Copying OCI Images to different repositories can require different user or authentication. When copying an OCI Image
  to a particular Repository the user needs to have credentials to do so, in a scenario where the
  repository `my.registry.io/controller` was previously created by a different user, the current user might not have
  permission to copy the new OCI Image to that particular Repository.
- ECR requires users to create a Repository before pushing OCI
  Images. [reference](https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-push-ecr-image.html)

The adopted solution can solve the above problems, but it introduces some other problems, like the discoverability of
the OCI Images when they are being used in Kubernetes context or other contexts, as it is expressed in the quotes
provided in the [Problem Statement section](#Problem-Statement).

With this proposal, we will introduce a mechanism to enable the users of `imgpkg` to decide the Repository where each
OCI Image will be copied to. This will be achieved by providing the `copy` command with a Strategy that `imgpkg` will
follow to place OCI Images in the desired locations.

### Goals and Non-goals

#### Goals

- Propose a solution for the problems in the [Problem Statement section](#problem-statement)
- Maintain compatibility with the current implementation
- Provide the minimum amount of strategies to solve current problems
- Define API to provide the Strategy information to `imgpkg`
- Define provisioning information for this strategy

#### Non-goals

- Provide all possible combinations of strategies for copying OCI Images
- Provide strict implementation guide

### Specification / Use Cases

#### API

The following code block contains an example of the configuration YAML Document that would be provided to `imgpkg`

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: SingleRepository
overrides:
  - imageRepo: public-reg.io/simple-app
    destinationRepo: myname/simple-app
  - image: public-reg.io/exact-app@sha256:aaaaaaa
    destinationRepo: myname/exact-app
```

Fields:

- `apiVersion` (required; string) Version of this configuration.
- `kind` (required; `CopyConfig`) Type of configuration, this value should always be `CopyConfig`. Used to
  allow `imgpkg` to understand what type of configuration this document is defining
- `strategy` (required; `SingleRepository|RelativeToRegistryKeepNamespace`) This field will contain the Strategy that
  will be used by `imgpkg` the default value is `SingleRepository`. Check
  the [Copy Strategies section](#copy-strategies) for more information.
- `overrides` (optional; array of overrides) This is a list of overrides for the `strategy`. In this list, the user can
  define specific behavior for a particular set of OCI Images. Check the [Overrides section](#overrides) for more
  information on overrides.
- `mappingFunctionYttStar` (optional; `ytt` starlark function) This field contains an implementation of a `ytt` starlark
  function that will be responsible for creating a map between the OCI Images and the new locations

**Note:** `overrides` and `mappingFunctionYttStar` cannot be present at the same time in a configuration.

#### Copy Strategies

In this proposal, only 4 strategies will be defined, but in the future, more strategies can be made available. These
Strategies and Overrides will apply to all OCI Images that are contained in the Bundle that is being copied, this
includes all OCI Images from the current Bundle and Nested Bundles.

##### Copy all OCI Images to the same Repository as the Bundle

This strategy is the current strategy that `imgpkg` uses. Given this is the default strategy if this is the intended
behavior the users will not need to provide any configuration to the copy command.

The next code block is the minimum YAML configuration for this strategy and it will be applied to all the OCI Images
associated with the Bundle

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: SingleRepository
```

Given the Bundle contains a particular OCI Image that the user wants to copy to a different Repository or rename the
following configuration can be used

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: SingleRepository
overrides:
  - image: public-reg.io/exact-app@sha256:aaaaaaa
    destinationRepo: myname/exact-app
```

Given we provide the above configuration to the following command

`imgpkg copy -b public-reg.io/bundle@bbbbbb --to-repository my.private-registry.io/myname/bundle --config single-repo.yaml`

And the Bundle points to the OCI Image `public-reg.io/exact-app@sha256:aaaaaaa`, the above command will copy it
to `my.private-registry.io/myname/exact-app@sha256:aaaaaaa`. For other options of matching check
the [Overrides section](#overrides)

##### Copy all OCI Images to the new Registry but maintain the Original Repository

This strategy will copy each OCI Image to the target Registry and will store it in the same Repository as the OCI Image
originated.

The base configuration to enable this strategy is

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: RelativeToRegistryKeepNamespace
```

Given we provide the above configuration to the following command

`imgpkg copy -b public-reg.io/bundle@bbbbbb --to-repository my.private-registry.io/myname/bundle --config maintain-origin-repo.yaml`

All OCI Images will be copied to `my.private-registry.io/{Original Repository}`

**Example:**

- OCI Image from `origin.io/my-image@sha256:aaaa` will be copied to `destination.io/my-image@sha256:aaaa`
- OCI Image from `origin.io/some/path/my-image@sha256:aaaa` will be copied
  to `destination.io/some/path/my-image@sha256:aaaa`

This strategy also allows overrides, for other options of matching check the [Overrides section](#overrides)

##### Copy all OCI Images to the new Registry and all paths are relative to Bundle Namespace and keeps Image Namespace

This strategy will copy each OCI Image to the target Registry and will store each OCI Image under the same Namespace of
the Bundle, maintaining the Original Namespace of each Image.

The base configuration to enable this strategy is

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: RelativeToBundleKeepNamespace
```

Given we provide the above configuration to the following command

`imgpkg copy -b public-reg.io/bundle@bbbbbb --to-repository my.private-registry.io/myname/bundle --config maintain-origin-repo.yaml`

All OCI Images will be copied to `my.private-registry.io/{Destination Bundle Namespace}/{Original Repository}`

**Example using the above `imgpkg` command:**

- OCI Image from `origin.io/my-image@sha256:aaaa` will be copied to `my.private-registry.io/myname/my-image@sha256:aaaa`
- OCI Image from `origin.io/some/path/my-image@sha256:aaaa` will be copied
  to `my.private-registry.io/myname/some/path/my-image@sha256:aaaa`

This strategy also allows overrides, for other options of matching check the [Overrides section](#overrides)

##### Copy all OCI Images to the new Registry and all paths are relative to Bundle Namespace

This strategy will copy each OCI Image to the target Registry and will store each OCI Image under the same Namespace of
the Bundle.

The base configuration to enable this strategy is

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: RelativeToBundleNoNamespace
```

Given we provide the above configuration to the following command

`imgpkg copy -b public-reg.io/bundle@bbbbbb --to-repository my.private-registry.io/myname/bundle --config maintain-origin-repo.yaml`

All OCI Images will be copied to `my.private-registry.io/{Destination Bundle Namespace}/{Image Name}`

**Example using the above `imgpkg` command:**

- OCI Image from `origin.io/my-image@sha256:aaaa` will be copied to `my.private-registry.io/myname/my-image@sha256:aaaa`
- OCI Image from `origin.io/some/path/my-image@sha256:aaaa` will be copied
  to `my.private-registry.io/myname/my-image@sha256:aaaa`

This strategy also allows overrides, for other options of matching check the [Overrides section](#overrides)

#### Overrides

The `overrides` key in the configuration will contain a list of a Matcher and a Destination. All the OCI Images present
in the Bundle, and Nested Bundles will be matched against this list, but only the OCI Images the user wish to Override
should be present in this section. If an OCI Image do not Match any of these Overrides it will be copied based on the
strategy defined.

**Note:** `overrides` key cannot be present in the configuration if the key `mappingFunctionYttStar` is also defined

##### Assumptions

- In the beginning, a few possible overrides will be provided, eventually only the exact match one
- The user cannot copy different OCI Images to different Registries

##### Destination field in overrides

This field can be used in 2 different ways:

1. the value starts with a `/`

   When this is the case `destinationRepo: /here` the OCI Image will be copied to `registry.io/here`. `imgpkg` assumes
   that the OCI Image should be copied to the root of the Registry

1. the value **does not** start with `/`

   When the Bundle is being copied to `registry.io/place/bundle` then the OCI Image will be copied
   to `registry.io/place/img`. `imgpkg` assumes that the OCI Image should be copied to the same Registry and Namespace
   as the Bundle.

**Note:** In our documentation we should highlight this feature, since it might be a cause for problems

##### Exact match

This overrides will do an exact match of the OCI Image present in `.imgpkg/images.yml` and will copy the OCI Image to
the Repository that is provided in the `destinationRepo` field

Schema fields:

- `image` (required; string) Registry,Repository and Tag or SHA that will be used to do an exact match. This can be an
  OCI Image with a Tag or a SHA. (ex: `my.registry.io/img@sha256:aaaaaa` or `my.registry.io/img:my-tag`)
- `destinationRepo` (required; string) Namespace and Repository the OCI Image will be copied to. This field can only
  contain Namespace+Repository, where the Namespace is optional (not all registries support it)

Example of the override using a SHA

```yaml
image: public-reg.io/exact-app@sha256:aaaaaaa
destinationRepo: myname/exact-app
```

###### Exact Matching with Tag

When the user provides a Tag instead of the SHA for a particular OCI Image, `imgpkg` will do the following:

1. Try to match all OCI Images with the Registry+Repository
1. Look in ImagesLock file for the annotation `imgpkg.carvel.dev/image-tags`
1. Check if the Matched OCI Image in point 1 contains the Tag specified

Example

```yaml
kind: ImagesLock
images:
  - image: public-reg.io/exact-app@sha256:aaaaaa
    annotations:
      - imgpkg.carvel.dev/image-tags: [ "will-not-match" ]
  - image: public-reg.io/exact-app@sha256:bbbbb
    annotations:
      - imgpkg.carvel.dev/image-tags: [ "some-tag" ]
```

Example of override using a Tag

```yaml
image: public-reg.io/exact-app:some-tag
destinationRepo: myname/exact-app
```

In the above example the OCI Images that will be matched is `public-reg.io/exact-app@sha256:bbbbb`

##### Match by Registry and/or Repository

This Matcher is more flexible than the [Exact Match](#exact-match) where the only mandatory part provided is the
Repository. The Registry, Tag and Digest are optional.

This Matcher will match the Repository provided in the field `imageRepo` against all the OCI Images that are present in
the ImagesLock files for the Bundle and Nested Bundles. In order to match the Repository have to be equal, no wildcards
allowed.

Schema fields:

- `imageRepo` (required; string) matches values that follow image reference format ([registry]repository[:tag]\[@sha256:
  ...])
  and expects Repository portion to match (e.g. project/app).
- `destinationRepo` (required; string) Repository to where the OCI Image will be copied to. This field can only contain
  Namespace+Repository, where the Namespace is optional (not all registries support multiple Namespace nesting it)

Example of the override

```yaml
imageRepo: public-reg.io/simple-app
destinationRepo: /myname/simple-app
```

Result:

The OCI Image present in `public-reg.io/simple-app` will be copied to `my.private-registry.io/myname/simple-app`. The
example assumes that in the ImagesLock file there is 1 entry that will match `public-reg.io/simple-app@sha256:*`.

###### Match Registry and/or Repository with Tag

When the user provides a Tag instead of the SHA for a particular OCI Image, `imgpkg` will do the following:

1. Try to match all OCI Images with the [registry]repository
1. Look in ImagesLock file for the annotation `imgpkg.carvel.dev/image-tags`
1. Check if the Matched OCI Image in point 1 contains the Tag specified

Example

```yaml
kind: ImagesLock
images:
  - image: public-reg.io/exact-app@sha256:aaaaaa
    annotations:
      - imgpkg.carvel.dev/image-tags: [ "will-not-match" ]
  - image: public-reg.io/exact-app@sha256:bbbbb
    annotations:
      - imgpkg.carvel.dev/image-tags: [ "some-tag" ]
```

Example of override using a Tag

```yaml
imageRepo: exact-app:some-tag
destinationRepo: myname/exact-app
```

In the above example the OCI Images that will be matched is `public-reg.io/exact-app@sha256:bbbbb`

#### Order of matching

In the event that 2 Overrides match a particular OCI Image the first Override in the list takes precedence.

In the following example both overrides match the OCI Image, the first one is an exact match, and the second one is a
match on Registry+Repository. Nevertheless `imgpkg` will not take into account any further matches after the first one.
The OCI Image will be copied to `my.private-registry.io/myname/exact-app`.

```yaml
overrides:
  - image: public-reg.io/exact-app@sha256:aaaaaaa
    destinationRepo: myname/exact-app
  - imageRepo: public-reg.io/exact-app
    destinationRepo: image/will/not/be/copied/here
```

#### Advanced Mapping Scenarios

To enable more advanced matching integrating with `ytt` should be in the roadmap.

Example of using `ytt` starlark function that maps an OCI Image with a destination

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: RelativeToRegistryKeepNamespace
mappingFunctionYttStar: |
  def process(image):
    if regexMatch(image, '.*image1.*')
      return 'myname/exact-app'
    elif imageName(image) == 'other-image'
      return 'some-other-repo'
    end
    return imageName(image)
  end
```

**Note:** `mappingFunctionYttStar` key cannot be present in the configuration if the key `overrides` is also defined
**Note2:** The return value of this function will follow the rules defined in [here](#destination-field-in-overrides).

The mapping function will have to respect the following definition `process(string) -> string`.

To help users to create simpler mapping the following functions will be available:

##### regexMatch

Function that given a Regular Expression, will check if the provided string matches.

Definition: `def regexMatch(image, expression) -> boolean`

Parameters:

- `image` string with the Full Qualified Image Reference
- `expression` string with the Regular Expression that is being matched

Return:

- `True` if the `image` matches the Regular Expression
- `False` if the `image` does not match the regular expression

##### registry

Function will remove all information from an OCI Image except the registry location.

Definition: `def registry(image) -> string`

Parameters:

- `image` string with the Full Qualified Image Reference

Return: Registry URL

**Examples:**

- `registry('https://index.docker.io/u/ubuntu@sha256:aaaaaaaa')` will return `index.docker.io`
- `registry('gcr.io/some/deep/path/my-image@sha256:aaaaaaaa')` will return `gcr.io`
- `registry('gcr.io/some/deep/path/my-image:some-tag')` will return `gcr.io`
- `registry('https://my.private.registry:9999/some/deep/path/my-image:some-tag')` will return `my.private:registry:9999`

##### imageName

Function will remove all information from an OCI Image location and return the Image Name.

Definition: `def imageName(image) -> string`

Parameters:

- `image` string with the Full Qualified Image Reference

Return: Image Name

**Examples:**

- `imageName('https://index.docker.io/u/ubuntu@sha256:aaaaaaaa')` will return `ubuntu`
- `imageName('gcr.io/some/deep/path/my-image@sha256:aaaaaaaa')` will return `my-image`
- `imageName('gcr.io/some/deep/path/my-image:some-tag')` will return `my-image`

##### namespace

Function will remove the Registry information as well as the Image Name and return the Namespace information.

Definition: `def namespace(image) -> string`

Parameters:

- `image` string with the Full Qualified Image Reference

Return: Namespace

**Examples:**

- `namespace('https://index.docker.io/u/ubuntu@sha256:aaaaaaaa')` will return ``
- `namespace('gcr.io/some/deep/path/my-image@sha256:aaaaaaaa')` will return `some/deep/path`

##### matchTag

Function will check if a particular tag is associated with an OCI Image.

Definition: `def matchTag(image, tag) -> boolean`

Parameters:

- `image` string with the Full Qualified Image Reference
- `tag` string with the tag to check

Return:

- `True` if the `tag` can be found in the annotation `imgpkg.carvel.dev/image-tags` for the `image`
- `False` if the `tag` cannot be found in the annotation `imgpkg.carvel.dev/image-tags` for the `image`

#### Nested bundles considerations

If we assume the following bundle

```
other.registry.io/main-bundle
  - another.registry.io/img2
  - yet.another.registry.io/nested-bundle-2
    - world.io/img3
```

When a user tries to copy `other.registry.io/main-bundle` to `registry.acme.io/main-bundle` `imgpkg` will have to find
all the OCI Images that are part of this bundle. Before this proposal `imgpkg` could assume that any particular OCI
Image could be found in one of the following locations:

1. Bundle Repository we are copying from
1. Original location of the OCI Image (if the bundle was never copied)

After this feature is implemented `imgpkg` cannot assume the location of the OCI Image in question, since these Bundle
might or might not have been copied and changed the location where the OCI Images are stored. An example of this can be
found next

1. Create the nested-bundle-2 with the following ImagesLock file
   ```yaml
   apiVersion: imgpkg.carvel.dev/v1alpha1
   kind: ImagesLock
   images:
   - image: world.io/img3@sha256:aaaaaaaaaa
   ```

   `imgpkg push -b yet.another.registry.io/nested-bundle-2 -f folder1`

1. Copy `yet.another.registry.io/nested-bundle-2` to the Registry `other.registry.io`
   `imgpkg copy -b yet.another.registry.io/nested-bundle-2@sha256:bbbbbbbbbb --to-repo other.registry.io/nested-bundle-2`

1. Create `other.registry.io/main-bundle`

   Using the following ImagesLock
   ```yaml
   apiVersion: imgpkg.carvel.dev/v1alpha1
   kind: ImagesLock
   images:
   - image: other.registry.io/nested-bundle-2@sha256:bbbbbbbbbb
   ```

   `imgpkg push -b other.registry.io/main-bundle -f folder2`

When we prepare to copy this new bundle we need to consider that the OCI Image `world.io/img3@sha256:aaaaaaaaaa` can be
found in the following places:

- other.registry.io/main-bundle@sha256:aaaaaaaaaa
- other.registry.io/nested-bundle-2@sha256:aaaaaaaaaa
- yet.another.registry.io/nested-bundle-2@sha256:aaaaaaaaaa
- world.io/img3@sha256:aaaaaaaaaa
- Or any other place the OCI Image was copied to using a strategy in this proposal

To try to address this problem we can create a new OCI Image that contains 1 layer with 1 file at the root
called `images-locations.yml` that will have the following layout

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: ImagesLocation
images:
  - image: world.io/img3@sha256:aaaaaaaaaa
    repository: nested-bundle-2
```

Field explanation:

- `apiVersion` (required; string) Version of this configuration.
- `kind` (required; `ImagesLocation`) Type of configuration, this value should always be `ImagesLocation`. Used to
  allow `imgpkg` to understand what type of configuration this document is defining
- `images` (required; []array) Must contain on entry per OCI Images present in the ImagesLock file.
- `images[].image` (required; string) This value MUST match an OCI Image defined in ImagesLock file for the Bundle.
- `images[].repository` (required; string) Is the Repository where this OCI Image was copied to.

When copying a Bundle between Registries and/or Repositories this new OCI Image will be created in the destination
Repository and will be tagged with the tag `sha256-{Bundle SHA}.imgpkg.locations`. This is not a perfect solution since
Tags are mutable, but this will fix the problem for now.

This file will contain the Locations for all the OCI Images of all the Nested Bundles. In the example above the file
will contain 1 entry for each one of the following OCI Images:

```
other.registry.io/main-bundle
another.registry.io/img2
yet.another.registry.io/nested-bundle-2
world.io/img3
```

**Note:** When doing `pull` and `copy` operations, `imgpkg` should rely only on the `ImagesLocation` file that is
associated with the bundle that is being copied or pulled.

##### Caveat

Since `imgpkg` would be using tags to find the new location for the OCI Images of a particular Bundle there is no
guarantee that this tag is accurate. This should be considered a best effort to locate the OCI Images. When the tag is
not present in the expected location `imgpkg` will assume that the Bundle was copied with `SingleRepository` strategy
with no overrides.

##### Searching order for OCI Image

The priority order to find the Image location is:

1. Bundle Registry URL + `repository` field specified in the ImagesLocation configuration
1. Bundle Registry URL + Bundle Repository + @sha256:{Image SHA}
   **Note:** We need to keep this possible location for retro compatibility.
1. Location present in ImagesLock file

Assumptions:

- Bundle being pulled or copied is in the following location `registry.io/some/place/bundle`
- ImagesLock file contains 1 Image that point to `other.registry.io/img1@sha256:aaaaaaa`
- ImagesLocation looks like the following

```
registry: private.registry.io
images:
- image: other.registry.io/img1@sha256:aaaaaaa
   repository: some/other/place/img1
```

`imgpkg` would search the following places for the OCI Image in the order

1. registry.io/some/other/place/img1@sha256:aaaaaaa
1. registry.io/some/place/bundle@sha256:aaaaaaa
1. other.registry.io/img1@sha256:aaaaaaa

**Note:** `copy` and `pull` will have to be able to implement the above search

#### Registry replication considerations

Given that Registries can define replication of OCI Images this can be a challenge to `imgpkg` because it relies on the
Registry Location to find the Images. One example of this problem is:

1. User creates a Bundle in registry.io/bundle
1. User copies Bundle to private.registry.io/copied-bundle
1. Registry is being replicated to other.registry.io
1. User pulls Bundle from replicated Registry other.registry.io/copied-bundle

At this point in time the OCI Image can be in

- other.registry.io
- private.registry.io
- registry.io

`imgpkg` only is aware of `registry.io` (the original location of the OCI Image) and `other.registry.io` (the location
where the Bundle is being pulled)

**Note:** It is out of scope situations where not all OCI Images are relocated. `imgpkg` will try to do a best effort to
find all the OCI Images for a particular Bundle using the order defined [here](#searching-order-for-oci-image).

### Use Cases

#### As a User When I copy a Bundle I want to ensure tags present in ImagesLock file are preserved

**How might `imgpkg` help to accomplish this Use Case:**

- Add one new annotation `imgpkg.carvel.dev/image-tags` that supports a comma separated list of tags
- While copying OCI Images `imgpkg` will read this annotation and will tag the OCI Image with the provided tags

#### As a User When I inspect my Kubernetes Manifest I can identify the origin of the OCI Images that is running

**Goals**

- Provide a way for a tool that manages the Manifests to update the images with the more up to date location of an Image

**Non-goals**

- `imgpkg` be aware of Kubernetes Manifests

**How might `imgpkg` help to accomplish this Use Case:**

- Maintaining annotations provided in ImagesLock
- Allow the user to customize the Destination Repository when copying OCI Images to match a naming convention that could
  help the user identify the original OCI Image using only the Repository name
- Ensure that tags present in the ImagesLock folder are maintained when OCI Images are copied

#### As a User When I can copy a Bundle to my local Registry to run one of the OCI Images

**Goals**

- Provide a way for users to know how to find the OCI Image they want to run

**Non-goals**

- `imgpkg` understand how to run images

**How might `imgpkg` help to accomplish this Use Case:**

- Allow the user to customize the Destination Repository when copying OCI Images to match a naming convention that could
  help the user identify the original OCI Image using only the Repository name
- Ensure that tags present in the ImagesLock folder are maintained when OCI Images are copied

#### As a User When I look at the OCI Images in a Registry I am able to easily understand what each OCI Images is used for

**Goals**

- Provide a way for users to differentiate OCI Image by looking at the Repository name

**How might `imgpkg` help to accomplish this Use Case:**

- Allow the user to customize the Destination Repository when copying OCI Images to match a naming convention that could
  help the user identify the original OCI Image using only the Repository name
- Ensure that tags present in the ImagesLock folder are maintained when OCI Images are copied

#### As a User When I have a Bundle with a big number of OCI Images I want to be able to create the CopyConfig file easily

**How might `imgpkg` help to accomplish this Use Case:**

- New command that generates a configuration file from a Bundle

  `imgpkg tools generate copy-config -b registry.io/bundle@sha256:aaaaaa -o /tmp/copy-config.yml`

#### As a User When I am creating my overrides I want to see if they are correct before copying

**How might `imgpkg` help to accomplish this Use Case:**

- New flag on the copy command would provide this information

  `imgpkg copy -b registry.io/bundle@sha256:aaaaaa -c strategy.yaml --to-repository other.registry.io/bundle --dry-run`

This command should provide the following output

```
Strategy: SingleRepository

Copying with rename will occur as following
registry.io/bundle@sha256:113123 -> other.reg.io/copied-bundle@sha256:113123
registry.io/img1@sha256:5555555 -> other.reg.io/copied-bundle@sha256:5555555
index.docker.io/img2@sha256:666666 -> other.reg.io/copied-bundle@sha256:666666
index.docker.io/img3@sha256:777777 -> other.reg.io/overrided-strategy@sha256:777777
```

#### As a User When I am using a Replicated Registry I want to use OCI Images stored in the Replicated Registry

**Goals**

- Ensure OCI Images that are used on copy and pull are the ones the User Expects
- When pull or copy the OCI Images will come from the source Registry or the Original location that is provided in the
  ImagesLock file

**How might `imgpkg` help to accomplish this Use Case:**

- Ability to find the OCI Images that are part of the Bundle in the Replicated Registry

### Implementation breakdown

This is quite a big proposal that can take some time to implement. In this section we try to split the current work
needed into multiple phases that could provide users with the most important parts of this feature as soon as possible.

#### Phase 0 - Implement Tags preservation

After this phase is complete the users would be able to:

1. `imgpkg` will tag OCI Images when copying a Bundle
1. `kbld` fill the new annotation with the tag values

   This need to happen in operations:
    - When `kbld` is creating a new OCI
      Image, [reference](https://carvel.dev/kbld/docs/latest/config/#imagedestinations)
    - When `kbld` is resolving OCI Images from configuration

New annotation:

Create Annotation `imgpkg.carvel.dev/image-tags` that will support a comma separated list of tags that should be applied
to the OCI Images when they are being copied between Registries

#### Phase 1 - Implementation of strategies (MVP-1)

After this phase is complete the users would be able to:

1. `imgpkg` will tag OCI Images when copying a Bundle
1. `kbld` fill the new annotation with the tag values
1. Copy OCI Images between registries choosing the strategy that best suites their needs.
1. When user is copying Bundles and do not select a strategy, copy will use `SingleRepository` as the default strategy.
1. Pull a Bundle to disk and see the correct locations for all OCI Images in the ImagesLock files.

New Flags:

- Copy command
    - `-c filename` and `--configuration filename` provide the configuration to select a strategy
    - `--dry-run` outputs the copy strategy and the new OCI Images location without executing the copy

API Implementation:
At this point in time the user can only select a Copy Strategy that will apply to all OCI Images in the Bundle

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: (SingleRepository|RelativeToRegistryKeepNamespace)
```

#### Phase 2 - Inclusion of simple overrides (MVP-2)

After this phase is complete the users would be able to:

1. `imgpkg` will tag OCI Images when copying a Bundle
1. `kbld` fill the new annotation with the tag values
1. Copy OCI Images between registries choosing the strategy that best suites their needs.
1. When user is copying Bundles and do not select a strategy, copy will use `SingleRepository` as the default strategy.
1. Pull a Bundle to disk and see the correct locations for all OCI Images in the Images Lock files.
1. Create Overrides for the selected strategy

API Implementation:
At this point in time the user can select a Copy Strategy and Overrides that will apply to all OCI Images in the Bundle

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: (SingleRepository|RelativeToRegistryKeepNamespace)
overrides:
  - imageRepo: public-reg.io/simple-app
    destinationRepo: myname/simple-app
  - image: public-reg.io/exact-app@sha256:aaaaaaa
    destinationRepo: myname/exact-app
```

Overrides available:

- `image` matches exactly the OCI Image location
- `imagesRepo` matches exactly the Registry and Repository

#### Phase 3 - Add more complex mapping functionality (Final)

After this phase is complete the users would be able to:

1. `imgpkg` will tag OCI Images when copying a Bundle
1. `kbld` fill the new annotation with the tag values
1. Copy OCI Images between registries choosing the strategy that best suites their needs.
1. When user is copying Bundles and do not select a strategy, copy will use `SingleRepository` as the default strategy.
1. Pull a Bundle to disk and see the correct locations for all OCI Images in the Images Lock files.
1. Create Overrides for the selected strategy
1. Create a mapping function using `ytt` starlark

API Implementation:
At this point in time the user can select a Copy Strategy that will apply to all OCI Images in the Bundle. To create
Overrides for the selected strategy the user will have to choose between Overrides or a Mapping function.

#### Phase 4 - Add Extra Strategies (Optional)

This phase can be done in parallel with Phase 3, and is Optional(not required to declare this track of work completed).

After this phase is complete the users would be able to:

1. `imgpkg` will tag OCI Images when copying a Bundle
1. `kbld` fill the new annotation with the tag values
1. Copy OCI Images between registries choosing the strategy that best suites their needs.
1. When user is copying Bundles and do not select a strategy, copy will use `SingleRepository` as the default strategy.
1. Pull a Bundle to disk and see the correct locations for all OCI Images in the Images Lock files.
1. Create Overrides for the selected strategy
1. Create a mapping function using `ytt` starlark
1. Copy Images using 2 new strategies `RelativeToBundleNoNamespace` and `RelativeToBundleKeepNamespace`

API Implementation:
At this point in time the user can select a Copy Strategy and Overrides that will apply to all OCI Images in the Bundle

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: (SingleRepository|RelativeToBundleNoNamespace|RelativeToBundleKeepNamespace|RelativeToRegistryKeepNamespace)
overrides:
  - imageRepo: public-reg.io/simple-app
    destinationRepo: myname/simple-app
  - image: public-reg.io/exact-app@sha256:aaaaaaa
    destinationRepo: myname/exact-app
```

### Other Approaches Considered

- Keep current behavior but provide more insight into the OCI Images inside a Bundle

  The proposal was to create a new command that would read a bundle and would provide the user with the Original
  Registry and Repository of a particular OCI Image as well as the new location. Something similar
  to `imgpkg images -b my.private-registry.io/bundle-with-simple-app`.

  Pros:
    - Command could provide all the needed information about the new OCI Image location
    - From the output of the command would be easy for the user to execute `docker run` or `podman run` with the OCI
      Image they were looking for

  Cons:
    - Kubernetes Manifest would still be hard to understand where a particular OCI Image is from
    - Require `imgpkg` to decode this information from the Bundle

  In summary, we believe that this could help the users find the OCI Images and provide other helpful information about
  the Bundle. The first con is one of the problems that this proposal is trying to solve, because of that we decided
  that this would not be a good approach.

- Tagging with origin Registry and Repository each OCI Image and copy OCI Images to the Bundle Repository

  The idea behind this approach would be to continue with the current behavior but whenever an OCI Image is copied it
  would be tagged with a kebab-case or snake_case Tag that would represent the original Registry and Repository(ex:
  origin: `gcr.io/some/location/repo@sha256:aaa` would be tagged with `gcr-io-some-location-repo`).

  Pros:
    - No major change required to `imgpkg`
    - Easier to find if the user can read all tags associated with a particular Repository

  Cons:
    - If the Digest is not included in Tag, the Tag might be overwritten if we copy a newer version of the bundle
    - Is easier to find the original location using the tag, but when a user tries to run the OCI image
      using `docker run` or `podman run` the line would look something like
      this `docker run my.destination.io/bundle:gcr-io-some-path-to-image-my-image-sha256-aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa`
    - Kubernetes Manifest would still be hard to understand where a particular OCI Image is from

  In summary, we believe this could help find the OCI Images in question by reading the tag. Nevertheless, these tags
  would not be provided to Kubernetes Manifests. The cons for this approach do not help to solve the problem statement
  this proposal is addressing.

## Open Questions

- Can we wait for the Artifact OCI spec change to be complete?

    - If we waited what would be the drawbacks?
    - Do we need to wait for it to be able to implement this feature?

- Would it be helpful to implement some intermediate steps while we build this feature?

- Can we in a first approach just provide a command-line flag to change strategies?
    - Would this reduce the total amount of complexity of this feature?

- Can we save the SHA of the OCI Image in the ImagesLocation file instead of having the full Location as the key?

  Something to consider while doing the implementation. Nevertheless if we decide to use the SHA we should change the
  field `images[].image` to `images[].sha`.

## Answered Questions

- How will `imgpkg` know where to find the OCI Images after they are copied?

  The problem can happen if a user copies a Bundles from Registry A -> Registry B and afterward a different user wants
  to copy the Bundle from Registry B to Registry C, how might `imgpkg` know where to get the Images from?

  **Answer:** Created a new OCI Image that will store this information, for more details
  check [Nested bundles considerations section](#nested-bundles-considerations)
- What is the implication for Nested Bundles?

  The same problem as the above questions, but we need to understand if there is any drawback on the Nested Bundles case

  **Answer:** The main drawback is the need to retain information to about where an OCI Image can be found, for more
  details check [Nested bundles considerations section](#nested-bundles-considerations)

- Will we need to keep some records about the location of the OCI Images after the first copy?

    - If we need how might we save that in a place that `imgpkg` can easily retrieve in the future?

      **Answer:** we need to save this information. In case we do not do it we will have to assume we do not know where
      the OCI Images can be found. This will not be a problem if the OCI Images are copied with the Strategy to maintain
      all the OCI Images inside the same Repository where the Bundles are stored
    - Maybe a new OCI Image in the Bundle Repository with a TAG that we know about?

      **Answer:** This was the approach selected but this have a big caveat, for more details
      check [Nested bundles considerations section](#nested-bundles-considerations)

- What is the good initial chunk of work that would make more sense?

  **Answer:** Created [the section Implementation Breakdown](#implementation-breakdown) to try to split the work into
  manageable phases

- Should copying OCI Images to different Registries be allowed?

  My initial inclination is to do not allow and even raise an error if the Registries do not match.

  **Answer:** We should not allow different Registries in the Overrides section

- Should `imgpkg` try to get OCI Images missing on a Replicated Registry from the Origin of the Replicated Registry

  Example: `replica.registry.io` is a replica of `registry.io`
  If the Bundle is present in `replica.registry.io` but one of the OCI Images is not, should `imgpkg` try to get the
  Image from `registry.io`

  **Answer:** No, unless the OCI Image original location (ImagesLock) is in `registry.io`