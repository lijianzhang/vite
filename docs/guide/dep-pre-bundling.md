# Dependency Pre-Bundling

When you run `vite` for the first time, you may notice this message:

```
Optimizable dependencies detected:
react, react-dom
Pre-bundling them to speed up dev server page load...
(this will be run only when your dependencies have changed)
```

## The Why

This is Vite performing what we call "dependency pre-bundling". This process serves two purposes:

1. **CommonJS and UMD compatibility:** During development, Vite's dev serves all code as native ESM. Therefore, Vite must convert dependencies that are shipped as CommonJS or UMD into ESM first.

   When converting CommonJS dependencies, Vite performs smart import analysis so that named imports to CommonJS modules will work as expected even if the exports are dynamically assigned (e.g. React):

   ```js
   // works as expected
   import React, { useState } from 'react'
   ```

2. **Performance:** Vite converts ESM dependencies with many internal modules into a single module to improve subsequent page load performance.

   Some packages ship their ES modules builds as many separate files importing one another. For example, [`lodash-es` has over 600 internal modules](https://unpkg.com/browse/lodash-es/)! When we do `import { debounce } from 'lodash-es'`, the browser fires off 600+ HTTP requests at the same time! Even though the server has no problem handling them, the large amount of requests create a network congestion on the browser side, causing the page to load noticeably slower.

   By pre-bundling `lodash-es` into a single module, we now only need one HTTP request instead!

## Automatic Dependency Discovery

If an existing cache is not found, Vite will crawl your source code and automatically discover dependency imports (i.e. "bare imports" that expect to be resolved from `node_modules`) and use these found imports as entry points for the pre-bundle. The pre-bundling is performed with `esbuild` so it's typically very fast.

After the server has already started, if a new dependency import is encountered that isn't already in the cache, Vite will re-run the dep bundling process and reload the page.

## Dependency Compatibility

While Vite tries its best to accommodate non-ESM dependencies, there are going to be some dependencies that won't work out of the box. The most common types are those that import Node.js built-in modules (e.g. `os` or `path`) and expect the bundler to automatically shim them. These packages are typically written assuming all users will be consuming it with `webpack`, but such usage does not make sense when targeting browser environments.

When using Vite, it is strongly recommended to always prefer dependencies that provide ESM formats. This will make your build faster, and results in smaller production bundles due to more efficient tree-shaking.

## Monorepos and Linked Dependencies

In a monorepo setup, a dependency may be a linked package from the same repo. Vite automatically detects dependencies that are not resolved from `node_modules` and treats the linked dep as source code. It will not attempt to bundle the linked dep, and instead will analyze the linked dep's dependency list instead.

## Customizing the Behavior

The default dependency discovery heuristics may not always be desirable. In cases where you want to explicitly include/exclude dependencies from the list, use the [`optimizeDeps` config options](/config/#dep-optimization-options).

A typical use case for `optimizeDeps.include` or `optimizeDeps.exclude` is when you have an import that is not directly discoverable in the source code. For example, maybe the import is created as a result of a plugin transform. This means Vite won't be able to discover the import on the initial scan - it can only discover it after the file is requested by the browser and transformed. This will cause the server to immediately re-bundle after server start.

Both `include` and `exclude` can be used to deal with this. If the dependency is large (with many internal modules) or is CommonJS, then you should include it; If the dependency is small and is already valid ESM, you can exclude it and let the browser load it directly.
## Caching

Vite caches the pre-bundled dependencies in `node_modules/.vite`. It determines whether it needs to re-run the pre-bundling step based on a few sources:

- The `dependencies` list in your `package.json`
- Package manager lockfiles, e.g. `package-lock.json`, `yarn.lock`, or `pnpm-lock.yaml`.
- Relevant fields in your `vite.config.js`, if present.

The pre-bundling step will only need to be re-run when one of the above has changed.

If for some reason you want to force Vite to re-bundle deps, you can either start the dev server with the `--force` command line option, or manually delete the `node_modules/.vite` cache directory.
