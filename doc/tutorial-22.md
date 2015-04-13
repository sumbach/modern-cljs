# Tutorial 22 - Learning by Contributing (Part 3)

In the [previous tutorial][1] we postponed implementing unit tests
for the revised [Enfocus][2] lib to give more attention
to the generation of its `jar` package. We altered both the
directory layout and some `project.clj` settings until we reached
full control of it. Then we implemented `hello-enfocus`, the simplest
`enfocus`-based project you can imagine. Finally, with the support
of [piggieback][3], we enabled interaction with a bREPL.

In this tutorial we're going to continue improving the
`Enfocus` lib.  We'll apply again the separation of concerns principle
to the `project.clj` file, and then we'll implement a few unit tests
based on the [clojurescript.test][4] lib.

## Preamble

If you did not type by yourself all the changes we made in the cloned
`Enfocus` repo during the [previous tutorial][1], I suggest you to
start working by cloning and branching the following repository:

```bash
cd ~/dev
git clone https://github.com/magomimmo/enfocus.git
cd enfocus
git checkout tutorial-21
git checkout -b tutorial-22
```

## Introduction

As you have already seen, the amount of information to be
managed in `project.clj` to control every aspect of a CLJ/CLJS
project tends to grow with the time spent adding features to the
project itself.

In this tutorial, before implementing the unit tests for
[enfocus][2], we want to find a way to keep `project.clj`
readable by applying some of the things we have learned in
the previous *housekeeping* tutorials (i.e., [Tutorial 18 - Housekeeping][9]
and [Tutorial 19 - Livin' On the Edge][10].

## Divide et Impera

Take a look of the final `project.clj` we ended up with in the
[previous tutorial][1].

```clj
(defproject org.clojars.magomimmo/enfocus "2.1.0-SNAPSHOT"
  :description "DOM manipulation tool for clojurescript inspired by Enlive"
  :url "https://github.com/magomimmo/enfocus/tree/tutorial-20"
  :license {:name "Eclipse Public License - v 1.0"
            :url "http://www.eclipse.org/legal/epl-v10.html"
            :distribution :repo}

  :min-lein-version "2.2.0"

  :source-paths ["src/clj" "src/cljs"]


  :dependencies [[org.clojure/clojure "1.5.1"]
                 [org.clojure/clojurescript "0.0-2069"]
                 [domina "1.0.3-SNAPSHOT"]
                 [org.jsoup/jsoup "1.7.2"]]

  :plugins [[lein-cljsbuild "1.0.0"]]

  :hooks [leiningen.cljsbuild]

  :cljsbuild
  {:crossovers [enfocus.enlive.syntax]
   :crossover-jar true

   :builds {:deploy
            {:source-paths ["src/cljs"]
             ;:jar true ; DON'T DO THIS
             :compiler
             {:output-to "dev-resources/public/js/deploy.js"
              :optimizations :none
              :pretty-print false}}}}

  :profiles {:dev {:resources-paths ["dev-resources"]
                   :test-paths ["test/clj" "test/cljs"]
                   :dependencies [[com.cemerick/piggieback "0.1.2"]
                                  [ring "1.2.1"]
                                  [compojure "1.1.6"]]
                   :plugins [[com.cemerick/clojurescript.test "0.2.1"]]

                   :cljsbuild
                   {:builds {:whitespace
                             {:source-paths ["src/cljs" "test/cljs" "src/brepl"]
                              :compiler
                              {:output-to "dev-resources/public/js/whitespace.js"
                               :optimizations :whitespace
                               :pretty-print true}}

                             :simple
                             {:source-paths ["src/cljs" "test/cljs"]
                              :compiler
                              {:output-to "dev-resources/public/js/simple.js"
                               :optimizations :simple
                               :pretty-print false}}

                             :advanced
                             {:source-paths ["src/cljs" "test/cljs"]
                              :compiler
                              {:output-to "dev-resources/public/js/advanced.js"
                               :optimizations :advanced
                               :pretty-print false}}}
                    :test-commands {"whitespace"
                                    ["phantomjs" :runner "dev-resources/public/js/whitespace.js"]

                                    "simple"
                                    ["phantomjs" :runner "dev-resources/public/js/simple.js"]

                                    "advanced"
                                    ["phantomjs" :runner "dev-resources/public/js/advanced.js"]}}
                   :repl-options {:nrepl-middleware [cemerick.piggieback/wrap-cljs-repl]}
                   :injections [(require '[cljs.repl.browser :as brepl]
                                         '[cemerick.piggieback :as pb])
                                (defn browser-repl []
                                  (pb/cljs-repl :repl-env
                                                (brepl/repl-env :port 9000)))]}})
```

As you can see, the project configuration we need to set as an *enfocus
developer* overwhelms our capacity to take full control of them as an
*enfocus user*. Luckly, the `profiles` feature of `lein` is going to
be very helpful in keeping the view of the project as an
*enfocus developer* separated from its view as an *enfocus user*.

You'll be amazed to discover how easy is to keep the
*enfocus user view* separate from the *enfocus developer view*. But only
if you have been careful to keep them separate in the `:dev` profile from the very beginning.

### The enfocus user view

Create a new file named `profiles.clj` in the main Enfocus project
directory. Open the `project.clj`, cut the whole `:profiles {:dev
{...}}` section and paste it into the new `profiles.clj`
file. Finally, delete the `:profiles` keyword option which is implicitly
added by Leiningen when reading `profiles.clj`.

Here is the resulting `project.clj`:

```clj
(defproject org.clojars.magomimmo/enfocus "2.1.0-SNAPSHOT"
  :description "DOM manipulation tool for clojurescript inspired by Enlive"
  :url "https://github.com/magomimmo/enfocus/tree/tutorial-20"
  :license {:name "Eclipse Public License - v 1.0"
            :url "http://www.eclipse.org/legal/epl-v10.html"
            :distribution :repo}

  :min-lein-version "2.2.0"

  :source-paths ["src/clj" "src/cljs"]


  :dependencies [[org.clojure/clojure "1.5.1"]
                 [org.clojure/clojurescript "0.0-2069"]
                 [domina "1.0.3-SNAPSHOT"]
                 [org.jsoup/jsoup "1.7.2"]]

  :plugins [[lein-cljsbuild "1.0.0"]]

  :hooks [leiningen.cljsbuild]

  :cljsbuild
  {:crossovers [enfocus.enlive.syntax]
   :crossover-jar true

   :builds {:deploy
            {:source-paths ["src/cljs"]
             ;:jar true ; DON'T DO THIS
             :compiler
             {:output-to "dev-resources/public/js/deploy.js"
              :optimizations :none
              :pretty-print false}}}})
```

Much simpler, eh!

### The enfocus developer view

And here is the contents of `profiles.clj`:

```clj
{:dev {:resources-paths ["dev-resources"]
       :test-paths ["test/clj" "test/cljs"]
       :dependencies [[com.cemerick/piggieback "0.1.2"]
                      [ring "1.2.1"]
                      [compojure "1.1.6"]]
       :plugins [[com.cemerick/clojurescript.test "0.2.1"]]

       :cljsbuild
       {:builds {:whitespace
                 {:source-paths ["src/cljs" "test/cljs" "src/brepl"]
                  :compiler
                  {:output-to "dev-resources/public/js/whitespace.js"
                   :optimizations :whitespace
                   :pretty-print true}}

                 :simple
                 {:source-paths ["src/cljs" "test/cljs"]
                  :compiler
                  {:output-to "dev-resources/public/js/simple.js"
                   :optimizations :simple
                   :pretty-print false}}

                 :advanced
                 {:source-paths ["src/cljs" "test/cljs"]
                  :compiler
                  {:output-to "dev-resources/public/js/advanced.js"
                   :optimizations :advanced
                   :pretty-print false}}}
        :test-commands {"whitespace"
                        ["phantomjs" :runner "dev-resources/public/js/whitespace.js"]

                        "simple"
                        ["phantomjs" :runner "dev-resources/public/js/simple.js"]

                        "advanced"
                        ["phantomjs" :runner "dev-resources/public/js/advanced.js"]}}
       :repl-options {:nrepl-middleware [cemerick.piggieback/wrap-cljs-repl]}
       :injections [(require '[cljs.repl.browser :as brepl]
                             '[cemerick.piggieback :as pb])
                    (defn browser-repl []
                      (pb/cljs-repl :repl-env
                                    (brepl/repl-env :port 9000)))]}}
```

The world in which the *enfocus developer* lives is much more complex
as compared to the world in which the *enfocus user* lives and we
don't want an *enfocus user* to even visually perceive that
complexity.

Even though this change was
very easy to make, this is only because we already moved settings for the *enfocus developer* to the
`:dev` profile. The effort we spent in applying the separation of
concerns principle are now paying us back.

Don't forget that you should never set any `:user` option in the *project*
`profiles.clj` file. Also remember that `lein` merges together three
sources of project configurations:

* `profiles.clj` from the `~/.lein/` directory: this file
  should contain `:user` profile settings only;
* `profiles.clj` local to the project: this file should never
  contain any `:user` profile settings;
* `project.clj`: this file should never contain any `:user`
  profile either;

In case of setting conflicts, the content of the local *project* `profiles.clj`
takes precedence over the `project.clj` content which, in turn, takes
precedence over the *user* `~/.lein/profiles.clj` content.

### Light the fire

As usual after making this kind of change we like to verify that everything is
still working as expected.

#### Clean, compile and test

```bash
lein do clean, compile, test
Deleting files generated by lein-cljsbuild.
Compiling ClojureScript.
Compiling "dev-resources/public/js/advanced.js" from ["src/cljs" "test/cljs"]...
...
Successfully compiled "dev-resources/public/js/advanced.js" in 16.323194 seconds.
Compiling "dev-resources/public/js/simple.js" from ["src/cljs" "test/cljs"]...
...
Successfully compiled "dev-resources/public/js/simple.js" in 7.01869 seconds.
Compiling "dev-resources/public/js/whitespace.js" from ["src/cljs" "test/cljs" "src/brepl"]...
...
Successfully compiled "dev-resources/public/js/whitespace.js" in 3.713015 seconds.
Compiling "dev-resources/public/js/deploy.js" from ["src/cljs"]...
Successfully compiled "dev-resources/public/js/deploy.js" in 3.191533 seconds.
Compiling ClojureScript.

lein test enfocus.server

Ran 0 tests containing 0 assertions.
0 failures, 0 errors.
Running all ClojureScript tests.

Testing enfocus.core-test

....
Ran 1 tests containing 1 assertions.
1 failures, 0 errors.
{:test 1, :pass 0, :fail 1, :error 0, :type :summary}
Subprocess failed
```

Good. The only unit test currently defined in the project is a
failing test to remind us that we still have to define
unit tests.

#### Package

Let's now verify if, by moving all that stuff from
`project.clj` to `profiles.clj`, we are still in
control of the generated `jar` package.

```clj
lein jar
Compiling ClojureScript.
Created /Users/mimmo/tmp/enfocus/target/enfocus-2.1.0-SNAPSHOT.jar

jar tvf target/enfocus-2.1.0-SNAPSHOT.jar
    92 Sun Nov 03 09:44:32 CET 2013 META-INF/MANIFEST.MF
  5116 Sun Nov 03 09:44:32 CET 2013 META-INF/maven/org.clojars.magomimmo/enfocus/pom.xml
   165 Sun Nov 03 09:44:32 CET 2013 META-INF/maven/org.clojars.magomimmo/enfocus/pom.properties
  1012 Sun Nov 03 09:44:32 CET 2013 META-INF/leiningen/org.clojars.magomimmo/enfocus/project.clj
  1012 Sun Nov 03 09:44:32 CET 2013 project.clj
     0 Sat Nov 02 11:45:38 CET 2013 enfocus/
 23336 Sat Nov 02 11:45:38 CET 2013 enfocus/core.cljs
  6851 Sat Nov 02 11:45:28 CET 2013 enfocus/effects.cljs
  4247 Sat Nov 02 11:45:38 CET 2013 enfocus/events.cljs
     0 Sat Nov 02 11:45:38 CET 2013 enfocus/enlive/
  1928 Sat Nov 02 11:45:38 CET 2013 enfocus/enlive/syntax.clj
  4386 Sat Nov 02 11:45:28 CET 2013 enfocus/macros.clj
  2103 Sun Nov 03 09:44:32 CET 2013 enfocus/enlive/syntax.cljs
```

Good. Everything is still in its right place. The last thing to
verify is bREPL activation.

#### bREPLing with Enfocus

Issue the following commands at the terminal window from the main
Enfocus project directory:

```clj
lein repl
Compiling ClojureScript.
nREPL server started on port 54609 on host 127.0.0.1
REPL-y 0.2.1
Clojure 1.5.1
    Docs: (doc function-name-here)
          (find-doc "part-of-name-here")
  Source: (source function-name-here)
 Javadoc: (javadoc java-object-or-class-here)
    Exit: Control+D or (exit) or (quit)
 Results: Stored in vars *1, *2, *3, an exception in *e

user=> (require '[enfocus.server :as http])
nil
user=> (http/run)
2013-10-25 20:07:50.503:INFO:oejs.Server:jetty-7.6.8.v20121106
2013-10-25 20:07:50.576:INFO:oejs.AbstractConnector:Started SelectChannelConnector@0.0.0.0:3000
#<Server org.eclipse.jetty.server.Server@2747ebcb>
user=> (browser-repl)
Type `:cljs/quit` to stop the ClojureScript REPL
nil
cljs.user=>
```

Visit the [localhost:3000][5] URL to activate the bREPL connection for
interacting with the JS Engine of your browser.

```clj
cljs.user=> (+ 1 41)
42
cljs.user=> (js/alert "Hello, Enfocus")
nil
cljs.user=> :cljs/quit
:cljs/quit
user=> exit
Bye for now!
```

Not bad so far. Before proceeding to implement any unit tests, I recommend
that you commit the above changes.

```bash
lein clean
git add project.clj profiles.clj
git commit -m "keep the user view of enfocus separate from the developer view"
```

## Unit testing Enfocus

Here we are. After our two-and-a-half-tutorial journey we've finally
reached our destination from which to start a new journey by
implementing unit tests for the `Enfocus` lib.

[Creighton Kirkendall][6] and I discussed how to unit
test Enfocus. At the beginning, we had two completely different views
on this topic.

As `Enfocus` is a live DOM manipulation lib, Creighton worried a lot about
the need to have a visual confirmation of its correctness on any
browser. On the contrary, my obsession with the application of the
separation of concerns principle drove me to look for a way to
separate the unit tests which do not need visual confirmation from those
which eventually need to be visualized and confirmed within a browser.

He's more of a front-end developer; I'm more of a back-end developer and this is a good
opportunity to fortify the `Enfocus` lib by taking the best of the two
worlds.

### Unit testing strategy

My personal unit testing strategy is very simple to explain:

1. start unit testing from the most *independent* namespace of a lib;
2. take a look at the namespaces depending on it and annotate
   usage of any symbol from the *independent* namespace;
3. start unit testing those symbols at the edge cases;
4. extend unit test coverage to all the public symbols of the
   most *independent* namespace by testing the edge cases only;
5. extend unit test coverage to the regular use cases of these
   symbols;
6. move to the next namespace with the same approach.

Steps `4`, `5` and `6` can be interleaved, depending on your
patience in filling in the regular use cases for each public symbol of
a namespace. My patience with unit testing is close to `nil` so most
of the time I thoroughly unit test the edge cases but only very few regular use cases.

### The most independent namespace

The `Enfocus` lib defines five namespaces:

* `enfocus.core`
* `enfocus.effects`
* `enfocus.events`
* `enfocus.enlive.syntax`
* `enfocus.macros`

By looking at the namespace declarations, the most independent one
is `enfocus.enlive.syntax`.

```clj
(ns enfocus.enlive.syntax)
```

The `enfocus.enlive.syntax` namespace is used by the `enfocus.macros`
and `enfocus.core` namespaces.

```clj
(ns enfocus.macros
  (:refer-clojure :exclude ...)
  (:require ...
            [enfocus.enlive.syntax :as syn])
  (...]))
```

```clj
(ns enfocus.core
  (:refer-clojure :exclude ...)
  (:require [enfocus.enlive.syntax :as en]
            ...)
  (:require-macros ...))
```

### Usages of public symbols of the most independent namespace

If you take a look at the `macros.clj` file, which defines the
`enfocus.macros` namespace, and search for usages of symbols from the
`enfocus.enlive.syntax` namespace, you'll discover that it uses the
`convert` symbol only: annotate it.

Go on searching for usages of public symbols from the
`enfocus.enlive.syntax` namespace in the next namespace which is
`enfocus.core`.

Surprise! The `enfocus.core` namespace only uses the `convert`
symbol from the `enfocus.enlive.syntax` namespace. That's very
good for my laziness regarding unit testing, because by following my
unit testing strategy we could limit our testing of the `enfocus.enlive.syntax`
namespace to the `convert` symbol only.

#### Back to CLJX

But wait a minute. Remember that `enfocus.enlive.syntax` is a
*portable* namespace and we should unit test it both in the CLJ
environment (i.e., against a Java VM) and in the CLJS environment
(i.e., against a JS VM).

As explained in
[Tutorial 16 - Better Safe Than Sorry (Part 3)][7], to apply
the DRY principle in the context of unit testing CLJ/CLJS code we need
to add the `cljx` plugin. By now you should already know that we're going
to add it in the `:dev` profile which has been kept in `profiles.clj`, separate from
`project.clj`.

```clj
{:dev {...
       :test-paths ["test/clj" "target/test/clj" "test/cljs" "target/test/cljs"]
       ...
       :plugins [...
                 [com.keminglabs/cljx "0.3.0"]]

       :cljx {:builds [{:source-paths ["test/cljx"]
                        :output-path "target/test/clj"
                        :rules :clj}

                       {:source-paths ["test/cljx"]
                        :output-path "target/test/cljs"
                        :rules :cljs}]}

       :cljsbuild
       {:builds {:whitespace
                 {:source-paths ["src/cljs" "test/cljs" "src/brepl" "target/test/cljs"]
                  ...}

                 :simple
                 {:source-paths ["src/cljs" "test/cljs" "target/test/cljs"]
                  ...}

                 :advanced
                 {:source-paths ["src/cljs" "test/cljs" "target/test/cljs"]
                  ...}}
        ...}
       ...}}
```

First we added `[com.keminglabs/cljx "0.3.0"]` to the `:plugins`
section. Then we configured two rules for it.

The first rule, `:clj`, reads any `.cljx` source file from the
`"test/cljx/"` directory and emits the corresponding `.clj` file in the
`"target/test/clj/"` directory.

The second rule, `:cljs`, does almost the same thing, but instead of
emitting `.clj` files, it emits `.cljs` and saves them to the
`"target/test/cljs/"` directory.

Note that we updated the `:test-paths` as well, by adding both
the newly-created CLJ and CLJS test output paths.
This is because `cljsbuild` does not add to the
Leiningen `:source-paths` and `:test-paths` any CLJS pathnames added
to its own `:source-paths`.

#### Ported Unit Tests

Now that we have added and configured the `cljx` plugin we can start
coding the first unit test for the `convert` portable function, but we
first need to create the `test/cljx/enfocus/` directory.

```bash
mkdir -p test/cljx/enfocus/enlive
```

Now create `syntax_test.cljx` in that directory and declare
its namespace for both CLJ and CLJS by using the `#+clj` and
`#+cljs` feature annotations as we did in
[Tutorial 16 - Better Safe Than Sorry (Part 3)][7].

```clj
#+clj (ns enfocus.enlive.syntax-test
        (:require [clojure.test :refer [deftest are is testing]]
                  [enfocus.enlive.syntax :refer [convert]]))

#+cljs (ns enfocus.enlive.syntax-test
         (:require-macros [cemerick.cljs.test :refer [deftest are is testing]])
         (:require [cemerick.cljs.test]
                   [enfocus.enlive.syntax :refer [convert]]))

(deftest convert-test
  (testing "Unit Test for (convert arg) function\n"
    (testing "Edge Cases\n"
      (are [expected actual] (= expected actual)
           true false))))
```

As you can see, for the moment we have defined a dummy failing unit test as
a placeholder, because we want first to verify if the machinery we set
up is working as expected.

#### Light the fire

As noted in the cited [Tutorial 16][7], the `cljx` plugin can be
hooked to builtin `lein` tasks as we did for the `cljsbuild` plugin, but the
interaction between them produces unexpected results (i.e., double
compilation) when both are hooked to `lein` tasks.

This issue can be easily worked around by explicitly calling `lein
cljx once` before issuing the `lein compile` command which
compiles the CLJS builds. To save some typing we are going to use
the `lein do` chain of commands.

```bash
lein do clean, cljx once, compile
Deleting files generated by lein-cljsbuild.
Rewriting test/cljx to target/test/clj (clj) with features #{clj} and 0 transformations.
Rewriting test/cljx to target/test/cljs (cljs) with features #{cljs} and 1 transformations.
Compiling ClojureScript.
Compiling "dev-resources/public/js/advanced.js" from ["src/cljs" "test/cljs" "target/test/cljs"]...
...
Successfully compiled "dev-resources/public/js/advanced.js" in 16.306388 seconds.
Compiling "dev-resources/public/js/simple.js" from ["src/cljs" "test/cljs" "target/test/cljs"]...
...
Successfully compiled "dev-resources/public/js/simple.js" in 6.617831 seconds.
Compiling "dev-resources/public/js/whitespace.js" from ["src/cljs" "test/cljs" "src/brepl" "target/test/cljs"]...
...
Successfully compiled "dev-resources/public/js/whitespace.js" in 3.968266 seconds.
Compiling "dev-resources/public/js/deploy.js" from ["src/cljs"]...
Successfully compiled "dev-resources/public/js/deploy.js" in 2.889881 seconds.
```

Not bad so far. As you can see, the `cljx` plugin did its rewriting
work and then the `cljsbuild` plugin compiled
down all the defined builds that now include the CLJS files generated
by the `cljx` plugin (at the moment just the `syntax_test.cljs` source
file).

Following is a view of the directory layout produced by the above
chained commands which shows the emitted files in the relevant
directories.

```bash
tree target/cljsbuild-crossover/ target/test/ target/cljsbuild-compiler-0/enfocus/enlive/
target/cljsbuild-crossover/
└── enfocus
    └── enlive
        └── syntax.cljs
target/test/
├── clj
│   └── enfocus
│       └── enlive
│           └── syntax_test.clj
└── cljs
    └── enfocus
        └── enlive
            └── syntax_test.cljs
target/cljsbuild-compiler-0/enfocus/enlive/
├── syntax.js
└── syntax_test.js

8 directories, 5 files
```

Not bad so far. Now let's run the tests.

```bash
lein test
Compiling ClojureScript.

lein test enfocus.enlive.syntax-test

lein test :only enfocus.enlive.syntax-test/convert-test

FAIL in (convert-test) (syntax_test.clj:14)
Unit Test for (convert arg) function
 Edge Cases

expected: (= true false)
  actual: (not (= true false))

Ran 1 tests containing 1 assertions.
1 failures, 0 errors.
Tests failed.
```

Oops. The `lein test` command ran the dummy CLJ unit test but did *not*
run the dummy CLJS unit test.

Again this is an issue related to a bad interaction between the `cljx`
and `cljsbuild` plugins. Luckily, we can work around it by running
the `lein cljsbuild test` subtask explicitly.

> NOTE 2: This bad interaction between `cljx` and `cljsbuild` is caused
> by the failing unit test in the CLJ environment and it disappears if
> you correct the failing test.

```bash
lein cljsbuild test
Compiling ClojureScript.
Running all ClojureScript tests.

Testing enfocus.enlive.syntax-test

FAIL in (convert-test) (:)
Unit Test for (convert arg) function
 Edge Cases

expected: (= true false)
  actual: (not (= true false))

Testing enfocus.core-test

FAIL in (empty-test) (:)
expected: (= 0 1)
  actual: (not (= 0 1))

Ran 2 tests containing 2 assertions.
2 failures, 0 errors.
{:test 2, :pass 0, :fail 2, :error 0, :type :summary}

Testing enfocus.enlive.syntax-test

FAIL in (convert-test) (:)
Unit Test for (convert arg) function
 Edge Cases

expected: (= true false)
  actual: (not (= true false))

Testing enfocus.core-test

FAIL in (empty-test) (:)
expected: (= 0 1)
  actual: (not (= 0 1))

Ran 2 tests containing 2 assertions.
2 failures, 0 errors.
{:test 2, :pass 0, :fail 2, :error 0, :type :summary}

Testing enfocus.enlive.syntax-test

FAIL in (convert-test) (:)
Unit Test for (convert arg) function
 Edge Cases

expected: (= true false)
  actual: (not (= true false))

Testing enfocus.core-test

FAIL in (empty-test) (:)
expected: (= 0 1)
  actual: (not (= 0 1))

Ran 2 tests containing 2 assertions.
2 failures, 0 errors.
{:test 2, :pass 0, :fail 2, :error 0, :type :summary}
Subprocess failed
```

That's much better. The `lein cljsbuild test` command executed two
unit tests for each build. The first failing test is the one we just
defined and the second is the one defined in
[Tutorial 20 - Learning by Contributing (Part 1)][8].

So far so good. It's now time to commit our work.

```bash
git add profiles.clj test/cljx
git commit -m "add cljx plugin and define a new dummy test"
```

#### Be extreme when testing

As I repeated several times, I don't particularly like to write unit tests, but
I have to admit that the implementation of a few unit tests covering the
edge cases can be very useful in clarifying the semantic/behavior of
the function under test. Sometimes you'll find some surprises as
well--the kind of surprises that it's better not to take care of few
months later, when you don't even remember the name of the involved
project.

Before coding the unit test, let's take a look at the `convert`
definition.

```clj
;;; from `enfocus.enlive.syntax` namespace
(defn convert [sel]
  (if (string? sel)
    sel
    (let [ors (sel-to-str sel)]
      (apply str (interpose " " (apply concat (interpose "," ors)))))))
```

OK. It seems that it receives a selector `sel`, and if `sel` is not a
string it converts it into a string, otherwise just returns the input
string. All the conversion stuff is delegated to the `sel-to-str`
function.

This is enough to start coding a few unit tests at the edges, because we
now know that `convert` returns a *string* and it accepts one argument
only which could be a *string* or something reducible to a string
(cf. `(apply str ...)`).

Open `syntax_test.cljx` and substitute the previous failing
test with the following:

```clj
(deftest convert-test
  (testing "Unit Test for (convert arg)\n"
    (testing "Edge Cases\n"
      (testing "(convert a-val)"
        (are [expected actual] (= expected actual)
             ;; extreme values for considered input type
             nil (convert nil)
             ""  (convert "")
             " " (convert " ")
             ""  (convert ())
             ""  (convert [])
             ""  (convert {})
             ""  (convert #{}))))))
```

As expected, return values we chose to have only a *string* or
`nil`. The only case in which we expect the `convert` function to
return `nil` is when the input value is `nil` as well.

#### Light the fire

Aren't you curious about the unit testing results? But wait a minute.

Let me suggest an effective workflow, otherwise you'll go crazy
restarting everything again and again each time you modify a unit
test in the `syntax_test.cljx` file.

Following is my personal workflow (when I want to be agnostic about
my choice of editor/IDE for CLJ/CLJS programming).

> NOTE 3: In a next tutorial we're going to introduce a more advanced
> workflow based on [My Clojure Workflow, reloaded][15] by
> [Stuart Sierra][16].

As said, the undesired interaction between `cljsbuild` and `cljx` is still under
investigation, because there are few things not working as
expected. While waiting for this bug to be fixed, those bad
interactions slow down our workflow. So, take your time.

The complete workflow, depending on the resources of your
development computer, can take minutes, because we defined three
testing builds and they are all recompiled each time you change
the `syntax_test.cljx` source file.

> NOTE 4: If you are in hurry I suggest that you comment out the
> `simple`, the `advanced` builds and the corresponding test commands
> from the `:dev` profile. When you are done implementing the unit tests you can
> uncomment them back.

* In *Terminal 1*: issue the `lein do clean, cljx auto` command. This
  will clean everything and write the `.clj` and `.cljs` files each time
  you save a new version of `syntax_test.cljx`;
* In *Terminal 2*: issue the `lein cljsbuild auto`
  command. It will recompile every build each time a `.cljs` file is
  rewritten by the `cljx` plugin;
* In *Terminal 3*: issue the `lein test` command. Here is where
  something does not work as it should, because the command returns
  after having executed the (failing) `clj` tests and does not
  execute the `cljs` tests;
* In *Terminal 3*: issue the `lein cljsbuild test whitespace`
  command. It executes the `cljs` tests for the `whitespace` build
  only.

Following is the trace of the above commands.

In *Terminal 1*

```bash
lein do clean, cljx auto
Deleting files generated by lein-cljsbuild.
Watching [test/cljx] for changes.
Rewriting test/cljx to target/test/clj (clj) with features #{clj} and 0 transformations.
Rewriting test/cljx to target/test/cljs (cljs) with features #{cljs} and 1 transformations.

```

In *Terminal 2*

```bash
lein cljsbuild auto
Compiling ClojureScript.
...
Successfully compiled "dev-resources/public/js/deploy.js" in 3.170459 seconds.
```

In *Terminal 3*

```bash
lein test
Compiling ClojureScript.

lein test enfocus.enlive.syntax-test

lein test :only enfocus.enlive.syntax-test/convert-test

FAIL in (convert-test) (syntax_test.clj:16)
Unit Test for (convert arg)
 Edge Cases
 (convert a-val)
expected: (= nil (convert nil))
  actual: (not (= nil ""))

Ran 1 tests containing 7 assertions.
1 failures, 0 errors.
Tests failed.
```

> NOTE 5: Because of the issue with the interaction between the
> `cljsbuild` and `cljx` plugins cited above, when you launch the
> `lein test` task, it only runs the CLJ tests.

The last command executed 7 unit tests on the `convert` function. The
one that failed is `(convert nil)` which returned an empty string
`""` instead of the expected `nil`.

We should see the same failed test from the CLJS unit tests.

In *Terminal 3*

```bash
lein cljsbuild test whitespace
Compiling ClojureScript.
Running ClojureScript test: whitespace

Testing enfocus.enlive.syntax-test

FAIL in (convert-test) (:)
Unit Test for (convert arg)
 Edge Cases
 (convert a-val)
expected: (= nil (convert nil))
  actual: (not (= nil ""))

Testing enfocus.core-test

FAIL in (empty-test) (:)
expected: (= 0 1)
  actual: (not (= 0 1))

Ran 2 tests containing 8 assertions.
2 failures, 0 errors.
{:test 2, :pass 6, :fail 2, :error 0, :type :summary}
Subprocess failed
```

As you remember, in the CLJS environment we left a
deliberately failing unit test for the `enfocus.core`
namespace. That's why we got 2 tests instead of 1 and we got 2 failed
assertions instead of 1.

But what about the unit test on `(convert nil)` which failed in both
cases (i.e., CLJ and CLJS)?

#### Logical true and false

In Chapter 3 of [The Joy of Clojure][11] by [Michael Fogus][12]
and [Chris Houser][13] there is a nice discussion about Clojure's *logical*
true and false based on a pragmatic decision by
[Rich Hickey][14] to consider everything **except** `nil` and `false`
to be `true` in a boolean context.

In a Boolean context

```clj
if, if-not, when, when-not, and, or, cond, etc.
```

only `nil` and `false` are logical false,
while `""`, `()`, `[]`, `{}`, `#{}`, and everything else are all logical true.

#### Time to fix by REPLing

One of the best things about writing *portable* CLJ/CLJS code, aside
from freeing you from the need to repeat yourself, is that you can
REPL in CLJ only, which is still a much more comfortable experience
than REPLing in the bREPL.

Let's take a look at the `convert` definition in the REPL.

In *Terminal 3*

```clj
lein repl
Compiling ClojureScript.
nREPL server started on port 50767 on host 127.0.0.1
REPL-y 0.2.1
Clojure 1.5.1
    Docs: (doc function-name-here)
          (find-doc "part-of-name-here")
  Source: (source function-name-here)
 Javadoc: (javadoc java-object-or-class-here)
    Exit: Control+D or (exit) or (quit)
 Results: Stored in vars *1, *2, *3, an exception in *e

user=> (require '[enfocus.enlive.syntax :as syn])
nil
user=> (source syn/convert)
(defn convert [sel]
  (if (string? sel)
    sel
    (let [ors (sel-to-str sel)]
      (apply str (interpose " " (apply concat (interpose "," ors)))))))
nil
user=>
```

As we have already seen, most of the conversion work is performed by
the `sel-to-str` function. Let's try it at the REPL.

```clj
user=> (syn/sel-to-str nil)
nil
user=>
```

This time the return value `nil` adheres to the Clojure choice of
considering `nil` as the only false value beside `false` itself.

Let's fix the `convert` function in the REPL.

```clj
user=> (in-ns 'enfocus.enlive.syntax)
#<Namespace enfocus.enlive.syntax>
enfocus.enlive.syntax=> (defn convert [sel]
                   #_=>   (if (string? sel)
                   #_=>     sel
                   #_=>     (if-let [ors (sel-to-str sel)]
                   #_=>       (apply str (interpose " " (apply concat (interpose "," ors)))))))
#'enfocus.enlive.syntax/convert
enfocus.enlive.syntax=> (in-ns 'user)
#<Namespace user>
user=> (syn/convert nil)
nil
user=>
```

Not bad. We just substituted the `let` call with an `if-let`
call. Nothing simpler than that.

Let's now run the unit tests from the REPL to verify that all the
`convert` unit tests succeed.

```clj
user=> (require '[clojure.test :as ct])
nil
user=> (require '[enfocus.enlive.syntax-test])
nil
user=> (ct/run-tests 'enfocus.enlive.syntax-test)

Testing enfocus.enlive.syntax-test

FAIL in (convert-test) (syntax_test.clj:16)
Unit Test for (convert arg)
 Edge Cases
 (convert a-val)
expected: (= "" (convert ()))
  actual: (not (= "" nil))

FAIL in (convert-test) (syntax_test.clj:16)
Unit Test for (convert arg)
 Edge Cases
 (convert a-val)
expected: (= "" (convert []))
  actual: (not (= "" nil))

FAIL in (convert-test) (syntax_test.clj:16)
Unit Test for (convert arg)
 Edge Cases
 (convert a-val)
expected: (= "" (convert {}))
  actual: (not (= "" nil))

FAIL in (convert-test) (syntax_test.clj:16)
Unit Test for (convert arg)
 Edge Cases
 (convert a-val)
expected: (= "" (convert #{}))
  actual: (not (= "" nil))

Ran 1 tests containing 7 assertions.
4 failures, 0 errors.
{:type :summary, :pass 3, :test 1, :error 0, :fail 4}
user=>
```

Too bad. All the unit tests pertaining to empty collections failed
because they now return `nil` instead of the expected empty string
`""`.

Let's take a look at the `sel-to-str` function internally used by the
`convert` function.

```clj
user=> (source syn/sel-to-str)
(defn sel-to-str [input]
  (let [item (first input)
        rest (rest input)
        end (if (empty? rest) '(()) (sel-to-str rest))]
      (cond
       (keyword? item) (map #(conj % (name item)) end)
       (string? item) (map #(conj % item) end)
       (set? item) (reduce (fn [r1 it]
                             (concat r1 (map #(conj % it) end)))
                           [] (flatten (sel-to-str item)))
       (coll? item) (let [x1 (sel-to-str item)
                          sub (map #(apply str %) (sel-to-str item))]
                      (for [s sub e end]
                        (do (println s e)
                            (conj e s)))))))
nil
user=>
```

WOW! A lot of things are going on here. But even without knowing
anything about the internal details of this function we can
immediately spot the bug.

If the `input` arg is an empty collection, the `item` local var will
be `nil` (cf. `(first [])`). The subsequent `cond` form has no clause
to handle this condition. The final return value of the `sel-to-str`
function will be the result of the `cond`, i.e., `nil` when the `input`
arg is an empty collection.

This behavior is very easy to correct as follows:

```clj
user=> (in-ns 'enfocus.enlive.syntax)
#<Namespace enfocus.enlive.syntax>
enfocus.enlive.syntax=> (defn sel-to-str [input]
                   #_=>   (let [item (first input)
                   #_=>         rest (rest input)
                   #_=>         end (if (empty? rest) '(()) (sel-to-str rest))]
                   #_=>       (cond
                   #_=>        (keyword? item) (map #(conj % (name item)) end)
                   #_=>        (string? item) (map #(conj % item) end)
                   #_=>        (set? item) (reduce (fn [r1 it]
                   #_=>                              (concat r1 (map #(conj % it) end)))
                   #_=>                            [] (flatten (sel-to-str item)))
                   #_=>        (coll? item) (let [x1 (sel-to-str item)
                   #_=>                           sub (map #(apply str %) (sel-to-str item))]
                   #_=>                       (for [s sub e end]
                   #_=>                         (do (println s e)
                   #_=>                             (conj e s))))
                   #_=>        :default input)))
#'enfocus.enlive.syntax/sel-to-str
enfocus.enlive.syntax=>
```

We just added a `:default` clause which returns the passed
`input` when the `cond` doesn't know how to handle the `input`
collection itself.

> NOTE 6: Take into account that if the `input` is not seq-able, the
> `(first input)` and the `(rest input)` in the `let` form will raise an
> exception.

You can immediately test the new definition by issuing the following
expressions at the REPL.

```clj
enfocus.enlive.syntax=> (sel-to-str ())
()
enfocus.enlive.syntax=> (sel-to-str [])
[]
enfocus.enlive.syntax=> (sel-to-str {})
{}
enfocus.enlive.syntax=> (sel-to-str #{})
#{}
enfocus.enlive.syntax=> (convert ())
""
enfocus.enlive.syntax=> (convert [])
""
enfocus.enlive.syntax=> (convert {})
""
enfocus.enlive.syntax=> (convert #{})
""
enfocus.enlive.syntax=>
```

We succeed at the REPL. Let's see if the defined unit tests for the
`convert` function are now working as expected.

```clj
enfocus.enlive.syntax=> (in-ns 'user)
#<Namespace user>
user=> (ct/run-tests 'enfocus.enlive.syntax-test)

Testing enfocus.enlive.syntax-test

Ran 1 tests containing 7 assertions.
0 failures, 0 errors.
{:type :summary, :pass 7, :test 1, :error 0, :fail 0}
```

Great. We can now modify the `syntax.clj` source file as we did at the REPL.

```clj
(ns enfocus.enlive.syntax)
(declare sel-to-string)

(defn sel-to-str [input]
  (let [item (first input)
        rest (rest input)
        end (if (empty? rest) '(()) (sel-to-str rest))]
    (cond
     (keyword? item) (map #(conj % (name item)) end)
     (string? item) (map #(conj % item) end)
     (set? item) (reduce (fn [r1 it]
                           (concat r1 (map #(conj % it) end)))
                         [] (flatten (sel-to-str item)))
     (coll? item) (let [x1 (sel-to-str item)
                        sub (map #(apply str %) (sel-to-str item))]
                     (for [s sub e end]
                         (do (println s e)
                             (conj e s))))
     :default input)))

(defn convert [sel]
  (if (string? sel)
    sel
    (if-let [ors (sel-to-str sel)]
      (apply str (interpose " " (apply concat (interpose "," ors)))))))

;;; follow the rest of the code
```

If you did not stop the running processes in *Terminal 1* and
*Terminal 2*, as soon as you save the above file the `cljx` plugin
starts to regenerate the `CLJ` and `CLJS` version of the modified file
and the `cljsbuild` then starts to rebuild all the CLJS builds.

In the above REPL session we only ran the unit tests for the CLJ
version of the `convert` function. We now want to run those unit tests
on all the builds of the CLJS version of the `convert` function.

In *Terminal 3*

First exit the REPL (or open a new terminal).

```clj
user=> (exit)
```

Then launch the task to run unit test for the CLJS builds.

```bash
lein cljsbuild test
Compiling ClojureScript.
Running all ClojureScript tests.

Testing enfocus.enlive.syntax-test

Testing enfocus.core-test

FAIL in (empty-test) (:)
expected: (= 0 1)
  actual: (not (= 0 1))

Ran 2 tests containing 8 assertions.
1 failures, 0 errors.
{:test 2, :pass 7, :fail 1, :error 0, :type :summary}

Testing enfocus.enlive.syntax-test

Testing enfocus.core-test

FAIL in (empty-test) (:)
expected: (= 0 1)
  actual: (not (= 0 1))

Ran 2 tests containing 8 assertions.
1 failures, 0 errors.
{:test 2, :pass 7, :fail 1, :error 0, :type :summary}

Testing enfocus.enlive.syntax-test

Testing enfocus.core-test

FAIL in (empty-test) (:)
expected: (= 0 1)
  actual: (not (= 0 1))

Ran 2 tests containing 8 assertions.
1 failures, 0 errors.
{:test 2, :pass 7, :fail 1, :error 0, :type :summary}
Subprocess failed
```

As expected the unit tests have been repeated three times, one per
build, and all the `convert` unit tests succeeded in the CLJS
environment too.

As usual, I suggest that you commit the changes.

```bash
git add src test
git commit -m "fix convert and sel-to-str functions"
```

Stay tuned for the next tutorial.

# Next Step - TO BE DONE

TO BE DONE

# License

Copyright © Mimmo Cosenza, 2012-14. Released under the Eclipse Public
License, the same as Clojure.

[1]: https://github.com/magomimmo/modern-cljs/blob/master/doc/tutorial-21.md
[2]: https://github.com/ckirkendall/enfocus
[3]: https://github.com/cemerick/piggieback
[4]: https://github.com/cemerick/clojurescript.test
[5]: http://localhost:3000/
[6]: https://github.com/ckirkendall
[7]: https://github.com/magomimmo/modern-cljs/blob/master/doc/tutorial-16.md
[8]: https://github.com/magomimmo/modern-cljs/blob/master/doc/tutorial-20.md
[9]: https://github.com/magomimmo/modern-cljs/blob/master/doc/tutorial-18.md
[10]: https://github.com/magomimmo/modern-cljs/blob/master/doc/tutorial-19.md
[11]: http://www.joyofclojure.com/
[12]: https://github.com/fogus
[13]: https://github.com/Chouser
[14]: https://github.com/richhickey
[15]: http://thinkrelevance.com/blog/2013/06/04/clojure-workflow-reloaded
[16]: https://github.com/stuartsierra
