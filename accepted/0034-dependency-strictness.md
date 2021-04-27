# Dependency strictness

## Summary

This RFC is a proposal to add a new opt-in installation mode.

This mode would be the missing ingredient to make workspaces truly scalable.

## Motivation

The introduction of workspaces in npm v7 brought the ability to split large codebases into smaller pieces of code with declared dependencies between them.

Several build tools took advantage of these declared dependencies to split the builds into multiple steps (one per workspace) which could be cached or run in parallel. This caching and parallelism usually improves significantly build performance.

The current default installation strategy does not properly communicates to NodeJS the information about the dependency graph.
NodeJS having an innacurate view of the dependency graph leads the build systems to wrongly skip build operations, leading to risk of bugs slipping through.

The installation mode proposed by this RFC would communicate accurately to NodeJS the dependency graph, allowing build tools to correctly reason about it.

## Naming

The proposed name for this mode is `pure-mode`, meaning "pure" as in "no side effects". This name comes from the parallel with functional programming; in functional programming, functions don't have side effects, in `pure-mode` workspaces have no side effects.

To make the discussions more efficient, we could also name the current default installation mode of npm. We will call this mode `hoisted-mode` as packages are shared accross workspaces by hoisting them to the root of the project.

## More on the problem

Packages and workspaces declare their dependencies to npm by using special fields in their package.json. The NodeJS runtime is unaware of these special fields, instead it uses a [module resolution algorithm](https://nodejs.org/api/modules.html) to determine the meaning of a module importing another module. npm bridges this gap by converting the declared dependency graph into a folder structure which makes sense to NodeJS. This operation is call `reification`.

When converting the dependency graph to a folder structure, the `hoisted-mode` is losing information. Multiple different dependency graphs can be converted to the same folder structure. Since the folder structure is all that NodeJS uses to resolve modules, NodeJS misses some information about dependencies. This loss of information is most frequently visible through the fact that workspaces are able to successfully import the dependencies declared by other workspaces. 

### Forgetting to declare dependencies

When workspaces can successfully use code of a package without having a dependency on it, people keep forgetting to declare their dependencies. This lead to situations where updating the dependencies of one workspace breaks a simingly unrelated workspace. 

It is worth noting that static code analysis tools can help significantly reduce the frequency of these mistakes.

### Duplication

The `hoisted-mode` is not always successful at sharing common dependencies. Conflicts in versions lead to packages being installed multiple times on disk. These conflicts become more frequent when the project scales. The frequency of these conflicts can be reduced by an effort to align dependencies' versions across the project.

These duplications come with the following cost:

- Performance degradation of the installation phase
- Inneficient disk usage
- Performance, memory cost at runtime as NodeJS sees duplicated modules as two different modules.

### Tools crawling the node_modules folders

Certain tools implement their own module resolution algorithm instead of the one provided by NodeJS. One of the motivation for a tool to implement its own resolution algorithm is that it can add more feature to it.

Crawling the node_modules folder is such a feature used by certain build tools. This feature means that the mere presence of a package in a `node_modules` folder will have an impact on the output of a workspace. This means that any modification of a project's `node_modules` folder is possibly a breaking change to every workspaces in this project, regardless of the dependency graph.

For example, the TypeScript compiler by default includes to its compilation *every* package in the folder `node_modules/@types` without any explicit import statement needed.

## Rationale

There seems to be a consensus in the community that [import-maps](https://github.com/WICG/import-maps) will become the best way to communicate the dependency graph to NodeJS. Because this standard is not yet implemented in NodeJS, npm needs to provide an alternative way to solve the problems stated earlier.

The goal is to provide an accurate dependency graph to NodeJS while still relying on NodeJS current module resolution algorithm.

We deem valuable to invest the time implementing the `pure-mode` now knowning that import-maps are coming because most of the implementation will be re-usable to implement a `import-maps-mode`.

There are already two compeeting solutions out there which implement a `pure-mode` (pnpm and yarn). We decided to choose the pnpm approach for the following reasons:

- *Works with current echosystem*, the pnpm `pure-mode` does not require any modification to NodeJS or to the various build tools.
- *Battle tested*, Microsoft has successfully used pnpm to manage large monorepo for year.
- *Recommended by NodeJS*, this approach is actually the recommended approach by [the NodeJS documentation](https://nodejs.org/api/modules.html#modules_addenda_package_manager_tips).

# Implementation

## Simple Explanation

The packages are laid out on disk with a flat structure in a folder called the "package store". The dependencies between packages will be expressed by creating symlinks between the various packages.

## How does it work?

This strategy is based on the fact that [Node.js module resolution algorithm](https://nodejs.org/api/modules.html#modules_all_together) follows symlinks and works with them exactly the same as if they were real folders.

Additionally, once a module is resolved, the resolution algorithm calls 'realpath()' on the result. This means that the resolution algorithm always returns a real path. This allows to setup an arbitrary complex dependency graph while making sure Node.js does not create more than one instance of a given module.

## Detailed Explanation

TODO: Describe implementation better.

- Packages are installed in folder called the store, this store is placed in the folder `node_modules/.npm`
- Each package is installed in a folder name containing a hash of the content of this package (and possibly of its dependencies).
- node_modules folders are created and populated by symlinks to the location of the dependency in the store.
- Each package can import itself 
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
   │              ┬
   │              │
   │              └───> B @ 1.0.0
   │
   └───> cat (workspace)
          ┬
          │
          └───> B @ 1.0.0

```

#### Installation on disk

```
  root /
   ┬
   │
   ├─> node_modules / .npm / ─┬─> A@1.0.0-21f95f7 / node_modules / A / ─┬─> [content of package A]
   │                          │                                         │
   │                          │                                         └─> node_modules / B ( symlink to ../../../../B@1.0.0-0d98566/node_modules/B )
   │                          │
   │                          └─> B@1.0.0-0d98566 / node_modules / B / ───> [content of package B]
   │
   └─> workspaces / ─┬─> foo / ─┬─> [content of workspace foo]
                     │          │
                     │          └─> node_modules / A ( symlink to ../../../node_modules/.npm/A@1.0.0-21f95f7/node_modules/A )
                     │
                     ├─> bar / ─┬─> [content of workspace bar]
                     │          │
                     │          └─> node_modules / A ( symlink to ../../../node_modules/.npm/A@1.0.0-21f95f7/node_modules/A )
                     │
                     └─> cat / ─┬─> [content of workspace cat]
                                │
                                └─> node_modules / B ( symlink to ../../../node_modules/.npm/B@1.0.0-0d98566/node_modules/B )

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
  root /
   ┬
   │
   ├─> node_modules / .npm / ─┬─> A@1.0.0+B@1.0.0-21f95f7 / node_modules / A / ─┬─> [ content of package A ]
   │                          │                                                 │
   │                          │                                                 └─> node_modules / B ( symlink to ../../../../B@1.0.0-0d98ab/node_modules/B )
   │                          │
   │                          ├─> A@1.0.0+B@2.0.0-66fe689 / node_modules / A / ─┬─> [ content of package A ]
   │                          │                                                 │
   │                          │                                                 └─> node_modules / B ( symlink to ../../../../B@2.0.0-a2ea56/node_modules/B )
   │                          │
   │                          ├─> B@1.0.0-0d98ab / node_modules / B / ───> [ content of package B (v1) ]
   │                          │
   │                          └─> B@2.0.0-a2ea56 / node_modules / B / ───> [ content of package B (v2) ]
   │
   └─> workspaces / ─┬─> foo / ─┬─> [ content of workspace foo ]
                     │          │
                     │          └───> node_modules / ─┬─> A ( symlink to ../../../node_modules/.npm/A@1.0.0+B@1.0.0-21f95f7/node_modules/A )
                     │                                │
                     │                                └─> B ( synlink to ../../../node_modules/.npm/B@1.0.0-0d98ab/node_modules/B )
                     │
                     └─> bar / ─┬─> [ content of workspace bar ]
                                │
                                └───> node_modules / ─┬─> A ( symlink to ../../../node_modules/.npm/A@1.0.0+B@2.0.0-66fe689/node_modules/A )
                                                      │
                                                      └─> B ( synlink to ../../../node_modules/.npm/B@2.0.0-a2ia56/node_modules/B )

```

## Performance

Compared to the current npm installation strategy, this proposal reduces package duplication, making the installation process faster. On a prototype, this installation strategy brought down the install time from 6 to 2 minutes on a large monorepo of 500+ workspaces.

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

  - answer: yes we should support them, circular symlink should not be an issue.

- Should we use symlinks or junctions on Windows? Both of them have drawbacks:

  - Junctions have to be representated by an absolute path, this means that junctions cannot be committed to git or packed into a package.
  - Symlinks can only be created in elevated shell [or when Windows is in "devloper mode"](https://blogs.windows.com/windowsdeveloper/2016/12/02/symlinks-windows-10/#LCiVBWTgQF5s7fmL.97).

  - answer: junctions by default and symlinks as opt-in
  
- Should the store folder be in the repo itself?
  - if yes, every package in the store will have access to the dependencies of the git repo, because they will have access to its node_module folder (`../../node_modules`)
  - if no, where? Should it be shared with other repositories installed on the system?
  
  - answer: in the first implementation the store will be in the repo, the dependencies of the repo itself will be concidered global because accessible by every packages. In a later stage, we can implement a store which is outside the repo and shared accross repos.
  
- How much community code will break when the system forbids access to undeclared dependencies? In other words, how much code needs to be fixed to work properly in strict mode?

- Should/can the content of the package store be read-only? This would enable faster incremental installation.

  - answer, not for the first implementation
