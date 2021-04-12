# Dependency strictness

## Summary

This RFC is a proposal to add a new opt-in installation mode called `strict-mode`.

Strict mode is an essential ingredient to fulfill the assumption that dependency-graph is an accurate description of the relationships between workspaces.

## Motivation

Monorepo-build-tools optimize the dev-workflows by assuming that the dependency graph is an accurate representation of the relationships between workspaces.

This assumption is a good approximation, but it is not always accurate. This inaccuracy can lead to a broken builds or releasing broken code.

What breaks this assumption is the fact that a dependency of one workspace can affect another workspace even if there is no dependency declared between these two workspaces. This is true because npm de-duplicates dependencies by installing them in the repo's root node_modules folder, making them accessible by every workspace.

## Rationale

There seems to be a consensus in the community that [import-maps](https://github.com/WICG/import-maps) is key to the future of dependency management. Because this standard is not yet implemented in NodeJS npm cannot use it yet as a strategy to implement strict-mode. Instead, the strategy suggested in this RFC is to implement a solution that works with the current ecosystem by making pieces that can be reused later-on to implement support for import-maps.

The goal is to have isolation between workspaces, so that one workspace's output cannot be impacted by the dependencies of an unrelated workspace. This means that we cannot install a dependency in the root of the repository as it would expose it to all the workspaces.
This means that packages need to be accessed only through the node_modules folder of each workspace. A naive implementation of this would create dependency duplication which would lead to performance and disk usage issues with a cost outweighing the benefit of strictness.

This RFC suggests to install packages on a flat structure on disk and enable the imports from one to another package by setting up symlinks between them.

This approach offers these benefits:

- *No package duplication*, a given version of a package is installed only once on disk, no matter how many workspaces depend on it.
- *Strict dependencies*, only the dependencies declared in package.json are installed as symlinks. Phantom dependencies are exposed by failing builds and tests.
- *Work with standard runtimes*, while other alternatives rely on modifying the runtimes (eg. NodeJS), this solution is compatible with the assumptions of most build tools.
- *Battle tested*, this installation strategy has been battle tested by large private repos in Microsoft which have successfully used it through [pnpm](https://pnpm.io/).

## Detailed Explanation

## How does it work?

This strategy is based on the following characteristic of the [Node.js module resolution algorithm](https://nodejs.org/api/modules.html#modules_all_together):

When a module is being resolved, the resolution algorithm follows symlinks as if there were real folders. Once a module is resolved, the resolution algorithm calls 'realpath()' on the result. This means that the resolution algorithm always returns a real path. This allows to setup an arbitrary complex dependency graph while making sure Node.js does not create more than one instance of a given module.

## Implementation

TODO: Describe implementation better.

- Packages are installed in folder called the store
- Each package is installed in a folder name containing a hash of the content of this package (and possibly of its dependencies).
- node_modules folders are created and populated by symlinks to the location of the dependency in the store.
- peer dependencies are resolved based on the parents and treated as normal dependencies. If a conflict occurs and two different parent provide a different version of a peer dependency, this will result in two different store entries for the same package (one for each resolved peer dependency version).

### Simple example 

#### Dependency graph

```
  root
   ┬
   │
   ├───> foo (workspace)
   │      ┬
   │      │
   │      └───> A @ 1.0.0
   │              ┬
   │              │
   │              └───> B @ 1.0.0
   │
   ├───> bar (workspace)
   │      ┬
   │      │
   │      └───> A @ 1.0.0
   │                ┬
   │                │
   │                └───> B @ 1.0.0
   │
   └───> fish (workspace)
          ┬
          │
          └───> B @ 1.0.0

```

#### Installation on disk

```
  root
   ┬
   │
   ├───> package_store
   │          ┬
   │          │
   │          ├───> A@1.0.0-21f95f7
   │          │        ┬
   │          │        │
   │          │        └───> node_modules
   │          │                   ┬
   │          │                   │
   │          │                   └───> B (symlink to ../../B@1.0.0-0d9856)
   │          │
   │          └───> B@1.0.0-0d9856
   │
   └───> workspaces
             ┬
             │
             ├───> foo
             │      ┬
             │      │
             │      └───> node_modules
             │                 ┬
             │                 │
             │                 └───> A (symlink to ../../package_store/A@1.0.0-21f95f7)
             │
             ├───> bar
             │      ┬
             │      │
             │      └───> node_modules
             │                 ┬
             │                 │
             │                 └───> A (symlink to ../../package_store/A@1.0.0-21f95f7)
             └───> fish
                    ┬
                    │
                    └───> node_modules
                               ┬
                               │
                               └───> B (symlink to ../../package_store/B@1.0.0-0d9856)


```

### More complex example: peer dependencies

#### Dependency graph

```
  root
   ┬
   │
   ├───> foo (workspace)
   │      ┬
   │      │
   │      ├───> A @ 1.0.0
   │      │       ┬
   │      │       │
   │      │       └───> B @ * (peer dependency)
   │      │
   │      └───> B @ 1.0.0
   │
   └───> bar (workspace)
          ┬
          │
          ├───> A @ 1.0.0
          │       ┬
          │       │
          │       └───> B @ * (peer dependency)
          │
          └───> B @ 2.0.0


```

#### Installation on disk

```
  root
   ┬
   │
   ├───> package_store
   │          ┬
   │          │
   │          ├───> A@1.0.0+B@1.0.0-21f95f7
   │          │        ┬
   │          │        │
   │          │        └───> node_modules
   │          │                   ┬
   │          │                   │
   │          │                   └───> B (symlink to ../../B@1.0.0-0d9856)
   │          │
   │          ├───> A@1.0.0+B@2.0.0-66fe689
   │          │        ┬
   │          │        │
   │          │        └───> node_modules
   │          │                   ┬
   │          │                   │
   │          │                   └───> B (symlink to ../../B@2.0.0-a2ea56)
   │          │
   │          ├───> B@1.0.0-0d9856
   │          │
   │          └───> B@2.0.0-a2ea56
   │
   └───> workspaces
             ┬
             │
             ├───> foo
             │      ┬
             │      │
             │      └───> node_modules
             │                 ┬
             │                 │
             │                 ├───> A (symlink to ../../package_store/A@1.0.0+B@1.0.0-21f95f7)
             │                 │
             │                 └───> B (synlink to ../../package_store/B@1.0.0-0d9856)
             │
             └───> bar
                    ┬
                    │
                    └───> node_modules
                               ┬
                               │
                               ├───> A (symlink to ../../package_store/A@1.0.0+B@2.0.0-66fe689)
                               │
                               └───> B (symlink to ../../package_store/B@2.0.0-a2ea56)
             
```


## Performance

Compared to the current npm installation strategy, this proposal reduces package duplication, making the installation process faster. On a prototype, this installation strategy brought down the install time from 5 to 1min on a large monorepo of 500+ workspaces.

The implementation can easily be applied to repos which don't use workspaces to get some perf benefit. Though it is unknown what this perf benefit would be in nono-workspace repo.

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
