# Websocket dependency update blockers

This audit covers one sample at base `3a07b3426ca9015fa2ddc9f7e178737999780851`
and the Dependabot updater failures observed on 2026-10-30. These are
`security_update_not_possible` results from dependency-update jobs, not failures
of an application build or test workflow. This fork has no current application
CI results.

## Bounded change

Remove the unused `css-loader` development dependency from this sample and let
Yarn prune its unused lockfile entries. The checked-in webpack configuration has
Fable and JavaScript loader rules only. The sources contain no CSS imports, and
`src/Client/public/index.html` loads stylesheets through external `<link>` tags.
The other samples are outside this change.

The resulting manifest, Yarn lockfile and installed dependency tree contain no
`css-loader` or `postcss`. No retained lock entry changes version or metadata,
and no new dependency is added. Node, Webpack, Fable and React versions and the
sample's webpack configuration remain unchanged. This removes the PostCSS
dependency path; it does not fix all remaining updater failures or reopen the
batch merge gate.

## Failure matrix

The minimum fixed versions below are those requested by the captured updater
jobs. They are a dated snapshot, not a claim that a listed version addresses
every future advisory.

| Failed dependency | Locked parent path / constraint | Required fixed version in the job | Compatible boundary and disposition | Related original batch PR |
| --- | --- | --- | --- | --- |
| PostCSS 7.0.16 | `css-loader@2.1.1` and its CSS Modules helpers require PostCSS 7 | 8.5.23 | This change removes the unused parent. If CSS importing is introduced later, css-loader 5.0.0 is an early compatible candidate for PostCSS 8 and Webpack 4.27+, but adds a Node 10.13+ requirement; that migration is not implemented here. css-loader 4.3.0 still requires PostCSS 7. | No direct PostCSS/css-loader PR in the finite batch. |
| webpack-dev-middleware 3.7.0 | `webpack-dev-server@3.6.0` requires `^3.7.0` | 7.4.5 | Even dev-server 3.11.3 requires middleware `^3.7.2`. Middleware 7.4.5 requires Webpack 5 and Node 18.12+. Dev-server 5.0.0 admits middleware 7, with the same Webpack/Node boundary. This needs a separate dev-server/config migration. | No direct middleware PR in the finite batch. |
| serialize-javascript 1.7.0 | `copy-webpack-plugin@4.6.0` requires `^1.4.0`; `webpack@4.33.0 -> terser-webpack-plugin@1.3.0` requires `^1.7.0` | 7.1.3 | Webpack 4.47.0 still uses terser-webpack-plugin 1.x, and plugin 1.4.6 still requires serialize-javascript `^4.0.0`. The requested package requires Node 20+. Updating just the terser library cannot remove both parent constraints. | [#61](https://github.com/ttak0422/Elmish.Streams/pull/61) updates terser 4.0.0 to 4.8.1, not these parents. |
| qs 6.7.0 | `webpack-dev-server@3.6.0 -> express@4.17.1` and `body-parser@1.19.0` pin qs 6.7.0 | 6.16.1 | A same-major candidate exists: Express 4.22.3 permits qs `~6.16.0` and body-parser `~1.20.5`; body-parser 1.20.8/1.20.9 permit qs `~6.16.0`, while 1.20.5/1.20.6 still permit only `~6.15.1`. Resolving both parents is necessary. This is a metadata-confirmed candidate, not an integration-tested fix in this PR. | [#76](https://github.com/ttak0422/Elmish.Streams/pull/76) proposes Express 4.18.2 and does not establish the required qs version. |
| decode-uri-component 0.2.0 | `source-map-resolve@0.5.2` requires `^0.2.0`, reached through Sass/chokidar, Webpack/micromatch, webpack-cli/findup-sync and dev-server/chokidar | 0.5.1 | source-map-resolve 0.6.0 is deprecated and still requires `^0.2.0`. A patch within that parent cannot admit 0.5.1. All listed dependency paths need replacement or removal; a forced Yarn resolution would cross an unverified range. | [#67](https://github.com/ttak0422/Elmish.Streams/pull/67) proposes 0.2.2, below the current job's required version. |

## Verification and limits

- Yarn 1.22.22 installs the changed lockfile with `--frozen-lockfile`.
- The dependency graph check finds no PostCSS/css-loader in the manifest, lock
  or installed modules; remaining lock entries are identical to base.
- Both base and changed trees pass a production React/JavaScript bundle through
  the actual sample webpack/Babel configuration, including source-map output.
- Both trees serve the original HTML with its stylesheet links, the generated
  bundle and an API request through the configured dev-server proxy. The test
  uses temporary loopback ports and an isolated backend.
- Existing dependency probes pass for tar archive round trips, AJV validation,
  terser minification, URL/URI/glob behavior, Express redirects and EventSource.

Tests ran on Node 22.22.2 with `--openssl-legacy-provider` for Webpack 4. Lifecycle
install scripts were disabled in the isolated validation environment. The bundle
test substitutes a JavaScript/React entry for the F# project entry: it does not
verify the full legacy .NET 2.2/Paket/Fake/Fable server build or browser HMR.
The Dependabot updater itself has not been rerun on the draft branch, so no
successful updater or CI result is claimed.

## Evidence

Captured failed updater runs: [PostCSS](https://github.com/ttak0422/Elmish.Streams/actions/runs/40994826051),
[middleware](https://github.com/ttak0422/Elmish.Streams/actions/runs/40994828883),
[serialize-javascript](https://github.com/ttak0422/Elmish.Streams/actions/runs/40994825945),
[qs](https://github.com/ttak0422/Elmish.Streams/actions/runs/40994825503),
[decode-uri-component](https://github.com/ttak0422/Elmish.Streams/actions/runs/40994491636).

Package constraints were checked against versioned publisher metadata:
[css-loader 4.3.0](https://registry.npmjs.org/css-loader/4.3.0),
[css-loader 5.0.0](https://registry.npmjs.org/css-loader/5.0.0),
[dev-server 3.11.3](https://registry.npmjs.org/webpack-dev-server/3.11.3),
[middleware 7.4.5](https://registry.npmjs.org/webpack-dev-middleware/7.4.5),
[dev-server 5.0.0](https://registry.npmjs.org/webpack-dev-server/5.0.0),
[serialize-javascript 7.1.3](https://registry.npmjs.org/serialize-javascript/7.1.3),
[Express 4.22.3](https://registry.npmjs.org/express/4.22.3),
[body-parser 1.20.8](https://registry.npmjs.org/body-parser/1.20.8),
[source-map-resolve 0.6.0](https://registry.npmjs.org/source-map-resolve/0.6.0).
