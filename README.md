<div align="center">
  <a href="https://github.com/rust-lang/rust">
    <img height="200" alt="binaryen logo" src="https://www.rust-lang.org/logos/rust-logo-blk.svg">
  </a>
  <a href="https://github.com/rollup/rollup">
    <img height="200" alt="webpack logo" src="https://jestjs.io/img/jest.svg">
  </a>
</div>

[![npm][npm]][npm-url]
[![node][node]][node-url]
[![size][size]][size-url]
[![npm][npm-download]][npm-url]
[![deps][deps]][deps-url]
[![tests][tests]][tests-url]

<!-- [![cover][cover]][cover-url] -->

# rs-jest

<sup><sup>tl;dr -- see [examples](#examples)</sup></sup>

This is a jest transformer that loads Rust code so it can be interop with Javascript base project. Currently, the Rust code will be compiled as:

- [x] WebAssembly module/instance
- [ ] Node.js addon/ffi

## Requirements

This module requires a minimum of Node v8.9.0, Jest v23.4.2, and Rust in [nightly channel][].

## Getting Started

To begin, you'll need to install `rs-jest`:

```console
npm install rs-jest --save-dev
```

Then configure Jest to make `rs-jest` to transform the Rust (`*.rs`) file. For example:

<b>jest.config.js</b>

```js
module.exports = {
  transform: {
    "^.+\\.rs$": "rs-jest"
  }
};
```

or if you prefer to put the config in
<b>package.json</b>

```json
  "jest": {
    "transform": {
      "^.+\\.rs$": "rs-jest"
    }
  }
```

<details>
<summary>quick usage</summary>

<b>lib.rs</b>

```rust
#[no_mangle]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

<b>index.js</b>

```js
import wasm from "lib.rs";

export async function increment(a) {
  const { instance } = await wasm;
  return instance.exports.add(1, a);
}
```

</details>

And run `jest` via your preferred method.

## Options

Pretty much like [ts-jest][], you can configure `rs-jest` by using global variables under the `"rs-jest"` key:

<details>
<summary><b><code>export</code></b></summary>

- Type: `string`
- Default: `promise`
- Expected value:
  - `buffer` will export wasm code as [Buffer][]
  - `module` will export wasm code as [WebAssembly.Module][]
  - `instance` will export wasm code as [WebAssembly.Instance][]
  - `async` will [instantiate][webassembly.instantiate] wasm code asynchronously, return promise of both [WebAssembly.Module][] and [WebAssembly.Instance][]
  - `async-module` will [compile][webassembly.compile] wasm code asynchronously, return promise of [WebAssembly.Module][]
  - `async-instance` will [instantiate][webassembly.instantiate] wasm code asynchronously, return promise of [WebAssembly.Instance][]

How wasm code would be exported. (see [examples](#examples))

```json
{
  "globals": {
    "rs-jest": {
      "export": "instance"
    }
  },
  "transform": {
    "^.+\\.rs$": "rs-jest"
  }
}
```

</details>

<details>
<summary><b><code>target</code></b></summary>

- Type: `String`
- Default: `wasm32-unknown-unknown`
- Expected value: see [supported platform](https://forge.rust-lang.org/platform-support.html)

The Rust target to use. Currently it **only support [wasm related target](https://kripken.github.io/blog/binaryen/2018/04/18/rust-emscripten.html)**

```json
{
  "globals": {
    "rs-jest": {
      "target": "wasm32-unknown-emscripten"
    }
  },
  "transform": {
    "^.+\\.rs$": "rs-jest"
  }
}
```

</details>

<details>
<summary><b><code>release</code></b></summary>

- Type: `Boolean`
- Default: `true`

Whether to compile the Rust code in debug or release mode.

```json
{
  "globals": {
    "rs-jest": {
      "release": false
    }
  },
  "transform": {
    "^.+\\.rs$": "rs-jest"
  }
}
```

</details>

## Examples

See the test cases and example projects in [fixtures](./test/fixtures) and [examples](./examples/) for more insight.

<details>
<summary>The exported module are pretty much like <a href="https://github.com/DrSensor/rollup-plugin-rust">rollup-plugin-rust</a> so it can be used alongside with it</summary>

### Given this Rust code

<b>lib.rs</b>

```rust
#[no_mangle]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

<b>Cargo.toml</b>

```toml
[package]
name = "adder"
version = "0.1.0"
authors = ["Full Name <email@site.domain>"]

[lib]
crate-type = ["cdylib"]
path = "lib.rs"
```

### With options

#### `{export: 'buffer'}`

```js
import wasmCode from "./lib.rs";

WebAssembly.compile(wasmCode).then(module => {
  const instance = new WebAssembly.Instance(module);
  console(instance.exports.add(1, 2)); // 3
});
```

---

#### `{export: 'module'}`

```js
import wasmModule from "./lib.rs";

const instance = new WebAssembly.Instance(wasmModule);
console(instance.exports.add(1, 2)); // 3
```

---

#### `{export: 'instance'}`

```js
import wasm from "./lib.rs";

console(wasm.exports.add(1, 2)); // 3
```

---

#### `{export: 'async'}`

```rust
extern {
    fn hook(c: i32);
}

#[no_mangle]
pub fn add(a: i32, b: i32) -> i32 {
    hook(a + b)
}
```

```js
import wasmInstantiate from "./lib.rs";

wasmInstantiate(importObject | undefined).then(({ instance, module }) => {
  console(instance.exports.add(1, 2)); // 3

  // create different instance, extra will be called in different environment
  const differentInstance = new WebAssembly.Instance(module, {
    env: {
      hook: result => result * 2
    }
  });
  console(differentInstance.exports.add(1, 2)); // 6
});
```

---

#### `{export: 'async-instance'}`

```js
import wasmInstantiate from "./lib.rs";

wasmInstantiate(importObject | undefined).then(instance => {
  console(instance.exports.add(1, 2)); // 3
});
```

---

#### `{export: 'async-module'}`

```js
import wasmInstantiate from "./lib.rs";

wasmCompile(importObject | undefined).then(module => {
  const differentInstance = new WebAssembly.Instance(module);
  console(differentInstance.exports.add(1, 2)); // 3
});
```

---

</details>

## Who use this?

- [example-stencil-rust](https://github.com/DrSensor/example-stencil-rust)
- [add yours 😉]

## Contributing

- [CONTRIBUTING.md](./.github/CONTRIBUTING.md) for how you can make contribution
- [HACKING.md](./.github/HACKING.md) for technical details

## Credits

- [rollup-plugin-rust](https://github.com/DrSensor/rollup-plugin-rust)
- [rust-native-wasm-loader](https://github.com/dflemstr/rust-native-wasm-loader)
- [webpack-defaults](https://github.com/webpack-contrib/webpack-defaults)

---

## License

[![FOSSA Status](https://app.fossa.io/api/projects/git%2Bgithub.com%2FDrSensor%2Frs-jest.svg?type=large)](https://app.fossa.io/projects/git%2Bgithub.com%2FDrSensor%2Frs-jest?ref=badge_large)

[ts-jest]: https://github.com/kulshekhar/ts-jest/
[nightly channel]: https://rustwasm.github.io/book/game-of-life/setup.html#the-wasm32-unknown-unknown-target
[buffer]: https://nodejs.org/api/buffer.html
[webassembly.module]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Module
[webassembly.instance]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Instance
[webassembly.instantiate]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/instantiate
[webassembly.compile]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/compile
[npm]: https://img.shields.io/npm/v/rs-jest.svg
[npm-url]: https://npmjs.com/package/rs-jest
[npm-download]: https://img.shields.io/npm/dm/rs-jest.svg
[node]: https://img.shields.io/node/v/rs-jest.svg
[node-url]: https://nodejs.org
[deps]: https://david-dm.org/DrSensor/rs-jest.svg
[deps-url]: https://david-dm.org/DrSensor/rs-jest
[tests]: https://img.shields.io/circleci/project/github/DrSensor/rs-jest.svg
[tests-url]: https://circleci.com/gh/DrSensor/rs-jest
[cover]: https://codecov.io/gh/DrSensor/rs-jest/branch/master/graph/badge.svg
[cover-url]: https://codecov.io/gh/DrSensor/rs-jest
[size]: https://packagephobia.now.sh/badge?p=rs-jest
[size-url]: https://packagephobia.now.sh/result?p=rs-jest
