# Metadata-free lock validation

The `lock-without-metadata` preview writes lockfile version 1, revision 4. Workspace members,
registry packages, and local sources omit `[package.metadata]`; their resolved dependency edges
remain in the lockfile. HTTP(S) and Git dependencies retain declaration metadata because it cannot
always be reconstructed offline; dependencies without declarations may omit the table. Git metadata
is copied from the resolver output because ordinary immutable Git entries do not otherwise retain
it.

A lock is current when its packages, sources, versions, edges, extras, groups, and marker coverage
represent the current declarations. Changes that alter any observable graph or source assignment
make it stale. Adding, removing, or renaming an empty extra or group without changing the serialized
graph may be unobservable: detecting it would require the metadata this format omits. Declared empty
extras must nevertheless remain selectable with `--frozen`, and recorded extra and group selections
retain their existing frozen behavior.

## Declaration ownership

Workspace projects contribute production requirements, optional dependencies, and dependency groups.
Projectless workspace roots contribute root-owned groups; PEP 723 scripts contribute inline
requirements. Entries in `[tool.uv.sources]` belong to their project and dependency context; an old
locked edge cannot authorize a removed source.

Production, extra, and group edges are reconstructed independently. Name-only registry requirements
select a compatible locked package; version specifiers constrain its version. A requested extra
selects both the base distribution and its extra. Validation includes previously recorded sections,
so removing requirements or changing selections invalidates the lock.

Global overrides replace matching declarations before edge reconstruction. Package-scoped overrides
apply only to the specified package and optional version, not to that package's dependency groups.
Exclusions remove matching requirements within their declared scope. Constraints can select or
restrict an already reachable package but cannot introduce a package or revive an inactive source.

## Environments and conflicts

A requirement applies where its platform and Python markers overlap the parent package's reachable
environment. Fork markers restrict each locked candidate to the branches where it was selected. The
reconstructed candidates must cover the complete required environment: matching one fork does not
justify an uncovered branch. Markers are simplified under the lockfile's `requires-python`; trees
with different interned identities are equivalent only when each implies the other.

Extras and dependency groups contribute their own activation contexts. Mutually exclusive projects,
extras, and groups may select different package versions or sources within separate conflict worlds.
A source from one incompatible world cannot satisfy another, while compatible overlapping conflict
sets and a project conflicting with its own extra must retain their valid selections. Propagating
active package and extra environments avoids expanding independent conflicts into a Cartesian
product.

## Source identity and provenance

Locked candidates must match their refreshed declaration's package name, applicable version, and
source identity. Registry selections preserve their index; HTTP(S) selections compare normalized
URLs and subdirectories; local selections compare normalized paths and editable or virtual status;
Git selections compare the repository, requested reference, subdirectory or path, and pinned commit
when specified.

Validation computes which locked packages and extras remain reachable under current declarations,
forks, and conflicts. It collects direct sources from reachable workspace projects, groups, local
packages, retained remote metadata, global overrides, and active constraints. A bare registry-shaped
requirement can reuse a direct source only when an authorized declaration selects that source.

Direct selections can be shared across disjoint platform environments, but their conflict context
remains isolated. If a consumer overlaps both sides of a conditional source, that source cannot
authorize the broader environment. Inactive extras, excluded requirements, removed declarations, and
unreachable constraints never provide source evidence.

Local directories, editable projects, and workspace members are refreshed from `pyproject.toml`
where static metadata is available. Dynamic or backend-only projects may still require generating
local package metadata. A local archive is inspected only after a reachable declaration selects its
exact locked source. Removed or inactive archives are never opened simply because stale locked edges
mention them.

Remote HTTP(S) and Git providers supply transitive direct declarations from retained lockfile
metadata without fetching the artifact or reading a Git checkout. URL matching ignores credentials
while preserving requested extras, recursive extras, and active groups. A pinned Git dependency
remains valid if its original local repository disappears.

Configured `[[tool.uv.dependency-metadata]]` overrides package declarations, including declarations
from reachable local and registry packages. Configured metadata for a registry package cannot
authorize a direct source: a first-party declaration, global override, or active constraint must
independently select that source. These checks require project configuration; `--no-config` would
discard the configured metadata.

## Offline freshness checks

An unchanged, supported lock validates without network access, cached artifacts, or falling back to
dependency resolution:

```console
$ uv lock --preview-features lock-without-metadata --check --offline --no-cache
```

The validator may still read local source trees and archives or generate metadata for a selected
dynamic local package. `--offline` and `--no-cache` do not disable local build-backend execution,
and `--no-build` alone does not protect editable projects. For untrusted inputs, use `--no-build`
and exclude editable projects and any dynamic or local source that requires a build backend.
