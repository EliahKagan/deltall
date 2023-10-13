<!-- SPDX-License-Identifier: 0BSD -->

# deltall - dynamic chronicle of `__all__`

`deltall` is a Bash script that checks how the runtime value of a Python
module's `__all__` attribute changes across versions of the PyPI package that
provides the module.

**Running this script has serious security implications.** You should only ever
run it on a package you trust, as it installs multiple versions of the package
and actually runs code from them to obtain the runtime values of `__all__` to
be compared.

This script also has a fairly limited use case, as detailed below. It should
currently be considered to be of prerelease quality, both because it has only
undergone a moderate amount of testing--and because constraining what package
versions are installed, if you choose to do that, can only be done by passing a
regular expression, which is unwieldy. It requires Bash, and also only works on
Unix-like systems. (It is unlikely to work on Windows, even with Bash. But it
works on WSL.)

## License

deltall is licensed under [0BSD](https://spdx.org/licenses/0BSD.html), which is
a [“public-domain
equivalent”](https://en.wikipedia.org/wiki/Public-domain-equivalent_license)
license. See [**`LICENSE`**](LICENSE).

## Use case

The uses case of `deltall` is actually pretty narrow. *All* the following
should hold:

1. **You trust the package, every version of it.** The script installs all the
   versions, and imports a module from them, *running their code*. It uses each
   one in a separate Python virtual environment, but these are not isolated
   from the rest of the system in *any* way relevant to security!
2. You want to know what is listed in `__all__`, which is not necessarily the
   same as what is imported by a `*` import. Imports of the form `from module
   import *` work even when `module.__all__` is absent, in which case they
   import names from module that do not begin with `_`.
3. You want to check `__all__` *dynamically*. For a package where `__all__` is
   always written as a list or tuple of literal strings--which is what should
   almost always be done--it would be sufficient to look at those strings. It
   would be valuable to have a script that does that, for example by using
   Python's `ast` module, and such a script could be written in such a way as
   to be safe even when analyzing untrusted code. But point of this script is
   to assess how the contents of a dynamically defined `__all__`, such as one
   written as a comprehension, has changed, perhaps unintentionally, across
   versions of a package.

In addition, the information you see from this script is influenced by two
major effects:

- Python version dependence. A module's `__all__` can have different contents
  on different versions and even, occasionally, different implementations of
  Python. This is the case even for statically defined `__all__`, but it is
  more unpredictable when dynamic. This script compares `__all__`'s contents
  across versions of the package, but only for the specific Python interpreter
  you specify.
- Dependency skew. When you install a package from PyPI, its dependencies are
  also installed. The versions it requires often advance over time, but it is
  still very common to get much newer or otherwise different versions of
  dependencies when rerunning an experiment in the future. This can sometimes
  affect the contents of `__all__`, especially if `__all__` is dynamically
  generated. The reports this script generates are therefore is *not* real
  historical snapshots of `__all__` from the times when users would most likely
  have installed each of the package's versions--though one might hope that it
  offers a sufficient approximation of them.

Note that only non-prerelease, non-yanked versions are currently tested (see
"How it works" below for details, including why you may well want that
limitation).

## How to use

It's best to run `deltall` in an empty directory you have created for that
purpose. The syntax is:

```sh
deltall <python> <package> [<module>] [<regex>]
```

It's also fine to clone the `deltall` repository itself and run the script from
there. Change `deltall` to `./deltall`.

When you run `deltall`, pass the desired arguments for:

- `<python>`: Python interpreter command (e.g., `python3.9`), *and*
- `<package>`: PyPI package name (e.g., `GitPython`).

Optionally, you can also specify:

- `<module>`: what module's `__all__` should be examined, otherwise a module
  with the same name as the package is assumed, *and*
- `<regex>` a regular expression that must match (part of) the version string,
  otherwise `^`, which matches everything, is assumed.

## How it works

`deltall` creates an "outer" virtual environment using the interpreter you
specify, updates PyPA packages (such as `pip`) in that environment, and then
uses `pip index versions` to find out about all **non-prerelease, non-yanked
versions of the package that are compatible with that interpreter**.

Prereleases are omitted because stability in `__all__` (and otherwise) would
not ordinarily be assumed. Yanked versions are omitted because of the higher
risk of them being uninstallable due to bugs related to dependencies, and
because yanking is sometimes used for versions that pose special risks (e.g.,
accidentally listing a malicious dependency instead of the intended one).

`deltall` then creates a `pypackage.toml` file for a dummy package that lists
your package without a version constraint, and a `tox.ini` file to "test" the
dummy package with one environment per version of its `<package>` dependency.
It installs `tox` in the outer virtual environment and uses it to install each
version, import `<module>` from it, and save the contents of its `__all__`
attribute. This is done for each version in a separate `tox`-managed virtual
environment (separate from each other and also from the outer virtual
environment).

If any `tox` environment failed, the script stops, though partial results can
be examined in the files it produces in the temporary `tox` directories (which
are not deleted automatically). Otherwise, `deltall` reports additions and
removals in `__all__` for each pair of versions, from oldest to newest. This is
written to standard output, as well as to a `report.txt` file in the current
directory.
