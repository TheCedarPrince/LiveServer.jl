# Main functions

The two main functions exported by the package are
* [`serve`](@ref) and
* [`servedocs`](@ref),

they are discussed in some details here.

## `serve`

The exported [`serve`](@ref) function is the main function of the package.
The basic usage is

```julia-repl
julia> cd("directory/of/website")
julia> serve()
✓ LiveServer listening on http://localhost:8000/ ...
  (use CTRL+C to shut down)
```

which will make the content of the folder available to be viewed in a browser.

You can specify the port that the server should listen to (default is `8000`) as well as the directory to serve (if not the current one) as keyword arguments.
There is also a `verbose` keyword-argument if you want to see messages being displayed on file changes and
connections.

More interestingly, you can optionally specify the `filewatcher` (the only
regular argument) which allows to define what will trigger the messages to the client and ultimately cause the active browser tabs to reload.
By default, it is file modifications that will trigger page reloads but you may want to write your own file watcher to perform additional actions upon file changes or trigger browser reload differently.

Finally, you can specify a function `coreloopfun` which is called continuously while the server is running.
There may be circumstances where adjusting `coreloopfun` helps to complement the tuning of a `FileWatcher`.

See the section on [Extending LiveServer](@ref) for more informations.

## `servedocs`

The exported [`servedocs`](@ref) function is a convenient function derived from `serve`.
The main purpose is to allow Julia package developpers to live-preview their documentation while working on it by coupling `Documenter.jl` and `LiveServer.jl`.

Let's assume the structure of your package looks like

```
.
├── LICENSE.md
├── Manifest.toml
├── Project.toml
├── README.md
├── docs
│   ├── Project.toml
│   ├── build
│   ├── make.jl
│   └── src
│       └── index.md
├── src
│   └── MyPackage.jl
└── test
    └── runtests.jl

```

The standard way of running `Documenter.jl` is to run `make.jl`, wait for completion and then use a standard browser or maybe some third party tool to see the output.

With `servedocs` however, you can edit the `.md` files in your `docs/src` and see the changes being applied directly in your browser which makes writing documentation faster and easier.
To launch it, navigate to `YourPackage.jl/` and simply

```julia-repl
julia> using YourPackage, LiveServer
julia> servedocs()
```

This will execute `make.jl` (a pass of `Documenter.jl`) before live serving the resulting `docs/build` folder with `LiveServer.jl`.
Upon modifying a `.md` file (e.g. updating `docs/src/index.md`), the `make.jl` will be applied and the corresponding changes propagated to active tabs (e.g. a tab watching `http://localhost:8000/index.html`)

!!! note

    The first pass of `Documenter.jl` takes a few seconds to complete, but subsequent passes are quite fast so that the workflow with `Documenter.jl`+`LiveServer.jl` is pretty quick.

    The first pass collects all information in the code (i.e. docstrings), while
    subsequent passes only consider changes in the markdown (`.md`) files. This
    restriction is necessary to achieve a fast update behavior.

### Tracking Changes in Package Docstrings

# Issue with LiveServer not Updating Documenter Files after a Change in a Function's Docstring

When using the LiveServer, if a developer makes a change to one of the documentation files of a package, those changes are instantly updated in the user's browser. 
However, if changes are made to the docstring of a function within a package, then updates to the docstring do not trigger a corresponding update to the live preview of package documentation when using `servedocs`. 

To resolve this issue, one can use the following procedure: 

First, the user must add the [`Revise.jl`](https://github.com/timholy/Revise.jl) dependency to the `Project.toml` file
of the `docs` environment in the package. The following code shows how
to implement this change from the command line located in the root directory 
of the package.

```bash
> julia --project=docs/
> using Pkg
> Pkg.add("Revise")
```

Second, in the `make.jl` file of the package `docs/` folder, the user needs to use 
and activate the `Revise.jl` package at the top of the file. The following code shows how to implement
this change in the file.

```julia
using Revise

Revise.revise()

#=

... Rest of configuration within your make.jl file

=#
```

3. Finally the user should start the `LiveServer` session with an additional 
argument to watch the package `src/` folder for changes. This additional argument
will force `LiveServer` to watch the `src/` folder for changes and re-render
the documentation site once `LiveServer` notices a change.

```julia
using LiveServer
servedocs(include_dirs=["src/"])
```

Once these change are completed, changes to
a function's docstring will trigger a re-rendering of the documentation website
in the user's browser.

### Additional keywords

The `servedocs` function now takes extra keywords which may, in some cases, make your life easier:

* `foldername="docs"`, is the name of folder that contains the documentation, which can be changed if it's different than `docs`.
* `doc_env=false`, if set to true, the `Project.toml` available in `docs/` will be activated (note 1),
* `skip_dir=""`, indicates a directory to skip when looking at the docs folder for change, this can be useful when using packages like Literate or Weave that may generate files inside your `src` folder.

Note that in latter two cases these keywords are there for your convenience but would be best not used. See also the discussion in [this issue](https://github.com/asprionj/LiveServer.jl/issues/85).
In the first case, doing

```
julia --project=docs -e 'using LiveServer; servedocs()'
```

is more robust.

In the second case, it would be best if you made sure  that  all generated files are saved in `docs/build/...`.
