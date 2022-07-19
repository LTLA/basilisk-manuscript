---
title: "basilisk: a Bioconductor package for coordinating Python environments"
tags:
  - R
  - Python
  - Bioconductor
  - bioinformatics
authors:
  - name: Aaron T. L. Lun
    orcid: 0000-0002-3564-4813
    affiliation: 1
affiliations:
  - name: Genentech Inc., South San Francisco, USA
    index: 1
date: 19 July 2022
---

# Summary

# Statement of need

# Usage

A developer of a client package is expected to define one or more `BasiliskEnvironment` objects that describe the Python environments required by the package.
I show an small example below from the [`snifter`](https://bioconductor.org/packages/snifter) Bioconductor package (CITE ME):

```r
snifter.env <- BasiliskEnvironment(
    "fitsne",
    pkgname = "snifter",
    packages = c(
      "opentsne=0.4.3",
      "scikit-learn=0.23.1",
      if (basilisk.utils::isWindows()) "scipy=1.5.0" else "scipy=1.5.1",
      "numpy=1.19.0"
    )
)
```

Once defined, any `BasiliskEnvironment` object can be used in the `basiliskRun()` function to execute arbitrary R code in the context of the associated Python environment.
This is most typically combined with `reticulate` to provide an intuitive developer experience when calling Python from R.
To demonstrate, I'll again use an example from `snifter`, paraphrased for brevity:

```r
out <- basiliskRun(
    env = snifter.env,
    fun = function(x, ...) {
        openTSNE <- reticulate::import("openTSNE", convert = FALSE)
        obj <- openTSNE$TSNE(...)
        out <- obj$fit(x)
        list(
            x = reticulate::py_to_r(out),
            affinities = reticulate::py_to_r(out$affinities$P)
        )
    },
    x = input_matrix # for some observation x dimension matrix.
)
```

# Managing Python environments

`basilisk` uses `conda` to automatically manage the creation of Python environments on the user's device.
Each `conda` environment required by a client package is lazily instantiated on the first call to `basiliskRun()` that uses the corresponding `BasiliskEnvironment` object.
Subsequent uses of that `BasiliskEnvironment` via `basiliskRun()` will then re-use the cached `conda` environment. 

I used `conda` and lazy installation to reduce the burden on the user during installation of client packages.
With `conda`, the user does not have to perform any system configuration such as installing Python or the relevant Python packages.
(Nevertheless, developers of client packages may choose to install further Python packages from `pip` into the `basilisk`-constructed `conda` environment.)
Client packages can also define any number of Python environments, but the use of lazy instantiation means that only the ones that are used will actually be created on the user's machine.
Similarly, if a client package only uses Python for some optional functionality, the cost of installation is only paid when that functionality is requested.

Lazy instantiation involves the construction of a user-owned cache of `conda` environments.
These environments can consume a large amount of disk space, so `basilisk` will automatically remove old environments that have not been used in the past month.
Some care is also taken to ensure that cache management is thread-safe -
if multiple processes attempt to instantiate a particular environment, only one will proceed while the others will wait for its completion.

In some scenarios, it is preferable to pay the environment instantiation cost during client package installation. 
This avoids any delay on first use of `basiliskRun()` within the client package, which provides more predictable end-user experiences for R-based applications like Shiny.
To do this, administrators of an R installation can set the `BASILISK_SYSTEM_DIR` environment variable, which will cause the `conda` environments to be created in the client package's installation directory.
This "system-wide" installation is also useful on shared systems where a single environment is provisioned for any number of users, rather than requiring each user to create and cache their own.

# Integrating with `reticulate`

`basilisk` supports the use of `reticulate` with multiple Python environments in a single R session.
This involves some work as only one Python instance can be loaded by `reticulate` for the lifetime of an R session (see comments [here](https://github.com/rstudio/reticulate/issues/27)).
`basiliskRun()` will first check whether a Python instance has already been loaded into the current R session.
If not, it will load Python from the requested environment into the current session before executing the developer-supplied function in `fun`=.
Otherwise, it will spin up a new R process to run `fun=` before transferring the results back to the current session.

This strategy ensures that a `basilisk` client package will always be able to successfully execute its Python-related code via `reticulate`.
The client package remains functional even if other packages - or indeed, the user themselves - load a different Python instance into the current session.
In fact, client packages can be forced to always start a new process in `basiliskRun()` by turning off the `getBasiliskShared()` option,
which ensures that `basilisk` does not interfere with other uses of Python in the current session.
However, this robustness comes at the cost of performance due to the need to spin up a new R process (with the associated delay from package loading) as well as the overhead of communication between different R processes.
As such, loading of Python into the current session is preferred by default. 

It is also possible to obtain the path to the environment's directory for execution of Python code outside of `reticulate`.
This is more onerous but allows clients to directly call executables that are provided in the environment. 
For example, the [`crisprScore`](https://bioconductor.org/packages/crisprScore) package relies on Python 2 environments that will [no longer be supported by `reticulate`](https://github.com/rstudio/reticulate/pull/1242);
by directly acquiring the path to the environment, the Python 2 executable can be located and used to execute legacy code.

# Acknowledgements

# References

