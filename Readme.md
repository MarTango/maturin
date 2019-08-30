# Maturin

_formerly pyo3-pack_

[![Linux and Mac Build Status](https://img.shields.io/travis/PyO3/maturin/master.svg?style=flat-square)](https://travis-ci.org/PyO3/maturin)
[![Windows Build status](https://img.shields.io/appveyor/ci/konstin/maturin/master.svg?style=flat-square)](https://ci.appveyor.com/project/konstin/maturin/branch/master)
[![FreeBSD](https://img.shields.io/cirrus/github/PyO3/maturin?style=flat-square&task=freebsd)](https://cirrus-ci.com/github/PyO3/maturin)
[![Crates.io](https://img.shields.io/crates/v/maturin.svg?style=flat-square)](https://crates.io/crates/maturin)
[![PyPI](https://img.shields.io/pypi/v/maturin.svg?style=flat-square)](https://pypi.org/project/maturin/)
[![Chat on Gitter](https://img.shields.io/gitter/room/nwjs/nw.js.svg?style=flat-square)](https://gitter.im/PyO3/Lobby)

Build and publish crates with pyo3, rust-cpython and cffi bindings as well as rust binaries as python packages.

This project is meant as a zero configuration replacement for [setuptools-rust](https://github.com/PyO3/setuptools-rust) and [milksnake](https://github.com/getsentry/milksnake). It supports building wheels for python 3.5+ on windows, linux, mac and freebsd, can upload them to [pypi](https://pypi.org/) and has basic pypy support.

## Usage

You can either download binaries from the [latest release](https://github.com/PyO3/maturin/releases/latest) or install it with pip:

```shell
pip install maturin
```

There are three main commands:

 * `maturin publish` builds the crate into python packages and publishes them to pypi.
 * `maturin build` builds the wheels and stores them in a folder (`target/wheels` by default), but doesn't upload them.
 * `maturin develop` builds the crate and install it's as a python module directly in the current virtualenv.

`pyo3` and `rust-cpython` bindings are automatically detected, for cffi or binaries you need to pass `-b cffi` or `-b bin`. maturin doesn't needs extra configuration files and doesn't clash with an existing setuptools-rust or milksnake configuration. You can even integrate it with testing tools such as [tox](https://tox.readthedocs.io/en/latest/). There are examples for the different bindings in the `test-crates` folder.

The name of the package will be the name of the cargo project, i.e. the name field in the `[package]` section of Cargo.toml. The name of the module, which you are using when importing, will be the `name` value in the `[lib]` section (which defaults to the name of the package). For binaries it's simply the name of the binary generated by cargo.

## pyo3 and rust-cpython

For pyo3 and rust-cpython, maturin can only build packages for installed python versions. On linux and mac, all python versions in `PATH` are used. If you don't set your own interpreters with `-i`, a heuristic is used to search for python installations. On windows all versions from the python launcher (which is installed by default by the python.org installer) and all conda environments except base are used. You can check which versions are picked up with the `list-python` subcommand.

pyo3 will set the used python interpreter in the environment variable `PYTHON_SYS_EXECUTABLE`, which can be used from custom build scripts.

## Cffi

 Cffi wheels are compatible with all python versions, but they need to have `cffi` installed for the python used for building (`pip install cffi`).

maturin will utilize cbindgen to generate a header file. To customize the header file you can either configure cbindgen through a cbindgen.toml file inside your project root or write a setup a build script which writes a header file to `$PROJECT_ROOT/target/header.h`. 

Based on these header file maturin is going to generated a module which exports a `ffi` and a `lib` object.

<details>
<summary>Example of a custom build script</summary>

```rust
use cbindgen; // Use `extern crate cbindgen` in rust 2015
use std::env;
use std::path::Path;

fn main() {
    let crate_dir = env::var("CARGO_MANIFEST_DIR").unwrap();

    let bindings = cbindgen::Builder::new()
        .with_no_includes()
        .with_language(cbindgen::Language::C)
        .with_crate(crate_dir)
        .generate()
        .unwrap();
    bindings.write_to_file(Path::new("target").join("header.h"));
}
```

</details>

## Mixed rust/python projects

To create a mixed rust/python project, create a folder with your module name (i.e. `lib.name` in Cargo.toml) next to your Cargo.toml and add your python sources there:

```
my-project
├── Cargo.toml
├── my_project
│   ├── __init__.py
│   └── bar.py
├── Readme.md
└── src
    └── lib.rs
```

maturin will add the native extension as a module in your python folder. When using develop, maturin will copy the native library and for cffi also the glue code to your python folder. You should add those files to your gitignore.

With cffi you can do `from .my_project import lib` and then use `lib.my_native_function`, with pyo3/rust-cpython you can directly `from .my_project import my_native_function`.

Example layout with pyo3 after `maturin develop`:

```
my-project
├── Cargo.toml
├── my_project
│   ├── __init__.py
│   ├── bar.py
│   └── my_project.cpython-36m-x86_64-linux-gnu.so
├── Readme.md
└── src
    └── lib.rs
```

## Python metadata

To specifiy python dependecies, add a list `requires-dist` in a `[package.metadata.maturin]` section in the Cargo.toml. This list is equivalent to `install_requires` in setuptools:

```toml
[package.metadata.maturin]
requires-dist = ["flask~=1.1.0", "toml==0.10.0"]
```

Pip allows adding so called console scripts, which are shell commands that execute some function in you program. You can add console scripts in a section `[package.metadata.maturin.scripts]`. The keys are the script names while the values are the path to the function in the format `some.module.path:class.function`, where the `class` part is optional. The function is called with no arguments. Example:

```toml
[package.metadata.maturin.scripts]
get_42 = "my_project:DummyClass.get_42"
```

You can also specify [trove classifiers](https://pypi.org/classifiers/) in your Cargo.toml under `package.metadata.maturin.classifier`:

```toml
[package.metadata.maturin]
classifier = ["Programming Language :: Python"]
```

You can use other fields from the [python core metadata](https://packaging.python.org/specifications/core-metadata/) in the `[package.metadata.maturin]` section, specifically ` maintainer`, `maintainer-email` and `requires-python` (string fields), as well as `requires-external`, `project-url` and `provides-extra` (lists of strings).

## pyproject.toml

maturin supports building through pyproject.toml. To use it, create a `pyproject.toml` next to your `Cargo.toml` with the following content:

```toml
[build-system]
requires = ["maturin"]
build-backend = "maturin"
```

If a `pyproject.toml` with a `[build-system]` entry is present, maturin will build a source distribution (sdist) of your package, unless `--no-sdist` is specified. The source distribution will contain the same files as `cargo package`. To only build a source distribution, pass `--interpreter` without any values.

You can then e.g. install your package with `pip install .`. With `pip install . -v` you can see the output of cargo and maturin.

You can use the options `manylinux`, `skip-auditwheel`, `bindings`, `strip`, `cargo-extra-args` and `rustc-extra-args` under `[tool.maturin]` the same way you would when running maturin directly.  The `bindings` key is required for cffi and bin projects as those can't be automatically detected. Currently, all build are in release mode (see [this thread](https://discuss.python.org/t/pep-517-debug-vs-release-builds/1924) for details).

For a non-manylinux build you could with cffi bindings you could use the following:

```toml
[build-system]
requires = ["maturin"]
build-backend = "maturin"

[tool.maturin]
bindings = "cffi"
manylinux = "off"
```

Using tox with build isolation is currently blocked by a tox bug ([tox-dev/tox#1344](https://github.com/tox-dev/tox/issues/1344)). There's a `cargo sdist` command for only building a source distribution as workaround for [pypa/pip#6041](https://github.com/pypa/pip/issues/6041).

## Manylinux and auditwheel

For portability reasons, native python modules on linux must only dynamically link a set of very few libraries which are installed basically everywhere, hence the name manylinux. The pypa offers a special docker container and a tool called [auditwheel](https://github.com/pypa/auditwheel/) to ensure compliance with the [manylinux rules](https://www.python.org/dev/peps/pep-0513/#the-manylinux1-policy).

maturin contains a reimplementation of a major part of auditwheel automatically checking the generated library. If you want to disable those checks or build for native linux target, use the `--manylinux` flag.

For full manylinux compliance you need to compile in a cent os 5 docker container. The [konstin2/maturin](https://hub.docker.com/r/konstin2/maturin) image is based on the official manylinux image. You can use it like this:

```
docker run --rm -v $(pwd):/io konstin2/maturin build
```

maturin itself is manylinux compliant when compiled for the musl target. The binaries on the release pages have additional keyring integration (through the `password-storage` feature), which is not manylinux compliant.

## PyPy

maturin can build wheels for pypy with pyo3. Note that pypy [is not compatible with manylinux1](https://github.com/antocuni/pypy-wheels#why-not-manylinux1-wheels) and you can't publish pypy wheel to pypi pypy has been only tested manually and on linux. See [#115](https://github.com/PyO3/maturin/issues/115) for more details.

### Build

```
FLAGS:
    -h, --help               
            Prints help information

        --no-sdist           
            Don't build a source distribution

        --release            
            Pass --release to cargo

        --skip-auditwheel    
            [deprecated, use --manylinux instead] Don't check for manylinux compliance

        --strip              
            Strip the library for minimum file size

    -V, --version            
            Prints version information


OPTIONS:
    -b, --bindings <bindings>
            Which kind of bindings to use. Possible values are pyo3, rust-cpython, cffi and bin

        --cargo-extra-args <cargo-extra-args>...
            Extra arguments that will be passed to cargo as `cargo rustc [...] [arg1] [arg2] --`
            
            Use as `--cargo-extra-args="--my-arg"`
    -i, --interpreter <interpreter>...
            The python versions to build wheels for, given as the names of the interpreters. Uses autodiscovery if not
            explicitly set.
        --manylinux <manylinux>
            Control the platform tag on linux.
            
            - `1`: Use the manylinux1 tag and check for compliance
             - `1-unchecked`: Use the manylinux1 tag without checking for compliance
             - `2010`: Use the manylinux2010 tag and check for compliance
             - `2010-unchecked`: Use the manylinux1 tag without checking for compliance
             - `off`: Use the native linux tag (off)
            
            This option is ignored on all non-linux platforms [default: 1]  [possible values: 1, 1-unchecked, 2010,
            2010-unchecked, off]
    -o, --out <out>
            The directory to store the built wheels in. Defaults to a new "wheels" directory in the project's target
            directory
    -m, --manifest-path <path>                      
            The path to the Cargo.toml [default: Cargo.toml]

        --rustc-extra-args <rustc-extra-args>...
            Extra arguments that will be passed to rustc as `cargo rustc [...] -- [arg1] [arg2]`
            
            Use as `--rustc-extra-args="--my-arg"`
        --target <triple>                           
            The --target option for cargo
```

### Publish

```
FLAGS:
        --debug              
            Do not pass --release to cargo

    -h, --help               
            Prints help information

        --no-sdist           
            Don't build a source distribution

        --no-strip           
            Strip the library for minimum file size

        --skip-auditwheel    
            [deprecated, use --manylinux instead] Don't check for manylinux compliance

    -V, --version            
            Prints version information


OPTIONS:
    -b, --bindings <bindings>
            Which kind of bindings to use. Possible values are pyo3, rust-cpython, cffi and bin

        --cargo-extra-args <cargo-extra-args>...
            Extra arguments that will be passed to cargo as `cargo rustc [...] [arg1] [arg2] --`
            
            Use as `--cargo-extra-args="--my-arg"`
    -i, --interpreter <interpreter>...
            The python versions to build wheels for, given as the names of the interpreters. Uses autodiscovery if not
            explicitly set.
        --manylinux <manylinux>
            Control the platform tag on linux.
            
            - `1`: Use the manylinux1 tag and check for compliance
             - `1-unchecked`: Use the manylinux1 tag without checking for compliance
             - `2010`: Use the manylinux2010 tag and check for compliance
             - `2010-unchecked`: Use the manylinux1 tag without checking for compliance
             - `off`: Use the native linux tag (off)
            
            This option is ignored on all non-linux platforms [default: 1]  [possible values: 1, 1-unchecked, 2010,
            2010-unchecked, off]
    -o, --out <out>
            The directory to store the built wheels in. Defaults to a new "wheels" directory in the project's target
            directory
    -p, --password <password>
            Password for pypi or your custom registry. Note that you can also pass the password through MATURIN_PASSWORD

    -m, --manifest-path <path>                      
            The path to the Cargo.toml [default: Cargo.toml]

    -r, --repository-url <registry>
            The url of registry where the wheels are uploaded to [default: https://upload.pypi.org/legacy/]

        --rustc-extra-args <rustc-extra-args>...
            Extra arguments that will be passed to rustc as `cargo rustc [...] -- [arg1] [arg2]`
            
            Use as `--rustc-extra-args="--my-arg"`
        --target <triple>                           
            The --target option for cargo

    -u, --username <username>                       
            Username for pypi or your custom registry
```

### Develop

```
FLAGS:
    -h, --help       
            Prints help information

        --release    
            Pass --release to cargo

        --strip      
            Strip the library for minimum file size

    -V, --version    
            Prints version information


OPTIONS:
    -b, --binding-crate <binding-crate>
            Which kind of bindings to use. Possible values are pyo3, rust-cpython, cffi and bin

        --cargo-extra-args <cargo-extra-args>...
            Extra arguments that will be passed to cargo as `cargo rustc [...] [arg1] [arg2] --`
            
            Use as `--cargo-extra-args="--my-arg"`
    -m, --manifest-path <manifest-path>             
            The path to the Cargo.toml [default: Cargo.toml]

        --rustc-extra-args <rustc-extra-args>...
            Extra arguments that will be passed to rustc as `cargo rustc [...] -- [arg1] [arg2]`
            
            Use as `--rustc-extra-args="--my-arg"`
```

## Code

The main part is the maturin library, which is completely documented and should be well integratable. The accompanying `main.rs` takes care username and password for the pypi upload and otherwise calls into the library.

The `sysconfig` folder contains the output of `python -m sysconfig` for different python versions and platform, which is helpful during development.

You need to install `cffi` (`pip install cffi`) to run the tests.

There are two optional hacks that can speed up the tests (over 80s to 17s on my machine). By running `cargo build --release --manifest-path test-crates/cargo-mock/Cargo.toml` you can activate a cargo cache avoiding to rebuild the pyo3 test crates with every python vesion. Delete `target/test-cache` to clear the cache (e.g. after changing a test crate) or remove `test-crates/cargo-mock/target/release/cargo` to deactive it. By running the tests with the `faster-tests` feature, binaries are stripped and wheels are only stored and not compressed.

You might want to have look into my [blog post](https://blog.schuetze.link/2018/07/21/a-dive-into-packaging-native-python-extensions.html) which explains the intricacies of building native python packages.
