# next-routes-dir

`next-routes-dir` provides a way to use Next.js 13's `app/` directory colocation features while still using the feature-set of the old `pages/` directory. With `next-routes-dir`, you can create a `routes/` folder in your Next.js app that mirrors the folder structure of Next.js 13's `app/` directory, and it will automatically generate a valid `pages/` directory based on the structure of the `routes/` directory.

> You might be wondering, why don't we use Next.js 13's `app/` directory directly? In addition to the `app/` directory still being in beta, there are many incompatibilities and breaking changes from the `pages/` directory that require a substantial amount of work to migrate. However, the co-location features of the `app/` directory is very useful, so `next-routes-dir` provides a way to leverage the co-located structure of files inside the `app/` directory while still maintaining the same feature set provided by the `pages/` directory.

## Usage

Install `next-routes-dir` from npm:

```sh
npm install next-routes-dir
```

Then, refactor your existing `pages/` folder to the `app/` directory structure inside a `routes/` folder.

For example, the following `pages/` directory:

```text
.
└── pages/
    ├── index.tsx
    ├── login.tsx
    ├── app/
    │   └── index.tsx
    └── profile/
        └── [id].tsx
```

could be converted into the following `routes/` directory:

```text
.
└── routes/
    ├── login/
    │   └── page.tsx
    ├── (marketing)/
    │   └── page.tsx
    └── (app)/
        ├── layout.tsx
        ├── app/
        │   └── page.tsx
        └── profile/
            └── [id]/
                └── page.tsx
```

Then, import and run `setupRoutesDirectoryWatcher` in your `next.config.js` file:

```ts
// next.config.js
const { setupRoutesDirectoryWatcher } = require('next-routes-dir')
const path = require('path')

setupRoutesDirectoryWatcher({
  routesDir: path.join(__dirname, 'routes'),
  pagesDir: path.join(__dirname, 'src/pages')
})

module.exports = {
  // ...
}
```

If you want to wrap your exported components inside a wrapper function, you can pass `componentWrapperFunction`:

```ts
// next.config.js
const { setupRoutesDirectoryWatcher } = require('next-routes-dir')
const path = require('path')

setupRoutesDirectoryWatcher({
  routesDir: path.join(__dirname, 'routes'),
  pagesDir: path.join(__dirname, 'src/pages'),
  componentWrapperFunction: {
    name: '__definePage',
    path: '~/utils/page'
  }
})

module.exports = {
  // ...
}
```

This would generate component exports that look like the following:

```jsx
import { __definePage } from '~/utils/page'
export default __definePage(() => (
  <div>...</div>
))
```

Unlike the `pages/` directory, `next-routes-dir` also supports `getServerSideProps` from layout files! It does this by merging the results of the `getServerSideProps` in the layout file and the `getServerSideProps` in the route file.

## Caveats

Since the generated pages simply re-export  `getServerSideProps` from the routes file, Next.js is unable to treeshake the `getServerSideProps` from the original routes file. To work around this, you can use a custom Webpack plugin like the following:

```typescript
// next.config.js
module.exports = {
  webpack(config) {
    // Use string-replace-loader last, so that it runs _after_ everything has been transformed to JavaScript
    config.module.rules.unshift({
      test(id: string) {
        return routesDirs.some((routesDir) => isPathInside(id, routesDir))
      },
      loader: 'string-replace-loader',
      options: {
        search: /^[\S\s]*$/,
        replace(match: string) {
          const ast = acorn.parse(match, {
            ecmaVersion: 2020,
            sourceType: 'module'
          }) as any

          // Find the `getServerSideProps` named export
          const exportNamedDeclaration = ast.body.find(
            (node: any) =>
              (node.type === 'ExportNamedDeclaration' &&
                // Matches `export const getServerSideProps = ...`
                node.declaration?.declarations?.[0]?.id?.name ===
                  'getServerSideProps') ||
              // Matches `export function getServerSideProps() { ... }`
              node.declaration?.id === 'getServerSideProps'
          )

          // If a `getServerSideProps` export can't be found, return the original source code
          if (exportNamedDeclaration === undefined) {
            return match
          }

          const replacement =
            match.slice(0, exportNamedDeclaration.start) +
            match.slice(exportNamedDeclaration.end)

          return replacement
        }
      }
    })
  }
}
```
