# nuxt-bug-externals-whitelist

> Bug Reproduction for 'node_modules with "." in their module path accidentally whitelisted'

[GitHub Issue](https://github.com/nuxt/nuxt.js/issues/7462) | [CTMY Issue](https://cmty.app/nuxt/nuxt.js/issues/c10729)

## Bug Reproduction

1. Run `yarn install`
2. Add `console.log("vue-states installation trace", new Error().stack);` <br />in the install function in `node_modules/@sumcumo/vue-states/dist/vue-states.umd.js:176`
3. Run `yarn dev`
4. Make a request to the server
5. Check the stack trace that is printed.
   - It does not refer to `node_modules/@sum.cumo/vue-states` but only to `server.js`
   - The plugin is installed once per request
6. Move the module
   - `mv node_modules/@sum.cumo node_modules/@sumcumo`
   - Adjust path in `plugins/vue-states.js:2`
7. Restart the dev server
8. Make a request to the server
9. Check the stack trace that is printed.
   - It does now refer to `node_modules/@sumcumo/vue-states/dist/vue-states.umd.js:177:58`
10. Fix the bug by changing `/\.(?!js(x|on)?$)/i,` in `node_modules/@nuxt/webpack/dist/webpack.js:5098` <br />to `path => !/^(|\.js(x|on)?)$/.test(require('path').extname(path))` <br />Source: https://github.com/nuxt/nuxt.js/blob/dev/packages/webpack/src/config/server.js#L23
11. Move the package back to `@sum.cumo` and restart
12. Make a request to the server
13. Check the stack-trace, the plugin is now **not** bundled

The current implementation interprets paths like `@scoped.package/name` **and** `package/some.umd.js`
as imports that must be bundled with the app, resulting in unintended behaviour.

## Additional comments

One side-effect of this behaviour is, that the package from node_modules is re-executed whenever the bundle is re-executed. 
That is once per SSR-request in devMode, since the default for runInNewContext is then set to `true`.

The re-execution produces a *new Instance* of the plugin, which effectively disables [the Vue.js internal check if the Plugin has already been installed](https://github.com/vuejs/vue/blob/dev/src/core/global-api/use.js#L8).
Since the Vue module is *not* re-executed, the Plugin is then installed multiple times, which can lead to secondary, Plugin-dependent Bugs.

In case of `@sum.cumo/vue-states` the Plugin (unsuccessfully) tried to redefine a property on a component once per installation.

## Notes about the fix

It seems impossible to check if `@scope/some.module` should be bundled (without checking the file system),
since `some.module` could be the name both of a package (folder) or of a (non-native) file.
