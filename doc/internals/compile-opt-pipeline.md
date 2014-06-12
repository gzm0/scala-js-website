---
layout: page
title: Compilation and optimization pipeline
---
{% include JB/setup %}

From Scala source files to optimized JavaScript code, there are a few steps
which are described in this document.

* *compiling*: compile .scala files to 1 .js file per class using the scalac
  compiler with the Scala.js compiler plugin.

For libraries, it stops here. The compiled .js files, their source maps and
the .sjsinfo files are put together with .class files in the binary jars that
can be published to an Ivy or Maven repository.

For the application project, one more mandatory step creates 1 or 3 .js files
consumable by browsers. Then an optional step can be used to optimize the
result.

* *linking* all the .js files together, including those in transitive
  dependencies. This is either one of:
  * *packaging*: concatenate blindly all compiled .js files in 3 big .js files.
  * *preoptimizing*: apply a type-aware inter-method global dead code
    elimination that results in 1 preoptimized .js file.
* *optimizing* (optional, with the
  [Closure Compiler](https://developers.google.com/closure/compiler/)): apply
  the Closure Compiler to the linked .js file(s).

## Compilation

Compilation is similar to using the Scala compiler daily. .scala source files
are compiled with incremental compilation by the Scala compiler. They can
depend on external libraries or other projects in the build, as usual.

The Scala.js compiler plugin produces .js files in addition to the .class files
generated by scalac. Roughly one .js file is produced for each .class file.
(Sometimes, fewer .js files are necessary when the compiler can optimize
anonymous function classes as JavaScript functions.)

For each class, three files are emitted:

* `TheClass.js`: the JavaScript code for the class, its methods, its runtime
  class data, and, for module classes, the module accessor.
* `TheClass.js.map`: the source map linked to `TheClass.js`.
* `TheClass.sjsinfo`: additional information used by the linkers (packaging or
  preoptimizing).

These files are bundled together with .class files in jars published on Ivy or
Maven repositories.

This step completely supports separate compilation and incremental compilation,
just like the regular Scala compiler.

## Packaging

Packaging is the simplest form of linking. Conceptually, it simply takes the
.js files generated by the compilation step for all the transitive dependencies
of the project and concatenates them.
There are however 2 subtleties: a decomposition in 3 files for performance,
and an ordering for correctness.

In sbt, the packaging task is called `packageJS`.

### Decomposition in 3 files

Concatenating everything every time one changes one source file in the project
takes time, because this is a heavily I/O-bound operation. However, typically
most of the code is concentrated in the *external dependencies* of the project
(in sbt terminology, i.e., the libraries referenced in `libraryDependencies`),
and these tend to change rarely (definitely not in a quick edit-compile-reload
cycle). Next, much less code is located in the *internal dependencies*, i.e.,
the other projects in the same build referenced via `dependsOn`). Finally, the
code that changes on every single dev cycle are in the project itself, which
are the *exported products* of the project.

The packaging takes advantage of this obversation by producing 3 .js files
instead of just 1: `app-extdeps.js` contains the external dependencies,
`app-intdeps.js` contains the internal dependencies, and `app.js` contains the
exported products.
`app-extdeps.js` is typically very large (the Scala standard library itself
accounts for more than 20 MB of JavaScript code), but need not be repackaged
often.
`app.js` has to be repackaged on every single dev cycle, but contains
relatively much fewer code, and so this is fast.

### Ordering

The internal structure of the produced .js files imposes a constraint on the
order in which they are packaged. A class must always be packaged after its
superclass (and hence, transitively, all its superclasses).

To do so, the .sjsinfo file contains (among other things), the number of class
ancestors of each class. Since a subclass *A* of a superclass *B* has at least
one more ancestor than *B* (namely, *B*), it is sufficient, to guarantee the
required ordering, that classes with more ancestors are ordered after classes
with less ancestors.

## Preoptimizing

The size of the JavaScript files produced by packaging are huge.
They are always bigger than 20 MB because they include the full Scala standard
library.
But typically, only a small fraction of the libraries you depend on are
actually used.

Preoptimizing is a task that identifies all the classes and methods reachable
from a given set of entry points, and removes everything else.
Entry points are classes and methods that are
[exported to JavaScript]({{ BASE_PATH }}/doc/export-to-javascript.html).

The preoptimizer uses information stored by the compilation step in .sjsinfo
to derive a reachability graph from the entry points.
Then it produces a single .js file containing all the code that is actually
useful (or, since in theory the algorithm computes an over-approximation, all
the code that could not be proved to be useless).

In sbt, the preoptimizing task is called `preoptimizeJS`. The result of
preoptimization is typically between 1.5 MB and 2.5 MB.

The preoptimizer can also be called programmatically using the class
[ScalaJSOptimizer](https://github.com/scala-js/scala-js/blob/master/tools/src/main/scala/scala/scalajs/tools/optimizer/ScalaJSOptimizer.scala)
in the Scala.js toolbox.

## Optimizing using Closure

Scala.js emits code that is always guaranteed to follow the
[restrictions](https://developers.google.com/closure/compiler/docs/limitations)
imposed by the Advanced Optimizations of the Google Closure Compiler.
Hence, we can optimize the linked JavaScript files (either packaged or
preoptimized) using Closure.

In sbt, the optimizing task is called `optimizeJS` and is applied on the result
of `preoptimizeJS` by default. The result optimization is typically between
150 KB and a few hundreds of KB.

The optimizer can also be called programmatically using the class
[ScalaJSClosureOptimizer](https://github.com/scala-js/scala-js/blob/master/tools/src/main/scala/scala/scalajs/tools/optimizer/ScalaJSClosureOptimizer.scala)
in the Scala.js toolbox.