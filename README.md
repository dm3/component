# component

'Component' is a tiny Clojure framework for managing the lifecycle of
software components with runtime state which must be manually
initialized and destroyed.

This is primarily a design pattern with a few helper functions. It can
be seen as a style of dependency injection using immutable data
structures.



## Releases and Dependency Information

No releases yet. Run `lein install` in this directory to install the
current SNAPSHOT version.

* Releases will be published to [Clojars]

* Latest stable release is TODO_LINK

* All released versions TODO_LINK

[Leiningen] dependency information:

    [com.stuartsierra/component "0.1.0-SNAPSHOT"]

[Maven] dependency information:

    <dependency>
      <groupId>com.stuartsierra/groupId>
      <artifactId>component</artifactId>
      <version>0.1.0-SNAPSHOT</version>
    </dependency>

[Clojars]: http://clojars.org/
[Leiningen]: http://leiningen.org/
[Maven]: http://maven.apache.org/



## Dependencies and Compatibility

I have successfully tested 'Component' with Clojure versions
1.4.0 and 1.5.1.

'Component' depends on my [dependency] library.

[dependency]: https://github.com/stuartsierra/dependency



## Introduction

For the purposes of this framework, a *component* is a collection of
functions or procedures which share some runtime state.

Some examples of components:

* Database access: query and insert functions sharing a database
  connection

* External API service: functions to send and retrieve data sharing an
  HTTP connection pool

* Web server: functions to handle different routes sharing all the
  runtime state of the web application, such as a session store

* In-memory cache: functions to get and set data in a shared mutable
  reference such as a Clojure Atom or Ref

A *component* is similar in spirit to the definition of an *object* in
Object-Oriented Programming.

Clojure is not an object-oriented programming language, so we do not
have to cram everything into this model. Some (most) functions are
just functions. But real-world applications need to manage state.
Components are a tool to help with that.



## Usage

```clojure
(ns com.example.your-application
  (:require [com.stuartsierra.component :as component]))
```

### Creating Components

To create a component, define a Clojure record that implements the
`Lifecycle` protocol.

```clojure
(defrecord Database [host port connection]
  ;; Implement the Lifecycle protocol
  component/Lifecycle

  (start [component]
    (println ";; Starting database")
    ;; In the 'start' method, initialize this component
    ;; and start it running. For example, connect to a
    ;; database, create thread pools, or initialize shared
    ;; state.
    (let [conn (connect-to-database host port)]
      ;; Return an updated version of the component with
      ;; the run-time state assoc'd in.
      (assoc component :connection conn)))

  (stop [component]
    (println ";; Stopping database")
    ;; In the 'stop' method, shut down the running
    ;; component and release any external resources it has
    ;; acquired.
    (.close connection)
    ;; Return the component, optionally modified.
    component))
```

Optionally, provide a constructor function that takes arguments for
the essential configuration parameters of the component, leaving the
runtime state blank.

```clojure
(defn new-database [host port]
  (map->Database {:host host :port port}))
```

Define the functions implementing the behavior of the
component to take the component itself as an argument.

```clojure
(defn get-user [database username]
  (execute-query (:connection database)
    "SELECT * FROM users WHERE username = ?"
    username))

(defn add-user [database username favorite-color]
  (execute-insert (:connection database)
    "INSERT INTO users (username, favorite_color)"
    username favorite-color))
```

Define other components in terms of the components on which they
depend.

```clojure
(defrecord ExampleComponent [options cache database scheduler]
  component/Lifecycle

  (start [this]
    (println ";; Starting ExampleComponent")
    ;; In the 'start' method, a component may assume that its
    ;; dependencies are available and have already been started.
    (assoc this :admin (get-user database "admin")))

  (stop [this]
    (println ";; Stopping ExampleComponent")
    ;; Likewise, in the 'stop' method, a component may assume that its
    ;; dependencies will not be stopped until AFTER it is stopped.
    this))
```

Not all the dependencies need to be supplied at construction time.
In general, the constructor should not depend on other components
being available or started.

```clojure
(defn example-component [config-options]
  (map->ExampleComponent {:options config-options
                          :cache (atom {})}))
```


### Systems

Components are composed into systems. A system is a component which
knows how to start and stop other components.

A system can use the helper functions `start-system` and `stop-system`,
which take a set of keys naming components in the system to be
started/stopped. Order of the keys doesn't matter here.

```clojure
(def example-system-components [:scheduler :app :db])

(defrecord ExampleSystem [config-options db scheduler app]
  component/Lifecycle
  (start [this]
    (component/start-system this example-system-components))
  (stop [this]
    (component/stop-system this example-system-components)))
```

When constructing the system, specify the dependency relationships
among components with the `using` function.

```clojure
(defn example-system [config-options]
  (let [{:keys [host port]} config-options]
    (map->ExampleSystem
      {:config-options config-options
       :db (new-database host port)
       :scheduler (new-scheduler)
       :app (component/using
              (example-component config-options)
              {:database  :db
               :scheduler :scheduler})})))
```

`using` takes a component and a map telling the system where to
find that component's dependencies. Keys in the map are the keys in
the component record itself, values are the map are the
corresponding keys in the system record.

    {:component-key :system-key}

In the example above:

       (component/using
         (example-component config-options)
         {:database  :db
          :scheduler :scheduler})
    ;;     ^          ^
    ;;     |          |
    ;;     |          \- Keys in the ExampleSystem record
    ;;     |
    ;;     \- Keys in the ExampleComponent record

Based on this information (stored as metadata on the component
records) the `start-system` function will construct a dependency graph
of the components and start them all in the correct order.

Before starting each component, `start-system` will `assoc` its
dependencies based on the metadata provided by `using`.

Optionally, if the keys in the system map are the same as the keys in
the component map, `using` can take a vector of those keys instead of
a map. If you know the names of all the components in your system, you
can add the metadata in the component's constructor:

```clojure
(defrecord AnotherComponent [component-a component-b])

(defrecord AnotherSystem [component-a component-b component-c])

(defn another-component []
  (component/using
    (map->AnotherComponent {})
    [:component-a :component-b]))
```


### Example REPL Session

```clojure
(def system (example-system {:host "dbhost.com" :port 123}))
;;=> #'examples/system

(alter-var-root #'system component/start)
;; Starting database
;; Opening database connection
;; Starting scheduler
;; Starting ExampleComponent
;; execute-query
;;=> #examples.ExampleSystem{ ... }

(alter-var-root #'system component/stop)
;; Stopping ExampleComponent
;; Stopping scheduler
;; Stopping database
;; Closing database connection
;;=> #examples.ExampleSystem{ ... }
```


### Errors

While starting/stopping a system, if any component's `start` or `stop`
method throws an exception, the `start-system` or `stop-system`
function will catch and wrap it in an `ex-info` exception with the
following keys in its `ex-data` map:

* `:system` is the current system, including all the components which
  have already been started.

* `:component` is the component which caused the exception, with its
  dependencies already `assoc`'d in.

The original exception which the component threw is available as
`.getCause` on the exception.

'Component' makes no attempt to recover from errors in a component,
but you can use the system attached to the exception to clean up any
partially-constructed state.

You may find it useful to define your `start` and `stop` methods to be
idempotent, i.e., to have effect only if the component is not already
started or stopped.

```clojure
(defrecord IdempotentDatabaseExample [host port connection]
  component/Lifecycle
  (start [this]
    (if connection  ; already started
      this
      (assoc this :connection (connect host port))))
  (stop [this]
    (if (not connection)  ; already stopped
      this
      (assoc this :connection nil))))
```

'Component' does not require that stop/start be idempotent, but it can
make it easier to clean up state after an error, because you can call
`stop` indiscriminately on everything.

In addition, you could wrap the body of `stop` in a try/catch that
ignores all exceptions.

```clojure
(try ;; ...
  (catch Throwable t
    (log/warn t "Error when stopping component")))
```

That way, errors stopping one component will not prevent other
components from shutting down cleanly.


### Reloading

I developed this pattern in combination with my "reloaded" [workflow].
For development, I might create a `user` namespace like this:

[workflow]: http://thinkrelevance.com/blog/2013/06/04/clojure-workflow-reloaded

```clojure
(ns user
  (:require [com.stuartsierra.component :as component]
            [clojure.tools.namespace.repl :refer (refresh)]
            [examples :as app]))

(def system nil)

(defn init []
  (alter-var-root #'system
    (constantly (app/example-system {:host "dbhost.com" :port 123}))))

(defn start []
  (alter-var-root #'system component/start))

(defn stop []
  (alter-var-root #'system
    (fn [s] (when s (component/stop s)))))

(defn go []
  (init)
  (start))

(defn reset []
  (stop)
  (refresh :after 'user/go))
```


### Production

In the deployed or production version of my application, I typically
have a "main" function that creates and starts the top-level system.

```clojure
(ns com.example.application.main
  (:gen-class)
  (:require [com.stuartsierra.component :as component]
            [examples :as app]))

(defn -main [& args]
  (let [[host port] args]
    (component/start
      (app/example-system {:host host :port port}))))
```


### Usage Notes

The top-level "system" record is intended to be used exclusively for
starting and stopping other components. No component in the system
should depend on its parent system.

I do not intend that application functions should receive the
top-level system as an argument. Rather, functions are defined in
terms of components. Each component receives references only to the
components on which it depends.

The "application" or "business logic" may itself be represented by a
component.

Components may, of course, implement other protocols besides
`Lifecycle`.

Different implementations of a component (for example, a stub version
for testing) can be injected into a system with `assoc` before calling
`start`.

Systems may be composed into larger systems, although I have not yet
found a need for this.


### Notes for Library Authors

'Component' is intended as a tool for application developers, not
library authors. I do not believe that a general-purpose library
should impose any particular framework on the application which uses
it.

That said, libraries can make it trivially easy for application
authors to use their libraries in combination with the 'Component'
pattern by following these guidelines:

* Never create global mutable state (for example, an Atom or Ref
  stored in a `def`).

* Never rely on dynamic binding to convey state (for example, the
  "current" database connection).

* Encapsulate all the runtime state needed by the library in a single
  data structure.

* Provide functions to construct and destroy that data structure.

* Take the encapsulated runtime state as an argument to any library
  functions which depend on it.



## References / More Information

* [tools.namespace](https://github.com/clojure/tools.namespace)
* [On the Perils of Dynamic Scope](http://stuartsierra.com/2013/03/29/perils-of-dynamic-scope).
* [Clojure in the Large](http://www.infoq.com/presentations/Clojure-Large-scale-patterns-techniques) (video)
* [Relevance Podcast Episode 32](http://thinkrelevance.com/blog/2013/05/29/stuart-sierra-episode-032) (audio)
* [My Clojure Workflow, Reloaded](http://thinkrelevance.com/blog/2013/06/04/clojure-workflow-reloaded)
* [reloaded](https://github.com/stuartsierra/reloaded) Leiningen template
* [Lifecycle Composition](http://stuartsierra.com/2013/09/15/lifecycle-composition)



## Change Log

* Version 0.1.0-SNAPSHOT



## Copyright and License

The MIT License (MIT)

Copyright © 2013 Stuart Sierra

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
