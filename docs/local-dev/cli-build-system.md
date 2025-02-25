---
id: cli-build-system
title: Build System
description: A deep dive into the Backstage build system
---

The Backstage build system is a collection of build and development tools that
help you lint, test, develop and finally release your Backstage projects. The
purpose of the build system is to provide an out-of-the-box solution that works
well with Backstage and lets you focus on building an app rather than having to
spend time setting up your own tooling.

The build system setup is part of the
[@backstage/cli](https://www.npmjs.com/package/@backstage/cli), and already
included in any project that you create using
[@backstage/create-app](https://www.npmjs.com/package/@backstage/create-app). It
is similar to for example
[react-scripts](https://www.npmjs.com/package/react-scripts), which is the
tooling you get with
[create-react-app](https://github.com/facebook/create-react-app). The Backstage
build system relies heavily on existing open source tools from the JavaScript
and TypeScript ecosystem, such as [Webpack](https://webpack.js.org/),
[Rollup](https://rollupjs.org/), [Jest](https://jestjs.io/), and
[ESLint](https://eslint.org/).

## Design Considerations

There are a couple of core beliefs and constraints that guided the design of the
Backstage build system. The first and most important is that we put the
development experience first. If we need to cut corners or add complexity we do
so in other areas, but the experience of firing up an editor and iterating on
some code should always be as smooth as possible.

In addition, there are a number of hard and soft requirements:

- Monorepos - The build system should support multi-package workspaces
- Publishing - It should be possible to build and publish individual packages
- Scale - It should scale to hundreds of large packages without excessive wait
  times
- Reloads - The development flow should support quick on-save hot reloads
- Simple - Usage should simple and configuration should be kept minimal
- Universal - Development towards both web applications, isomorphic packages,
  and Node.js
- Modern - The build system targets modern environments
- Editors - Type checking and linting should be available within most editors

During the design of the build system this collection of requirements was not
something that was supported by existing tools like for example `react-scripts`.
The requirements of scaling in combination of monorepo, publishing, and editor
support led us to adopting our own specialized setup.

## Structure

We can divide the development flow within Backstage into a couple of different
steps:

- **Formatting** - Applies a consistent formatting to your source code
- **Linting** - Analyzes your source code for potential problems
- **Type Checking** - Verifies that TypeScript types are valid
- **Testing** - Runs different levels of test suites towards your project
- **Building** - Compiles the source code in an individual package
- **Bundling** - Combines a package and all of its dependencies into a
  production-ready bundle

These steps are generally kept isolated form each other, with each step focusing
on its specific task. For example, we do not do linting or type checking
together with the building or bundling. This is so that we can provide more
flexibility and avoid duplicate work, improving performance. It is strongly
recommended that as a part of developing withing Backstage you use a code editor
or IDE that has support for formatting, linting, and type checking.

Let's dive into a detailed look at each of these steps and how they are
implemented in a typical Backstage app.

## Formatting

The formatting setup lives completely within each Backstage application and is
not part of the CLI. In an app created with `@backstage/create-app` the
formatting is handled by [prettier](https://prettier.io/), but each application
can choose their own formatting rules and switch to a different formatter if
desired.

## Linting

The Backstage CLI includes a `lint` command, which is a thin wrapper around
`eslint`. It adds a few options that can't be set through configuration, such as
including the `.ts` and `.tsx` extensions in the set of linted files. The `lint`
command simply provides a sane default and is not intended to be customizable.
If you want to supply more advanced options you can invoke `eslint` directly
instead.

In addition to the `lint` command, the Backstage CLI also includes a set of base
ESLint configurations, one for frontend and one for backend packages. These lint
configurations in turn build on top of the lint rules from
[@spotify/web-scripts](https://github.com/spotify/web-scripts).

In a standard Backstage setup, each individual package has its own lint
configuration, along with a root configuration that applies to the entire
project. Each configuration is initially one that simply extends a base
configuration provided by the Backstage CLI, but they can be customized to fit
the needs of each package.

## Type Checking

Just like formatting, the Backstage CLI does not have its own command for type
checking. It does however have a base configuration with both recommended
defaults as well as some required settings for the build system to work.

Perhaps the most notable part about the TypeScript setup in Backstage projects
is that the entire project is one big compilation unit. This is due to
performance optimization as well as ease of use, since breaking projects down
into smaller pieces has proven to both lead to a more complicated setup, as well
as type checking of the entire project being an order of magnitude slower. In
order to make this setup work, the entrypoint of each package needs to point to
the TypeScript source files, which in turn causes some complications during
publishing that we'll talk about in [that section](#publishing).

The type checking is generally configured to be incremental for local
development, with the output stored in the `dist-types` folder in the repo root.
This provides a significant speedup when running `tsc` multiple times locally,
but it does make the initial run a little bit slower. Because of the slower
initial run we disable incremental type checking in the `tcs:full` Yarn script
that is included by default in any Backstage app and is intended for use in CI.

Another optimization that is used by default is to skip the checking of library
types, this means that TypeScript will not verify that types within
`node_modules` are sound. Disabling this check significantly speeds up type
checking, but in the end it is still an important check that should not be
completely omitted, it's simply unlikely to catch issues that are introduced
during local development. What we opt for instead is to include the check in CI
through the `tsc:full` script, which will run a full type check, including
`node_modules`.

For the two reasons mentioned above, it is **highly** recommended to use the
`tsc:full` script to run type checking in CI.

## Testing

As mentioned above, the Backstage CLI uses [Jest](https://jestjs.io/), which is
a JavaScript test framework that covers both test execution and assertions. Jest
executes all tests in Node.js, including frontend browser code. The trick it
uses is to execute the tests in a Node.js VM using various predefined
environments, such as one based on [`jsdom`](https://github.com/jsdom/jsdom)
that helps mimic browser APIs and behavior.

The Backstage CLI has its own command that helps execute tests,
`backstage-cli test`, as well as its own configuration at
`@backstage/cli/config/jest.js`. The command is a relatively thin wrapper around
running `jest` directly. Its main responsibility is to make sure the included
configuration is used, as well setting the `NODE_ENV` and `TZ` environment
variables, and provided some sane default flags like `--watch` if executed
within a Git repository.

The by far biggest amount of work is done by the Jest configuration included
with the Backstage CLI. It both takes care of providing a default Jest
configuration, as well as allowing for configuration overrides to be defined in
each `package.json`. How this can be done in practice is discussed in the
[Jest configuration](#jest-configuration) section.

## Building

The primary purpose of the build process is to prepare packages for publishing,
but it's also used as part of the backend bundling process. Since it's only used
in these two cases, any Backstage app that does not use the Backend parts of the
project may not need to interact with the build process at all. It can
nevertheless be useful to know how it works, since all of the published
Backstage packages are built using this process.

The build is currently using [Rollup](https://rollupjs.org/) and executes in
isolation for each individual package. There are currently three different
commands in the Backstage CLI that invokes the build process, `plugin:build`,
`backend:build`, and simply `build`. The two former are pre-configured commands
for frontend and backend plugins, while the `build` command provides more
control over the output.

There are three different possible outputs of the build process: JavaScript in
CommonJS module format, JavaScript in ECMAScript module format, and type
declarations. Each invocation of a build command will write one or more of these
outputs to the `dist` folder in the package, and in addition copy any asset
files like stylesheets or images. For more details on what syntax and file
formats are supported by the build process, see the [loaders section](#loaders).

When building CommonJS or ESM output, the build commands will always use
`src/index.ts` as the entrypoint. All dependencies of the package will be marked
as external, meaning that in general it is only the contents of the `src` folder
that ends up being compiled and output to `dist`. All import statements of
external dependencies, even within the same monorepo, will stay intact. The
externalized dependencies are based on dependency information in `package.json`,
which means it's important to keep it up to date.

The build of the type definitions works quite differently. The entrypoint of the
type definition build is the relative location of the package within the
`dist-types` folder in the project root. This means that it is important to run
type checking before building any packages with type definitions, and that
emitting type declarations must be enabled in the TypeScript configuration. The
reason for the type definition build step is to strip out all types but the ones
that are exported from the package, leaving a much cleaner type definition file
and making sure that the type definitions are in sync with the generated
JavaScript.

## Bundling

The goal of the bundling process is to combine multiple packages together into a
single runtime unit. The way this is done varies between frontend and backend,
as well as local development versus production deployment. Because of that we
cover each combination of these cases separately.

### Frontend Development

There are two different commands that start the frontend development bundling:
`app:serve`, which serves an app and uses `src/index` as the entrypoint, and
`plugin:serve`, which serves a plugin and uses `dev/index` as the entrypoint.
These are typically invoked via the `yarn start` script, and are intended for
local development only. When running the bundle command, a development server
will be set up that listens to the protocol, host and port set by `app.baseUrl`
in the configuration. If needed it is also possible to override the listening
options through the `app.listen` configuration.

The frontend development bundling is currently based on
[Webpack](https://webpack.js.org/) and
[Webpack Dev Server](https://webpack.js.org/configuration/dev-server/). The
Webpack configuration itself varies very little between the frontend development
and production bundling, so we'll dive more into the configuration in the
production section below. The main differences are that `process.env.NODE_ENV`
is set to `'development'`, minification is disabled, cheap source maps are used,
and [React Hot Loader](https://github.com/gaearon/react-hot-loader) is enabled.

If you prefer to run type checking and linting as part of the Webpack process,
you can enable usage of the
[`ForkTsCheckerWebpackPlugin`](https://www.npmjs.com/package/fork-ts-checker-webpack-plugin)
by passing the `--check` flag. Although as mentioned above, the recommended way
to handle these checks during development is to use an editor that has built-in
support for them instead.

### Frontend Production

The frontend production bundling creates your typical web content bundle, all
contained within a single folder, ready for static serving. It is invoked using
the `app:build` command, and unlike the development bundling there is no way to
build a production bundle of an individual plugin. The output of the bundling
process is written to the `dist` folder in the package.

Just like the development bundling, the production bundling is based on
[Webpack](https://webpack.js.org/). It uses the
[`HtmlWebpackPlugin`](https://webpack.js.org/plugins/html-webpack-plugin/) to
generate the `index.html` entry point, and includes a default template that's
included with the CLI. You can replace the bundled template by adding
`public/index.html` to your app package. The template has access to two global
constants, `publicPath` which is the public base path that the bundle is
intended to be served at, as well as `config` which is your regular frontend
scoped configuration from `@backstage/config`.

The Webpack configuration also includes a custom plugin for resolving packages
correctly from linked in packages, the `ModuleScopePlugin` from
[`react-dev-utils`](https://www.npmjs.com/package/react-dev-utils) which makes
sure that imports don't reach outside the package, a few fallbacks for some
Node.js modules like `'buffer'` and `'events'`, a plugin that writes the
frontend configuration to the bundle as `process.env.APP_CONFIG` and build
information as `process.env.BUILD_INFO`, and lastly minification handled by
[esbuild](https://esbuild.github.io/) using the
[`esbuild-loader`](https://npm.im/esbuild-loader). There are of course also a
set of loaders configured, which you can read more about in the
[loaders](#loaders) and [transpilation](#transpilation) sections.

The output of the bundling process is split into two categories of files with
separate caching strategies. The first is a set of generic assets with plain
names in the root of the `dist/` folder. You will want to serve these with
short-lived caching or no caching at all. The second is a set of hashed static
assets in the `dist/static/` folder, which you can configure to be cached for a
much longer time.

The configuration of static assets is optimized for frequent changes and serving
over HTTP 2.0. The assets are aggressively split into small chunks, which means
the browser has to make a lot of small requests to load them. The upside is that
changes to individual plugins and packages will invalidate a smaller number of
files, thereby allowing for rapid development without much impact on the page
load performance.

### Backend Development

The backend development bundling is also based on Webpack, but rather than
starting up a web server, the backend is started up using the
[`RunScriptWebpackPlugin`](https://www.npmjs.com/package/run-script-webpack-plugin).
The reason for using Webpack for development of the backend is both that it is a
convenient way to handle transpilation of a large set of packages, as well us
allowing us to use hot module replacement and maintaining state while reloading
individual backend modules. This is particularly useful when running the backend
with in-memory SQLite as the database choice.

Except for executing in Node.js rather than a web server, the backend
development bundling configuration is quite similar to the frontend one. It
shares most of the Webpack configuration, including the transpilation setup.
Some differences are that it does not inject any environment variables or node
module fallbacks, and it uses
[`webpack-node-externals`](https://www.npmjs.com/package/webpack-node-externals)
to avoid bundling in dependency modules.

If you want to inspect the running Node.js process, the `--inspect` and
`--inspect-brk` flags can be used, as they will be passed through as options to
`node` execution.

### Backend Production

The backend production bundling uses a completely different setup than the other
bundling options. Rather than using Webpack, the backend production bundling
instead collects the backend packages and all of their local dependencies into a
deployment archive. The archive is written to `dist/bundle.tar.gz`, and contains
the packaged version of each of these packages. The layout of the packages in
the archive is the same as the directory layout in the monorepo, and the bundle
also contains the root `package.json` and `yarn.lock` files.

Note that before creating a production bundle you must first build all of the
backend packages. This can be done automatically when executing the
`backend:bundle` command by passing the `--build-dependencies` flag. It is an
optional flag since it is quite common that the packages are already built
earlier on in your build process, and building them again would result in
duplicate work.

In order to use the bundle, you extract it into a directory, run
`yarn install --production`, and then start the backend using your backend
package as the Node.js entry point, for example `node packages/backend`.

The `dist/bundle.tar.gz` is accompanied by a `dist/skeleton.tar.gz`, which has
the same layout, but only contains `package.json` files. This skeleton archive
can be used to run a `yarn install` in environments that will benefit from the
caching that this enables, such as Docker image builds. To use the skeleton
archive you copy it over to the target directory along with the root
`package.json` and `yarn.lock`, extract the archive, and then run
`yarn install --production`. Your target directory will then have all
dependencies installed, and as soon as you copy over and extract the contents of
the `bundle.tar.gz` archive on top of it, the backend will be ready to run.

The following is an example of a `Dockerfile` that can be used to package the
output of `backstage-cli backend:bundle` into an image:

```Dockerfile
FROM node:14-buster-slim
WORKDIR /app

COPY yarn.lock package.json packages/backend/dist/skeleton.tar.gz ./
RUN tar xzf skeleton.tar.gz && rm skeleton.tar.gz

RUN yarn install --production --frozen-lockfile --network-timeout 300000 && rm -rf "$(yarn cache dir)"

COPY packages/backend/dist/bundle.tar.gz app-config.yaml ./
RUN tar xzf bundle.tar.gz && rm bundle.tar.gz

CMD ["node", "packages/backend"]
```

## Transpilation

The transpilers used by the Backstage CLI have been chosen according to the same
design considerations that were mentioned above. A few specific requirements are
of course support for TypeScript and JSX, but also React hot reloads or refresh,
and hoisting of Jest mocks. The Backstage CLI also only targets up to date and
modern browsers, so we actually want to keep the transpilation process as
lightweight as possible, and leave most syntax intact.

Apart from these requirements, the deciding factor for which transpiler to use
is their speed. The build process keeps the integration with the transpilers
lightweight, without additional plugins or such. This enables us to switch out
transpilers as new options and optimizations become available, and keep on using
the best options that are available.

Our current selection of transpilers are [esbuild](https://esbuild.github.io/)
and [Sucrase](https://github.com/alangpierce/sucrase). The reason we choose to
use two transpilers is that esbuild is faster than Sucrase and produces slightly
nicer output, but it does not have the same set of features, for example it does
not support React hot reloading.

The benchmarking of the various options was done in
[ts-build-bench](https://github.com/Rugvip/ts-build-bench). This benchmarking
project allows for setups of different shapes and sizes of monorepos, but the
setup we consider the most important in our case is a large monorepo with lots
of medium to large packages that are being bundled with Webpack. Some rough
findings have been that esbuild is the fastest option right now, with Sucrase
following closely after and then [SWC](https://swc.rs/) closely after that.
After those there's a pretty big gap up to the TypeScript compiler run in
transpilation only mode, and lastly another jump up to Babel, being by far the
slowest out of the transpilers we tested.

Something to note about these benchmarks is that they take the full Webpack
bundling time into account. This means that even though some transpilation
options may be orders of magnitude faster than others, the total time is not
impacted in the same way as there are lots of other things that go into the
bundling process. Still, switching from for example Babel to Sucrase is able to
make the bundling anywhere from two to five times faster.

## Loaders

The Backstage CLI is configured to support a set of loaders throughout all parts
of the build system, including the bundling, tests, builds, and type checking.
Loaders are always selected based on the file extension. The following is a list
of all supported file extensions:

| Extension   | Exports         | Purpose                                                                       |
| ----------- | --------------- | ----------------------------------------------------------------------------- |
| `.ts`       | Script Module   | TypeScript                                                                    |
| `.tsx`      | Script Module   | TypeScript and XML                                                            |
| `.js`       | Script Module   | JavaScript                                                                    |
| `.jsx`      | Script Module   | JavaScript and XML                                                            |
| `.mjs`      | Script Module   | ECMAScript Module                                                             |
| `.cjs`      | Script Module   | CommonJS Module                                                               |
| `.json`     | JSON Data       | JSON Data                                                                     |
| `.yml`      | JSON Data       | YAML Data                                                                     |
| `.yaml`     | JSON Data       | YAML Data                                                                     |
| `.css`      | classes         | Style sheet                                                                   |
| `.eot`      | URL Path        | Font                                                                          |
| `.ttf`      | URL Path        | Font                                                                          |
| `.woff2`    | URL Path        | Font                                                                          |
| `.woff`     | URL Path        | Font                                                                          |
| `.bmp`      | URL Path        | Image                                                                         |
| `.gif`      | URL Path        | Image                                                                         |
| `.jpeg`     | URL Path        | Image                                                                         |
| `.jpg`      | URL Path        | Image                                                                         |
| `.png`      | URL Path        | Image                                                                         |
| `.svg`      | URL Path        | Image                                                                         |
| `.icon.svg` | React Component | SVG converted into a [MUI SvgIcon](https://mui.com/components/icons/#svgicon) |

## Jest Configuration

The Backstage CLI bundles its own Jest configuration file, which is used
automatically when running `backstage-cli test`. It's available at
`@backstage/cli/config/jest.js` and can be inspected
[here](https://github.com/backstage/backstage/blob/master/packages/cli/config/jest.js).
Usage of this configuration can be overridden either by passing a
`--config <path>` flag to `backstage-cli test`, or placing a `jest.config.js` or
`jest.config.ts` file in your package.

The built-in configuration brings a couple of benefits and features. The most
important one being a baseline transformer and module configuration that enables
support for the listed [loaders](#loaders) within tests. It will also
automatically detect and use `src/setupTests.ts` if it exists, and provides a
coverage configuration that works well with our selected transpilers.

The configuration also takes a project-wide approach, with the expectation most
if not all packages within a monorepo will use the same base configuration. This
allows for optimizations such as sharing the Jest transform cache across all
packages in a monorepo, avoiding unnecessary transpilation. It also makes it
possible to load in all Jest configurations at once, and with that run
`yarn test <pattern>` from the root of a monorepo without having to set the
working directory to the package that the test is in.

Where small customizations are needed, such as setting coverage thresholds or
support for specific transforms, it is possible to override the Jest
configuration through the `"jest"` field in `package.json`. These overrides will
be loaded in from all `package.json` files in the directory ancestry, meaning
that you can place common configuration in the `package.json` at the root of a
monorepo. If multiple overrides are found, they will be merged together with
configuration further down in the directory tree taking precedence.

The overrides in a single `package.json` may for example look like this:

```json
  "jest": {
    "coverageThreshold": {
      "global": {
        "functions": 100,
        "lines": 100,
        "statements": 100
      }
    }
  },
```

## Publishing

Package publishing is an optional part of the Backstage build system and not
something you will need to worry about unless you are publishing packages to a
registry. In order to publish a package, you first need to build it, which will
populate the `dist` folder. Because the Backstage build system is optimized for
local development along with our particular TypeScript and bundling setup, it is
not possible to publish the package immediately at this point. This is because
the entry points of the package will still be pointing to `src/index.ts`, but we
want them to point to `dist/` in the published package.

In order to work around this, the Backstage CLI provides `prepack` and
`postpack` commands that help prepare the package for publishing. These scripts
are automatically run by Yarn before publishing a package.

The `prepack` command will take entry point fields in `"publishConfig"`, such as
`"main"` and `"module"`, and move them to the top level of the `package.json`.
This lets you point at the desired files in the `dist` folder during publishing.
The `postpack` command will simply revert this change in order to leave your
project clean.

The following is an excerpt of a typical setup of an isomorphic library package:

```json
  "main": "src/index.ts",
  "types": "src/index.ts",
  "publishConfig": {
    "access": "public",
    "main": "dist/index.cjs.js",
    "module": "dist/index.esm.js",
    "types": "dist/index.d.ts"
  },
  "scripts": {
    "build": "backstage-cli build",
    "lint": "backstage-cli lint",
    "test": "backstage-cli test",
    "prepack": "backstage-cli prepack",
    "postpack": "backstage-cli postpack",
    "clean": "backstage-cli clean"
  },
  "files": ["dist"],
```
