# Dependency strictness

## Summary

This RFC is a proposal to add a new opt-in installation mode called `strict-mode`.

This mode will improve the predictability of the build systems by guaranteeing that the dependencies of a workspace cannot affect another workspace.

## Definitions

In the definitions below, the words `foo` and `bar` represent two different npm packages.

### Import

The two following sentences are equivalent.

> `foo` can import `bar`.

> `bar` is reachable from `foo` by the [modules resolution algorithm](https://nodejs.org/api/modules.html#modules_all_together).

We say that `foo` imports `bar` when the code in `foo` relies on its ability to import `bar`.

### Direct dependency

A direct dependency is an asymetrical relationship between two packages. These two statements are equivalent:

> `foo` has a direct dependency on `bar`.

> `bar` is listed in `foo`'s `package.json` as a dependency, a devDependency (in case of local packages) or a peerDependency.

The term "direct dependency" is sometimes used to refer to a package instead of a relationship (eg. `bar` is a direct dependency of `foo`). For the sake of clarity, this RFC will only use the term to refer to a relationship between two packages.

### Phantom dependency

A phantom dependency is an asymetric relationship between two packages.
These two statements are equivalent:

> `foo` has a phantom dependency on `bar`

> (`foo` does not have a direct dependency on `bar`) AND (`foo` imports `bar`)

### Strict-mode

Strict mode is an installation of the dependency graph which guarantees that phantom dependencies break at runtime.

## Motivation

Workspaces help scaling a large monorepo by offering a clean way to split code in logical units. The motivation behind this RFC is to take advantage of workspaces to scale build time.

Treating workspaces as pure functions enables to implement caching and incremental builds. Currently npm workspaces cannot be considered pure functions because a change the dependencies of a workspace can affect the output of another workspace even if not dependency is declared between these two workspaces.

Strict-mode is a necessary feature to be able to treat workspaces as pure functions.


## Detailed Explanation
//TODO

## Rationale and Alternatives
//TODO

## Implementation
//TODO

## Prior Art
//TODO

## Unresolved Questions and Bikeshedding
//TODO
