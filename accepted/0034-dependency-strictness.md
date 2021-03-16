# Dependency strictness

## Summary

This RFC is a proposal to add new feature would add a new opt-in mode called `strict-mode`.

This mode will improve the predictability of the build systems by guaranteeing that the dependencies of a workspace cannot affect the other workspaces.

## Definitions

In the definitions below, the words `foo` and `bar` represent two different npm packages.

### Import

The two following sentences are equivalent.

> `bar` is reachable from `foo` by the [modules resolution algorithm](https://nodejs.org/api/modules.html#modules_all_together).

> `foo` can import `bar`.

We say that `foo` imports `bar` when the code in `foo` relies on its ability to import `bar`.

### Direct dependency

A direct dependency is an asymetrical relationship between two packages. These two statements are equivalent:

> `bar` is listed in `foo`'s `package.json` as a dependency, a devDependency (in case of local packages) or a peerDependency.

> `foo` has a direct dependency on `bar`.

The term "direct dependency" is sometimes used to refer to a package instead of a relationship (eg. `bar` is a direct dependency of `foo`). For the sake of clarity, this RFC will only use the term to refer to a relationship between two packages.

### Phantom dependency

A phantom dependency is an asymetric relationship between two packages.
These two statements are equivalent:

> `foo` has a phantom dependency on `bar`

> (`foo` does not have a direct dependency on `bar`) AND (`foo` imports `bar`)

### Strict-mode

Strict mode is an installation of the dependency graph which guarantees that phantom dependencies break at runtime.

## Motivation

Installing a repository in strict mode make it possible to see each workspace as a pure function of its code and its dependencies. With the guarantee that a workspace cannot depend on a side effect from another workspace, we can implement more stable and optimized build system.

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
