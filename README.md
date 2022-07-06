# Import Reflection

## Status

Champion(s): Luca Casonato, Guy Bedford

Author(s): Luca Casonato, Guy Bedford

Stage: 1

## Motivation

For both JavaScript and WebAssembly, there is a need to be able to more closely
customize the loading, linking, and execution of modules beyond the standard host
execution model.

For WebAssembly, imports and exports for WebAssembly models often require custom
inspection and wrapping in order to behave correctly, which can be better
provided via manual instantiation than relying directly on the base
[ESM integration][wasm-esm] proposal.

For JavaScript, creating userland loaders would require a module reflection type
in order to share the host parsing, execution, security, and caching semantics.

## Proposal

This proposal enables module reflection capabilities for both JavaScript and
WebAssembly modules by enabling a new type of import, an import reflection:

```js
import module x from "<specifier>";
```

Here the module reflection keyword is added to the beginning of the ImportStatement.

Only the above form is supported - named exports and unbound declarations are not supported.

### Dynamic import()

```js
const x = await import("<specifier>", { reflect: "module" });
```

For dynamic imports, module import reflection is specified in the same second attribute
options bag that import assertions are specified in, using the `reflect` key.

If the [asset references proposal][] advances
in future it could share this same `reflect` key for dynamic asset imports as being symmetrical reflections:

```js
import asset x from "<specifier>";
await import("<specifier>", { reflect: "asset" });
```

Only the `module` reflection is specified by this proposal.

### JS Reflection

The type of the reflection object for a JavaScript module is currently host-defined,
but the goal would be for this to be a similar object to what is used by the
[module blocks][] specification or the [compartments][] specification.

### Wasm Reflection

The type of the reflection object for WebAssembly would be a
`WebAssembly.Module`, as defined in the
[WebAssembly JS integration API][wasm-js-api].

The reflection would represent an unlinked and uninstantiated module, while
still being able to support the same CSP policy as the native
[ESM integration][wasm-esm], avoiding the need for `unsafe-wasm-eval` for
custom Wasm execution.

Since reflection is defined through a host hook, this Wasm reflection is left
to be specified in the Wasm [ESM Integration][wasm-esm].

## Integration with other specs and environments

### WebAssembly ESM Integration

##### `WebAssembly.Module` imports

As explained in the motivation, supporting a `WebAssembly.Module` import reflection is a driving use case for this specification in order to change the behaviour of importing a direct compiled but unlinked [Wasm module object][]:

```js
import module FooModule from "./foo.wasm";
FooModule instanceof WebAssembly.Module; // true

// For example, to run a WASI execution with an API like Node.js WASI:
import { WASI } from 'wasi';
const wasi = new WASI({ args, env, preopens });

const fooInstance = await WebAssembly.instantiate(FooModule, {
  wasi_snapshot_preview1: wasi.wasiImport
});

wasi.start(fooInstance);
```

#### Imports from WebAssembly

Web Assembly would have the ability to expose import reflection in its
imports if desired, as the [Module Linking proposal currently aims to specify][module-linking].

#### Security improvements

The ability to relate a script or module to how it was obtained is an important
security property on the web and other JS runtimes. [CSP][] is the most well-known web platform feature that allows you to limit the capabilities the platform grants to a given site.

A common use is to disallow dynamic code generation means like `eval`. Wasm
compilation is unfortunatly completly dynamic right now (manual network fetch +
compile), so Wasm unconditionally requires a `script-src: unsafe-wasm-eval` CSP
attribute.

With this proposal, the Wasm module would be known statically, so would not have
to be considered as dynamic code generation. This would allow the web platform
to lift this restriction for statically imported Wasm modules, and instead just
require `script-src: self` (or possibly `wasm-src: self`) like for JS. Also see
https://github.com/WebAssembly/esm-integration/issues/56.

This property does not just impact platforms using CSP, but also other platforms
with systems to restrict permissions, such as Deno.

### Module Blocks

Ideally this proposal would use the module object defined by the [module blocks][] proposal.

Just as module blocks permit the dynamic import of blocks relating to their primary host instance,
it could be worth supporting the same for `WebAssembly.Module` for symmetry under a reflection model:

```js
import module WasmModule from './module.wasm';

WasmModule instanceof WebAssembly.Module; // true

await import(WasmModule); // returns the fully linked and instantiated ModuleNamespace
```

It is currently not clear if this would provide any strong benefits, therefore it is not currently specified, but could be an option for additions to this specification.

Implementing the above would require new specification machinery to relate a special internal slot on the `WebAssembly.Module`
to its Cyclic Module Record. One benefit of this approach would also be enabling Wasm modules to participate in cycles with JS modules.

### Compartments

Import reflection aims to be compatible with compartments hooks.

Compartment hooks may benefit from the ability to securely import module records as discussed in the security section.

This would allow creating custom loaders without requiring relaxing the security guarantees of the environment
since all compiled modules being loaded can be fully audited by the host instead of having to enable arbitrary modular evaluation string sources.

### Workers

Since `WebAssembly.Module` and the `Module` object in module blocks both support being passed to a worker, we should with this alignment have all reflections being worker-compatible.

## Cache Key Semantics

Semantically this proposal involves a relaxation of the `HostResolveImportedModule` idempotency requirement.

The proposed approach would be a _clone_ behaviour, where imports to the same module of different
reflection types result in separate keys. These semantics do run counter to the intuition
that there is just one copy of a module.

The specification would then split the `HostResolveImportedModule` hook into two components -
module asset resolution, and module asset interpretation. The module asset resolution component
would retain the exact same idempotency requirement, while the module asset interpretation component
would have idempotency down to keying on the module asset and reflection type pair.

Effectively, this splits the module cache into two separate caches - an asset cache retaining the
current idempotency of the `HostResolveImportedModule` host hook, pointing to an opaque cached asset reference,
and a module instance cache, keyed by this opaque asset reference and the reflection type.

Alternative proposals include:

- **Race** and use the attribute that was requested by the first import. This
  seems broken in that the second usage is ignored.
- **Reject** the module graph and don't load if attributes differ. This seems
  bad for composition--using two unrelated packages together could break, if
  they load the same module with disagreeing attributes.

Both of these alternatives seem less versatile than the proposed _clone_ behaviour above.

## Q&A

**Q**: How does this relate to import assertions?

**A**: Import assertions do not influence how an imported asset is evaluated, and they
do not influence the HostResolveImportedModule idempotency requirements. This
proposal does. Also see
https://github.com/tc39/proposal-import-assertions#follow-up-proposal-evaluator-attributes.

**Q**: Would this proposal enable the importing of other languages directly as modules?

**A**: While hosts may define import reflection, expanding the evaluation of
arbitrary language syntax to the web is not seen as a motivating use case for this proposal.

**Q**: Why not just use `const module = await WebAssembly.compileStreaming(fetch(new URL("./module.wasm", import.meta.url)));`?

**A**: There are multiple benefits: firstly if the module is statically referenced in the module
graph, it is easier to statically analyze (by bundlers for example). Secondly when using CSP,
`script-src: unsafe-eval` would not be needed. See the security improvements section for more
details.

[CSP]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy
[Wasm module object]: https://webassembly.github.io/spec/js-api/index.html#modules
[asset references proposal]: https://github.com/tc39/proposal-asset-references
[compartments]: https://github.com/tc39/proposal-compartments
[module-linking]: https://github.com/WebAssembly/module-linking/blob/main/proposals/module-linking/Binary.md#import-section-updates
[module blocks]: https://github.com/tc39/proposal-js-module-blocks
[wasm-js-api]: https://webassembly.github.io/spec/js-api/#modules
[wasm-esm]: https://github.com/WebAssembly/esm-integration/tree/master/proposals/esm-integration