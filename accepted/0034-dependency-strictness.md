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

### Phantom dependency

A phantom dependency is an asymetric relationship between two packages.
These two statements are equivalent:

> `foo` has a phantom dependency on `bar`

> (`foo` does not have a direct dependency on `bar`) AND (`foo` imports `bar`)

### Strict-mode

Strict mode is an installation of the dependency graph which guarantees that phantom dependencies break at runtime.

## Motivation

Workspaces help scaling a large monorepo by offering a clean way to split code in logical units. The motivation behind this RFC is to take advantage of workspaces to scale build time sub-linearly.

Treating workspaces as pure functions enables to implement caching and incremental builds. Currently npm workspaces cannot be considered pure functions because a change in the dependencies of a workspace can affect the output of another workspace even if there is no dependency declared between these two workspaces.

Strict-mode is a necessary feature to be able to treat workspaces as pure functions.


## Rational

There seems to be a consencus in the community that [import-maps](https://github.com/WICG/import-maps) is key to the future of dependency management. Because this standard is not yet implemented in NodeJS npm cannot use it yet as a strategy to implement strit-mode. Instead, the strategy suggested in this RFC is to implement a solution that works with the current ecosystem by making pieces that can be reused later on to implement support for import-maps.

The goal is to have isolation between workspaces, so that one workspace's output cannot be impacted by the dependencies of an unrelated workspace. This means that we cannot install a dependency in the root of the repository as it would expose it to all the workspaces.
This means that packages need to be accessed only through the node_modules folder of each workspace. A naive implementation of this would create duplication which would lead to performance and disk usage issues with a cost outweighing the benefit of strictness.

This RFC suggests to install packages on a flat structure on disk and enable the imports from one to another by setting up symlinks between them.

This approach offers these benefits:

- *No package duplication*, a given version of a package is installed only once on disk, no matter how many workspaces depend on it.
- *Strict dependencies*, only the dependencies declared in package.json are installed as symlinks. Phantom dependencies are exposed by failing build and tests.
- *Work with standard runtimes*, while other alterntives rely on modifying the runtimes (eg. NodeJS), this solution is compatible with the assumptions of most build tools.
- *Battle tested*, this installation strategy has been battle tested by large private repos in Microsoft which have successfully used it through [pnpm](https://pnpm.io/).

## Detailed Explanation

## How does it work?

This strategy is based on the following characteristic of the [nodejs module resolution algorithm](https://nodejs.org/api/modules.html#modules_all_together):

When a package is being resolved, the resolution algorithm follows symlinks as if there were real folders. Once a module is resolved, the resolution algorithm calls 'realpath()' on the result. This means that the resolution algorithm always returns a real path. This allows to setup an arbitrary complex dependency graph while making sure nodejs does not create more than one instance of a given module..

## Implementation

### Simple example 

#### Dependency graph

```
- root
  - A@1.0.0
    - B@1.0.0
```

#### Installation on disk

```
- root
  - .store
    - A@1.0.0-21f95f7
      - node_modules
        - B -> ../../B@1.0.0-0d98569
    - B@1.0.0-0d98569
  - node_modules
    - A -> ../.store/a@1.0.0-21f95f7
```

### More complex example: peer dependencies

#### Dependency graph

```
- root
  - A@1.0.0
    - B@1.0.0
    - C@1.0.0
      - B@* (peerDependency)
      - D@1.0.0
	- B@2.0.0
	- C@1.0.0 (circular dependency)
```

#### Installation on disk

```
- root
  - .store
    - A@1.0.0-3511350ab
      - node_modules
        - B -> ../../B@1.0.0-326a99159dc
        - C -> ../../C@1.0.0+B@1.0.0-552c48dcf8b26
    - B@1.0.0-326a99159dc
    - B@2.0.0-e720142097
    - C@1.0.0+B@1.0.0-552c48dcf8b26
      - node_modules
        - B -> ../../B@1.0.0-326a99159dc
        - D => ../../D@1.0.0-1993b2f64e
    - C@1.0.0+B@2.0.0-9a6b39e1d
      - node_modules
        - B -> ../../B@2.0.0-e720142097
        - D => ../../D@1.0.0-1993b2f64e
    - D@1.0.0-1993b2f64e
      - node_modules
        - B -> ../../B@2.0.0-e720142097
        - C -> ../../C@1.0.0+B@2.0.0-9a6b39e1d
  - node_modules
    - A -> ../../.store/A@1.0.0-3511350ab
```


## Nice side effects

Implementing strictness this way provide the following advantages:

- It reduces package duplication compared to the npm default installation mode, making the installation process faster.
- Choosing wisely the hash used to store the packages in the store can lead to very fast incremental installation.
- Since workspaces are fully isolated from each other, a workspace will always produce the same output regardless of whether it is installed by a scoped install or by a full install.
- The package store can be re-used as is to implement support for import-maps, we simply need to not create the symlinks but generate the import-map file instead.

## Prior Art and Alternatives

### [pnpm](https://github.com/pnpm/pnpm)

Strict package manager.
This RFC is mostly inspired by pnpm.

### [ied](https://github.com/alexanderGugel/ied)

Strict package manager.
Similar to pnpm but unmaintained.

### [nix](https://nixos.wiki/wiki/Nix)

Functional package manager. Can work with npm packages (eg. [node2nix](https://github.com/svanderburg/node2nix) or [nixfromnpm](https://github.com/adnelson/nixfromnpm) )

### [yarn](https://yarnpkg.com/)

Strict package manager and project manager.

### [Import maps](https://github.com/WICG/import-maps)

Standard supported by [a few browsers](https://caniuse.com/import-maps) and [deno](https://deno.land/) which makes it possible to implement strictness.

Secure runtime for JavaScript and TypeScript.
Strictness can be implemented using [import maps](https://github.com/WICG/import-maps)

## Unresolved Questions and Bikeshedding

- npm supports circular dependencies, should strict mode support them? If so, would circular symlink be an issue?

- Should we use symlinks or junctions on Windows? Both of them have drawbacks:
    - Junctions have to be representated by an absolute path, this means that junctions cannot be committed to git or packed into a package.
    - Symlinks can only be created in elevated shell [or when Windows is in "devloper mode"](https://blogs.windows.com/windowsdeveloper/2016/12/02/symlinks-windows-10/#LCiVBWTgQF5s7fmL.97).

- Should the store folder be in the repo itself?
    - if yes, every package in the store will have access to the dependencies of the git repo, because they will have access to its node_module folder (`../../node_modules`)
    - if no, where? Should it be shared with other repositories installed on the system?
	
- How much community code will break when the system stops allowing phantom dependencies? In other words, how much code needs to be fixed to work properly in strict mode?

- Should/can the content of the package store be read-only? This would enable faster incremental installation.
