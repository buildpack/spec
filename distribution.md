# Distribution Specification

This document specifies the artifact format, delivery mechanism, and order resolution process for buildpacks.


## Table of Contents

1. [Order Resolution](#order-resolution)
2. [Artifact Format](#artifact-format)
   1. [Buildpack](#buildpack)
   2. [Buildpackage](#buildpackage)


## Artifact Format

### Buildpack

A buildpack MUST contain a [`buildpack.toml`](buildpack.md#buildpacktoml-toml) file at its root directory.

### Buildpackage

A buildpackage MUST exist as either an OCI image on an image registry, an OCI image in a Docker daemon, or a `.cnb` file.

A `.cnb` file MUST be an uncompressed tar archive containing an OCI image. Its file name SHOULD end in `.cnb`.

Each FS layer blob in the buildpackage MUST contain a single buildpack at the following file path:

```
/cnb/buildpacks/<buildpack ID>/<buildpack version>/
```

After all buildpack layer blobs a buildpackage may contain `tar.gz` files at the following file path
```
/cnb/artifacts/<artifact SHA256>
```
( Top level sha gives us artifact de-duplication over buildpackage, but would make providing dependencies more expensive )
Alternative is a combination of two files:
```
/cnb/artifacts/<artifact SHA256>
/cnb/artifacts/<artifact SHA256>.toml -> lists buildpack-id & versions that contain this artifact
                                         [[buildpack]]
                                            id = io.cloudfoundry.nodejs
                                            version = 1.2.3
                                            
```

A buildpack ID, buildpack version, and at least one stack MUST be provided in the OCI image config as a Label.

Label: `io.buildpacks.buildpackage.metadata`
```json
{
  "id": "<entrypoint buildpack ID>",
  "version": "<entrypoint buildpack version>",
  "offline": true,
  "stacks": [
    {
      "id": "<stack ID>",
      "mixins": ["<mixin name>"]
    }
  ]
}
```

The buildpack ID and version MUST match a buildpack provided by a layer blob.

For a buildpackage to be valid, each `buildpack.toml` describing a buildpack implementation MUST have all listed stacks.

For each listed stack, all associated buildpacks MUST be a candidate for detection when the entrypoint buildpack ID and version are selected.

Each stack ID MUST only be present once.
For a given stack, the `mixins` list MUST enumerate mixins such that no included buildpacks are missing a mixin for the stack.

Fewer stack entries as well as additional mixins for a stack entry MAY be specified.

We define the following terms:
  `child buildpacks`: all buildpacks referenced from a `buildpack.toml` (in the order section) &

For a buildpackage to be offline the following conditions must be met:
1.) All buildpacks referenced in the tree of buildpacks starting with the root buildpack at the `entrypoint buildpack's` ( need some specification describing how this is built), must be an FS layer blob in the buildpackage.
1.) All additional downloadable artifacts are included as artifacts of the form `/cnb/artifacts/<artifact SHA256>`

So that the lifecycle can `detect`, `analize`, `build` and `export` on a local offline registry.


Open questions:
- Does the `build` phase of detect now need an extra argument to point to a mounted volume of the artifacts at `/cnb/artifacts/<artifact SHA256>`?

- Should we just take this one step further and make artifacts images as well? 
