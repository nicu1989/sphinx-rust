# sphinx-rust

[![PyPI][pypi-badge]][pypi-link]

Sphinx plugin for documentation of Rust projects.

**UNDER DEVELOPMENT!**

See [docs/](docs/) for more information.

[pypi-badge]: https://img.shields.io/pypi/v/sphinx-rust.svg
[pypi-link]: https://pypi.org/project/sphinx-rust

# Fork notes

## Release and Integration

**CURRENTLY FORK UNDER DEVELOPMENT! - PIP PACKAGE UNAVAILABLE**

### Release:

The repo supports releasing github hosted wheels based on tags. This requires choosing a release name(example 0.0.2-dev1) that needs to be updated in order:

* update `version = "0.0.2-dev1"` in `crates/py_binding/Cargo.toml`
```sh
git add .
git commit -m "Bump to 0.0.2-dev1"
git tag v0.0.2-dev1
git push --follow-tags
```
CI starts after `git push --follow-tags`, builds wheels, attaches them to the tag’s GitHub Release. Done.

### Integration:

In the desired `requirements.in` reference the released wheel:

```python
sphinx-rust @ https://github.com/useblocks/sphinx-rust/releases/download/v0.0.2-dev1/sphinx_rust-0.0.2-dev1-cp312-cp312-manylinux_2_17_x86_64.whl
```

## Development

* In order to integrate in score:

Update score/docs/conf.py :

```python
extensions = [
    "sphinx_rust",
    ...
]

# point to a rust crate in score(create a dummy one for testing if unavailable)
rust_crates = [
    "../rust-crates/dummy_crate",
]
```
In index.rst :
```
Rust API <api/crates/dummy_crate/index>
```

* Testing in score repo:

For fast development it is recomended to convert sphinx-rust module to bazel module by creating:

- __sphinx-rust__/BUILD.bazel:

```python
load("@rules_python//python:defs.bzl", "py_library")

py_library(
    name = "sphinx_rust",
    srcs = glob(["python/sphinx_rust/**/*.py"]),
    data = glob([
        "python/sphinx_rust/sphinx_rust.cpython-312-*.so",
    ]),
    imports = ["python"],
    visibility = ["//visibility:public"],
)
```

- __sphinx-rust__/MODULE.bazel

```python
module(name = "sphinx_rust", version = "0.0.0-dev")
bazel_dep(name = "rules_python", version = "1.4.1")
```

Now we can add the __sphinx_rust__ module directly in __score__:

- __score__/MODULE.bazel:

```python
bazel_dep(name = "sphinx_rust", version = "0.0.0-dev")
local_path_override(module_name = "sphinx_rust", path = "../sphinx-rust")
```

- __docs-as-code__/docs.bzl:

```python
def _incremental ...

    py_binary(
        name = incremental_name,
        srcs = ["@score_docs_as_code//src:incremental.py"],
        deps = dependencies + ["@sphinx_rust//:sphinx_rust"],
```

- create the dummy_crate in __score__ with the following structure and content:

```sh
../score/rust-crates/
└── dummy_crate
    ├── Cargo.toml
    └── src
        └── lib.rs
```

Cargo.toml:

```yaml
[package]
name = "dummy_crate"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]

[lib]
name = "dummy_crate"
```

lib.rs:

```rs
pub fn add(left: usize, right: usize) -> usize {
    left + right
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        let result = add(2, 2);
        assert_eq!(result, 4);
    }
}
```

It can be tested using requirements.in from docs-as-code as well but this is not that reliable as bazel will cache sphinx-rust:

- in __docs-as-code__/src/requirements.in:

```python
sphinx-rust @ file:///mnt/d/qorix/forks/sphinx-rust
```

- run pip packages update:

```sh
bazel run //src:requirements.update
```
