# Import Assertions

Champions: Sven Sauleau ([@xtuc](https://github.com/xtuc)), Daniel Ehrenberg ([@littledan](https://github.com/littledan)), Myles Borins ([@MylesBorins](https://github.com/MylesBorins)), and Dan Clark ([@dandclark](https://github.com/dandclark))

Status: Stage 3.

Please leave any feedback you have in the [issues](http://github.com/tc39/proposal-import-assertions/issues)!

## Synopsis

The Import Assertions proposal adds an inline syntax for module import statements to pass on more information alongside the module specifier.  The initial application for such assertions will be to support additional types of modules in a common way across JavaScript environments, starting with [JSON modules](http://github.com/tc39/proposal-json-modules).

The syntax will be as follows (shown here is the proposed method for importing a JSON module):
```js
import json from "./foo.json" assert { type: "json" };
import("foo.json", { assert: { type: "json" } });
```

The specification of JSON modules was originally part of this proposal, but it was [resolved](https://github.com/tc39/notes/blob/master/meetings/2020-07/july-21.md#import-conditions-for-stage-3) during the July 2020 meeting to split JSON modules out into a [separate Stage 3 proposal](http://github.com/tc39/proposal-json-modules).

## Motivation

Standards-track JSON ES modules were [proposed](https://github.com/w3c/webcomponents/issues/770) to allow JavaScript modules to easily import JSON data files, similarly to how they are supported in many nonstandard JavaScript module systems. This idea quickly got broad support from web developers and browsers, and was merged into HTML, with an implementation for V8/Chromium created by Microsoft.

However, in [an issue](https://github.com/w3c/webcomponents/issues/839), Ryosuke Niwa (Apple) and Anne van Kesteren (Mozilla) proposed that security would be improved if some syntactic marker were required when importing JSON modules and similar module types which cannot execute code, to prevent a scenario where the responding server unexpectedly provides a different MIME type, causing code to be unexpectedly executed. The solution was to somehow indicate that a module was JSON, or in general, not to be executed, somewhere in addition to the MIME type.

Some developers have the intuition that the file extension could be used to determine the module type, as it is in many existing non-standard module systems. However, it's a deep web architectural principle that the suffix of the URL (which you might think of as the "file extension" outside of the web) does not lead to semantics of how the page is interpreted. In practice, on the web, there is a widespread [mismatch between file extension and the HTTP Content Type header](content-type-vs-file-extension.md). All of this sums up to it being infeasible to depend on file extensions/suffixes included in the module specifier to be the basis for this checking.

There are other possible pieces of metadata which could be associated with modules, see [#8](https://github.com/tc39/proposal-import-assertions/issues/8) for further discussion.

Proposed ES module types that are blocked by this security concern, in addition to JSON modules, include [CSS modules](https://github.com/whatwg/html/pull/4898) and potentially [HTML modules](https://github.com/whatwg/html/pull/4505) if the HTML module  proposal is restricted to [not allow script](https://github.com/w3c/webcomponents/issues/805).

## Rationale

There are three places where this data could be provided:
- As part of the module specifier (e.g., as a pseudo-scheme)
    - Challenges: Adds complexity to URLs or other module specifier syntaxes, and risks being confusing to developers (further discussion: [#11](https://github.com/tc39/proposal-import-assertions/issues/11))
    - webpack supports this sort of construct ([docs](https://webpack.js.org/concepts/loaders/#inline)).
        - Demand from users for similar behavior in Parcel, with pushback from some maintainers ([#3477](https://github.com/parcel-bundler/parcel/issues/3477))
- Separately, out of band (e.g., a separate resource file)
    - Challenges: How to load that resource file; what should the format be; unergonomic to have to jump between files during development (further discussion: [#13](https://github.com/tc39/proposal-import-assertions/issues/13))
- In the JavaScript source text
    - Challenges: Requires a change at the JavaScript language level (this proposal)

This proposal pursues the third option, as we expect it to lead to the best developer experience, and are hopeful that language design/standardization issues can be resolved.

## Proposed syntax

Import assertions have to be made available in several different contexts. This section contains one possible syntax, but there are other options, discussed in [#6](https://github.com/tc39/proposal-import-assertions/issues/6).

Here, a key-value syntax is used, with the key `type` used as an example indicating the module type. Such key-value syntax can be used in various different contexts.

### import statements

The ImportDeclaration would allow any arbitrary assertions after the `assert` keyword.

For example, the `type` assertion could be used to indicate a module type, for example importing a JSON module with the following syntax.

```mjs
import json from "./foo.json" assert { type: "json" };
```

The `assert` syntax in the `ImportDeclaration` statement uses curly braces, for the following reasons (as discussed in [#5](https://github.com/tc39/proposal-import-assertions/issues/5)):
- JavaScript developers are already used to the Object literal syntax and since it allows a trailing comma copy/pasting assertions will be easy.
- Follow-up proposals might specify new types of import attributes (see [Follow-up proposal "evaluator attributes"](https://github.com/tc39/proposal-import-assertions#follow-up-proposal-evaluator-attributes)) and we will be able to group attributes with different keywords, for instance:
```js
import json from "./foo.json" assert { type: "json" } with { transformA: "value" };
```

The `assert` keyword is designed to match the check-only semantics. As shown by the example above, one could imagine a new follow-up proposal that uses `with` for transformations.

### re-export statements

Similar to import statements, the ExportDeclaration, when re-exporting from another module, would allow any arbitrary assertions after the `assert` keyword.

```mjs
export { val } from './foo.js' assert { type: "javascript" };
```

### dynamic import()

The `import()` pseudo-function would allow import assertions to be indicated in an options bag in the second argument.

```js
import("foo.json", { assert: { type: "json" } })
```

The second parameter to `import()` is an options bag, with the only option currently defined to be `assert`: the value here is an object containing the import assertions. The options bag can be extended by follow-up proposals, e.g. the [Import reflection](https://github.com/tc39/proposal-import-reflection) proposal defines a `reflect` option.

### Integration of modules into environments

Host environments (e.g., the Web platform, Node.js) often provide various different ways of loading modules. The analogous string could be passed through these ways of loading other kinds of modules.

#### Worker instantiation

```js
new Worker("foo.wasm", { type: "module", assert: { type: "webassembly" } });
```

Sidebar about WebAssembly module types and the web: it's still uncertain whether importing WebAssembly modules would need to be marked specially, or would be imported just like JavaScript. Further discussion in [#19](https://github.com/tc39/proposal-import-assertions/issues/19).

#### HTML

Although changes to HTML won't be specified by TC39, an idea here would be that each import attribute, preceded by `assert`, becomes an HTML attribute which could be used in script tags.

```html
<script src="foo.wasm" type="module" asserttype="webassembly"></script>
```

(See the caveat about WebAssembly above.)

#### WebAssembly

In the context of the [WebAssembly/ESM integration proposal](https://github.com/webassembly/esm-integration): For imports of other module types from within a WebAssembly module, this proposal would introduce a new custom section (named `importassertions`) that will annotate with assertions each imported module (which is listed in the import section).

## Proposed semantics and interoperability

This proposal does not specify behavior for any particular assertion key or value.  The [JSON modules proposal](https://github.com/tc39/proposal-json-modules) will specify that `type: "json"` must be interpreted as a JSON module, and will specify common semantics for doing so.  It is expected the `type` attribute will be leveraged to support additional module types in future TC39 proposals as well as by hosts.  HTML and CSS modules are under consideration, and these may use similar explicit `type` syntax when imported.

Assertions in addition  than `type` may also be introduced for purposes not yet foreseen.

JavaScript implementations are encouraged to reject assertions and type values which are not implemented in their environment (rather than ignoring them). This is to allow for maximal flexibility in the design space in the future--in particular, it enables new import assertions to be defined which change the interpretation of a module, without breaking backwards-compatibility.

## Follow-up proposal "evaluator attributes"

Implementations are not permitted to interpret a module differently at multiple import sites if the only difference between the sites is the set of import assertions.  Future follow-up proposals may relax this restriction with "evaluator attributes" that would change the contents of the module.

There are three possible ways to handle multiple imports of the same module with "evaluator attributes":
- **Race** and use the attribute that was requested by the first import. This seems broken--the second usage is ignored.
- **Reject** the module graph and don't load if attributes differ. This seems bad for composition--using two unrelated packages together could break, if they load the same module with disagreeing attributes.
- **Clone** and make two copies of the module, for the different ways it's transformed. In this case, the attributes would have to be part of the cache key. These semantics would run counter to the intuition that there is just one copy of a module.

It's possible that one of these three options may make sense for a module load, on a case-by-case basis by attribute, but it's worth careful thought before making this choice.

Plumbing-wise, the JavaScript standard would basically be responsible for passing the attributes up to the host environment, which would then decide how to interpret it, within the requirements listed above. Issues [#24](https://github.com/tc39/proposal-import-assertions/issues/24) and [#25](https://github.com/tc39/proposal-import-assertions/issues/25) discuss the Web and Node.js feature and semantic requirements respectively, and issue [#10](https://github.com/tc39/proposal-import-assertions/issues/10) discusses how to allow different JavaScript environments to have interoperability.

## FAQ

### Why not out of band?

Why not both? The champions of this proposal think that exploring both an in- and out of band solutions to various kinds of metadata. While we prefer in-band metadata for module types, we are happy to see the development of various out-of-band manifests of modules being proposed and implemented in certain JS environments:
- [import maps](https://github.com/WICG/import-maps) to map module specifiers to URLs/paths
- [Node.js policy files](https://nodejs.org/api/policy.html), e.g., to perform integrity checks on modules

This proposal does not exclude out-of-band metadata being used for module types. And it definitely doesn't argue that all metadata should be in-band. For example, integrity hashes simply don't work in-band, both because module circularities make them impossible to calculate, and because of the need for a "cascading" update when a deep dependency changes.

Out-of-band solutions face certain downsides; these are not necessarily fatal, but are interesting to take into account when considering the solution space and making tradeoffs:
- **By-hand authoring experience**: While an in-band solution is somewhat verbose, it is also more straightforward for developers to adopt when writing code without much tooling. For smaller projects developers do not need to create an extra file by hand.
- **Tooling complexity for large projects**: For large project with many dependencies, developers will not have to worry about creating a large manifest by compiling the metadata of all of their dependencies. Module authors will also not have to worry about shipping a manifest in order for consumers to be able to run their modules.
- **Performance tradeoffs**: The experience in Node.js's experimental, out-of-band policy files is that they can carry significant startup cost, due to certain aspects of loading and parsing.

### How is common behavior ensured across JavaScript environments?

A central goal of this proposal is to share as much syntax and behavior across JavaScript environments as possible. To the same end, we also [propose](https://github.com/tc39/proposal-json-modules) a standardization of JSON modules to the extent that this is possible (omitting just the contents of the redundant type check, which necessarily differs between environments, in addition to the pre-existing host-defined parts such as interpreting the module specifier and fetching the module).

However, at the same time, behavior of modules in general, and the set of module types specifically, is expected to differ across JavaScript environments. For example, WebAssembly, HTML and CSS modules may not make sense in certain minimal embedded JavaScript environments. We hope that environments can experiment and collaborate where it makes sense for them.

We see the management of compatibility issues across environments as similar, independent of whether metadata is held in-band or out-of-band. An out of band solution would also suffer from the risk of inconsistent implementation or support across host environments if some kind of coordination does not occur.

The topic of attribute divergence is further discussed in  [#34](https://github.com/tc39/proposal-import-assertions/issues/34).

### How would this proposal work with caching?

Assertions are not part of the module cache key. Implementations are required to return the same module, or an error, regardless of the assertions.

### Why not use more terse syntax to indicate module types, like `import json from "./foo.json" as "json"`?

Another option considered and not selected has been to use a single string as the attribute, indicating the type. This option is not selected due to its implication that any particular attribute is special; even though this proposal only specifies the `type` attribute, the intention is to be open to more assertions in the future. (discussion in [#12](https://github.com/tc39/proposal-import-assertions/issues/12)).

### Should more than just strings be supported as attribute values?

We could permit import assertions to have more complex values than simply strings, for example:

```js
import value from "module" assert { attr: { key1: "value1", key2: [1, 2, 3] } };
```

This would allow import assertions to scale to support a larger variety of metadata.

We propose to omit this generalization in the initial proposal, as a key/value list of strings already affords significant flexibility to start, but we're open to a follow-on proposal providing this kind of generalization.

### What are you open to changing? When do we have to settle down on the details?

We are planning to make descisions and reach consensus during specific stages of this proposal. Here's our plan.

#### Before stage 2

We have achieved consensus on the following core decisions as part of Stage 2, including:

- The attribute form; key-value or single string ([#12](https://github.com/tc39/proposal-import-assertions/issues/12))

```mjs
// Not selected
import value from "module" as "json";

// Not selected
import value from "module" with type: "json";

// Current proposal, to settle on before Stage 3
import value from "module" assert { type: "json" };
```

#### Before stage 3

After Stage 2 and before Stage 3, we're open to settling on some less core details, such as:

- Considering alternatives for the `with`/`if`/`assert` keywords ([#3](https://github.com/tc39/proposal-import-assertions/issues/3))

```mjs
import value from "module" when { type: 'json' };
import value from "module" given { type: 'json' };
```

- How dynamic import would accept import assertions:
```mjs
import("foo.wasm", { assert: { type: "webassembly" } });
```

For consistency the `assert` key is used for both dynamic and static imports.

An alternative would be to remove the `assert` nesting in the object:
```mjs
import("foo.wasm", { type: "webassembly" });
```

However, that's not possible with the `Worker` API since it already uses an object with a `type` key as the second parameter. Which would make the APIs inconsistent.

#### Before Stage 4

- The integration of import assertions into various host environments.
    - For example, in the Web Platform, how import assertions would be enabled when launching a worker (if that is supported in the initial version to be shipped on the Web) or included in a `<script>` tag.

```mjs
new Worker("foo.wasm", { type: "module", assert: { type: "webassembly" } });
```

Standardization here would consist of building consensus not just in TC39 but also in WHATWG HTML as well as the Node.js ESM effort and a general audit of semantic requirements across various host environments ([#10](https://github.com/tc39/proposal-import-assertions/issues/10), [#24](https://github.com/tc39/proposal-import-assertions/issues/24) and [#25](https://github.com/tc39/proposal-import-assertions/issues/25)).

## Specification

* [Specification Outline](https://tc39.es/proposal-import-assertions/)
* [Pull Request for HTML spec integration](https://github.com/whatwg/html/pull/5658)
