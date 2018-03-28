# Code Loading

Julia has two mechanisms for loading code:

1. **Code inclusion:** e.g. `include("source.jl")`. The `include` mechanism allows you to split a single project across multiple source files. It causes the contents of the file `source.jl` to be evaluated in the context where the `include` call occurs, much as if the text of `source.jl` were pasted into that file in place of the `include` expression. If `include("source.jl")` is evaluated multiple times, `source.jl` is parsed and evaluated multiple times. The included path is interpreted relative to the file in which the `include` call occurs, which makes it simple to move an entire project source tree without changing behavior. In the REPL, included paths are interpreted relative to the current working directory.
2. **Package loading:** e.g. `import X` or `using X`. The import mechanism allows you to load packages—i.e. independent, reusable units of Julia code—and access the the resulting `X` module in the context where the `import` or `using` expression occurs. If the same `X` package is imported multiple times in a Julia process, it is only loaded the first time; on subsequent occurrences, the same module is made available in different contexts. It should be noted that `import X` can refer to and load different packages in different contexts: `X` can mean one thing in the main project and a potentially different thing in each of the packages that the main project depends on.

The code inclusion mechanims is quite simple, it merely consists of parsing each expression in the source file and evaluating them each in turn. Package loading is built on top of code inclusion, and is the more subtle and complex of the two mechanisms. Accordingly, the rest of this chapter is about package loading.

## Package Loading

A package is a Julia source tree with a standard layout which provides functionality that can be reused by other Julia projects—both other packages and standalone applications. Julia code access a package with `import` and  `using` statements. When a Julia program evaluates the statements `import X` or `using X` it ensures that the appropriate `X` package is loaded and then makes that package and functionality it exports available in the context where the `import` or `using` statement appeared. To load a package named `X` a Julia process must determine answers to three questions:

1. What does `X` mean in this context?
2. Has this `X` already been loaded?
3. If not, where can this `X` be loaded from?

Understanding how Julia answers these three questions is key to understanding package loading.

### The meaning of `X`

Julia supports federated management of packages. This means that multiple independent parties can publish registries of packages with their names, versions, dependencies and other metadata. Organizations can have private packages and registries, which can depend on a mix of private and public packages. One consequence of federated packaging is that there is no central authority for package names. Different entities can and will use the same name to refer to different packages. This means that the dependenices of a single project, can end up disagreeing about which package a given name refers to. Julia's package loading mechanisms are designed to handle such situations without and issue.

Since the decentralized naming problem is somewhat abstract, it may help to walk through a concrete scenario. Suppose you're developing an application that depends on two packages named `Priv` and `Pub` which are private and public, respectively. When you named `Priv` you weren't aware of any packages by that name. Subsequently, however, a package named `Priv` has been published (which helps manage system priveleges) and the `Pub` package has started to use it. When you next update `Pub` to get new features, your project will depend on two different packages named `Priv`: the original private one and the new public one. In order for your project to continue working, the expression `import Priv` must load different `Priv` packages depending on whether it occurs in your main application's source or in the source of the `Pub` package.

The above scenario brings us to the first question that Julia must determine the answer to when evaluating `import X` (`using X` is the similar so we'll just write `import` from here): *What does `X` mean in this context?* The meaning of `X` is determined by a combination of three things:

1. The module in which the expression is evaluated
2. The load path as determined by the contents of the `LOAD_PATH` global array
3. The contents of each directory in the load path.



If the dependencies of a project needed share a single namespace, this would be a big problem for you if you ever tried to upgrade `Public` since you would then have two different packages named `Private` as dependencies of your project. With a shared package namespace, the the only real recourse is to rename your private package to something else. This situation is not hypthetical—it's been encountered repeatedly in the wild. 

To handle this kind of situation without  is why `import Private` can mean different things in 



here may not have been any public package named `X`, but at some point you want to upgrade 

Julia packages are identified by [universally unique identifiers](https://en.wikipedia.org/wiki/Universally_unique_identifier) (UUIDs), unique 128-bit values that can be generated independently with astronomically probability of collision. When a Julia package is created, a UUID is generated which identifies it persistently across various events, including:

- name changes,
- changes of repository location,
- changes of ownership,
- package forks.

In any situation where the question *"are these the same package?"* needs to be asked, the answer is determined by comparing UUIDs—two Julia source trees are the same package if and only if they have the same UUID. Accordingly, the question *"what is `X` in this context?"* when evaluating `import X` is answered by determining which UUID the name `X` corresponds to in the context in which the import occurs.

Julia package UUIDs serve a similar purpose to Java's reversed domain package names. For example, in Java you might write `import com.sun.net.httpserver.*;` to import all classes from the `com.sun.net.httpserver` package. This package name suggests that the `httpserver` package is maintained by Sun Microsystems, owner of the `sun.com` domain. Of course, Sun Microsystems no longer exists today, having been acquired by Oracle in 2010. The `com.sun` prefix for these packages, however, is permanent and cannot be updated to `com.oracle`. Even if Oracle were to sell the `sun.com` domain, all Java code using the `httpserver` package would still use the `com.sun` prefix. Since UUIDs have no inherent meaning, they avoid this problem—there's no reason to change a UUID since it entails no meaning. However, UUIDs are are hard for humans to remember and tedious to type. Accordingly, Julia's [package manager](https://julialang.org/Pkg3.jl/latest/) keeps UUIDs mostly out of sight. They don't appear in your Julia code, only in project files specifically designed to record the mapping from names to UUIDs in your projects.

Once the name `X` has been mapped to a UUID, answering the question *"has `X` already been loaded?"* is a simple matter of looking up `X`'s UUID in a table of loaded packages. If `X`'s UUID is in the table, then the loaded module associated with the UUID is bound to `X` in the module where the `import X` or `using X` expression is evaluated. If the `X`'s UUID is not in the global package table, then it must be loaded, which requires answering the next qeustion: *"where can `X` be loaded from?"* but with 

for example, but since UUIDs have no meaning, they can handle package renames and even changing the controlling entitity of a package. In the case, for example, Sun Microsystems no longer exists, and `sun.com` redirects to `oracle.com`. 



The load path is a stack of "environments", each of which provides three things:

1. `roots`: a map from top-level names to UUIDs
2. `graph`: a graph of dependencies from 

searched for in the load path, which is controlled by the `LOAD_PATH` global array 