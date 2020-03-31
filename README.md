# Webpack Sass Loader + Yarn Workspaces Pathing issue

Sass-loader v8.0.2 has an issue when used in combination with yarn workspaces.

Sass-loader seems to resolve relative path imports (e.g. `./foo`) to be within the `packages/` directory, but resolve "absolute" imports that are located in one of its configured `includePaths` (`node_modules` in this repository) to be in the included directory (e.g. `@test/foo`).

That seems logical at first, but is a problem when those two styles of paths can resolve to the same module. Sass-loader treats them as different modules, instead of realizing that they're the same module referred to by a different path.

If a Sass module-A that is referred to both by a relative and absolute path configures another module-B, then sass-loader incorrectly throws an error saying that module-B is "already loaded, so it can't be configured using 'with'", even though module-B is only loaded a single time, within module-A

I think it's important to note that sass (dart-sass) does not have this problem. This error is only seen when wrapping sass with sass-loader.

## Structure of this repo

This repository is a monorepo managed using [yarn workspaces](https://classic.yarnpkg.com/en/docs/workspaces/). It contains three sub-repos named `@test/first`, `@test/second` and `@test/third`. Each sub-repo is dependent on the previous one; third depends on second, second depends on first.

A variable defined in `first` is overridden in `second`. A module within `second` then uses that variable via a relative path import to create a function.

`third` then tries to import both the overridden variable from `second`, as well as the function defined in `second`. This throws an error when using sass-loader through webpack, but not when using sass (dart-sass) directly.

## Seeing the error

The file at `packages/third/_index.scss` can be compiled using both sass (dart-sass) and using sass-loader through webpack (which is also configured to use the same sass compiler under the covers). The former succeeds, while the latter throws an error.

- To compile successfully using sass directly, run: `yarn use-sass`
  - This also outputs the correct CSS to `dist/bundle.css`
- To compile with an error using sass-loader through webpack, run: `yarn use-webpack`.

The error is:

```
ERROR in ./packages/third/_index.scss
Module build failed (from ./node_modules/sass-loader/dist/cjs.js):
SassError: This module was already loaded, so it can't be configured using "with".
  ┌──> node_modules/@test/second/_variables.scss
1 │ ┌ @forward '@test/first' with (
2 │ │   $primary: rebeccapurple !default,
3 │ │ );
  │ └─^ new load
  ╵
  ┌──> packages/second/_variables.scss
1 │ ┌ @forward '@test/first' with (
2 │ │   $primary: rebeccapurple !default,
3 │ │ );
  │ └─^ original load
  ╵
  node_modules/@test/second/_variables.scss 1:1  @use
  node_modules/@test/second/_functions.scss 1:1  @use
  /Users/zach/dev/test/sass-re-forward/packages/third/_index.scss 2:1                                      root stylesheet
 @ multi ./packages/third/_index.scss main[0]
```

> Notice: Both the new load and original load point to the same exact file, but the paths are different. One paths through the `packages/` directory and the other passes through the `node_modules` directory. Because of how yarn workspaces works, the file located in `node_modules/` is actually a simlinked version of the file located in `packages/`
