# cyrus-config
[![Build Status](https://travis-ci.org/dryewo/cyrus-config.svg?branch=master)](https://travis-ci.org/dryewo/cyrus-config)
[![codecov](https://codecov.io/gh/dryewo/cyrus-config/branch/master/graph/badge.svg)](https://codecov.io/gh/dryewo/cyrus-config)
[![Clojars Project](https://img.shields.io/clojars/v/cyrus/config.svg)](https://clojars.org/cyrus/config)

REPL-friendly config loading library.

```clj
[cyrus/config "0.0.0"]
```

## Usage

*http.clj:*
```clj
(ns my.http
  (:require [cyrus-config.core :as cfg]))

;; Introduce a global constant that will contain validated and transformed value from the environment
;; By default uses variable name transformed from the defined name: "HTTP_PORT"
(cfg/def http-port {:info     "Port to listen on"
                    :spec     int?
                    :required true
                    :default  8080})

;; Available immediately, without additional loading commands, but can contain a special value indicating an error.

(defn start-server []
  (server/start-server {:port http-port}))
```

*core.clj:*
```clj
(ns my.core
  (:require [cyrus-config.core :as cfg]
            [my.http :as http]))

(defn -main [& args]
  ;; Will throw if some configuration variables are invalid
  (cfg/validate!)
  (println "Config loaded:\n" (cfg/show))
  (http/start-server))
```

When started, it will print something like:

```
Config loaded:
#'my.http/port: 8080 from HTTP_PORT in :enironment // Port to listen on
```

### Reference

#### Defining

`(cfg/def foo-bar <parameter-map>)` — defines a constant (like normal `def` does), assigns its value according to parameters.
Additionally, metadata is provided that contains all parameters, raw value, source (`:environment`, `:override`, `:default`) and error.
This metadata is used in `(cfg/show)`.

```clj
;; Assuming that HTTP_PORT environment variable contains "8080"
(cfg/def http-port {:spec int? :required true})
http-port
=> 8080
(meta #'http-port)
=> {::cfg/user-spec      {:spec #object[clojure.core$int_QMARK___5132 ...]}
    ::cfg/effective-spec {:required true
                          :default  nil
                          :secret   false
                          :var-name "HTTP_PORT"
                          :spec     #object[clojure.core$int_QMARK___5132 ...]}
    ::cfg/source         :environment
    ::cfg/raw-value      "8080"
    ::cfg/error          nil
    ...}
```

Parameters (all are optional):

* `:info` — string, description of the variable
* `:var-name` — string, environment variable name to get the value from. Defaults to `"FOO_BAR"` (according to the constant's name).
* `:required` — boolean, if the environment variable is not set, an error will be thrown during `(cfg/validate!)`. The constant
will silently get a special value that indicates an error, for example:
    ```clj
    (cfg/def required-1 {:required true})
    required-1
    => #=(cyrus_config.core.ConfigNotLoaded. {:code    :cyrus-config.core/required-not-present
                                              :message "Required not present"})

    ```
* `:default` — default value to use if the variable is not set. Cannot be used together with `:required true`. Defaults to `nil`.
* `:spec` — Clojure Spec to conform the value to. Defaults to `string?`, can also be `int?`, `keyword?`, `double?` and 
  any complex spec, in which case the original value will be parsed as EDN and then conformed. See Conforming/Coercing section below. 
* `:schema` — Prismatic Schema to coerce the value to (same as `:spec`, but for Prismatic). Complex schemas first parse the value as YAML.
* `:secret` — boolean, if true, the value will not be displayed in the overview returned by `(cfg/show)`.

    ```clj
    (cfg/def bad-config {:spec int? :default "a"})
    bad-config
    => #=(cyrus_config.core.ConfigNotLoaded.
         {:code :cyrus-config.core/invalid-value,
          :value "a",
          :message "java.lang.NumberFormatException: For input string: \"a\""})
    ```

#### Validation

`(cfg/validate!)` — checks if there were any errors during config loading, throws an `ex-info` with their description.
If everything is ok, does nothing.
The output looks like this:

```
               my.core.main                
                        ...                
              my.core/-main  core.clj:   21
              my.core/-main  core.clj:   32
cyrus-config.core/validate!  core.clj:  146
       clojure.core/ex-info  core.clj: 4739
clojure.lang.ExceptionInfo: Errors found when loading config:
                            #'my.http/port: <ERROR> because HTTP_PORT contains "abcd" - java.lang.NumberFormatException: For input string: "abcd" // Port to listen on
```

#### Summary

`(cfg/show)` — returns a formatted string containing information about all defined and loaded configuration constants.
The return value looks like this:

```
#'my.nrepl/bind: "0.0.0.0" from NREPL_BIND in :default // NREPL network interface
#'my.nrepl/port: 55000 from NREPL_PORT in :environment // NREPL port
#'my.db/password: <SECRET> because DB_PASSWORD is not set // Password
#'my.db/username: "postgres" from DB_USERNAME in :default // Username
#'my.db/jdbc-url: "jdbc:postgresql://localhost:5432/postgres" from DB_JDBC_URL in :default // Coordinates of the DB
#'my.authenticator/tokeninfo-url: nil because TOKENINFO_URL is not set // URL to check access tokens against. If not set, tokens won't be checked.
#'my.http/port: 8090 from HTTP_PORT in :environment // Port for HTTP server to listen on
```

It's recommended to print it from `-main`:

```clj
(defn -main [& args]
  ;; By the time -main starts, all the config is already loaded. Here we only find out about errors. 
  (cfg/validate!)
  (println "Config loaded:\n" (cfg/show))
  ...)

```

#### REPL support

`(reload-with-override! <env-map>)` — reloads all configuration constants from a merged source:

    (merge (System/getenv) <env-map>)


For REPL-driven development it's recommended to have it in a wrapper for `(refresh)`:

*dev/user.clj:*
```clj
(defn load-dev-env []
  (edn/read-string (slurp "./dev-env.edn")))

(defn refresh []
  (cfg/reload-with-override! (load-dev-env))
  (cfg/validate!)
  (clojure.tools.namespace.repl/refresh))
```

This will ensure that every time the code is reloaded, the overrides file `dev-env.edn` is also re-read.

### Conforming/Coercion

The library supports two ways of conforming (a.k.a. coercing) environment values (which are always string) to
various types: integer, keyword, double, etc. The ways of defining targer types are Clojure Spec and Prismatic Schema.
They are mutually exclusive, i.e. only one of `:spec` and `:schema` keys are possible at the same time for each configuration constant.

#### Clojure Spec

> [spec] is a Clojure library to describe the structure of data and functions.

It is enabled by setting `:spec` key in the parameters:

```clj
(cfg/def http-port {:spec int?})
```

Basic types are supported: `int?`, `double?`, `keyword?`, `string?` (the default one, does nothing).

Additionally, you can put a complex value in EDN format into the variable:

    IP_WHITELIST='["1.2.3.4" "4.3.2.1"]'

and then conform it:

```clj
(cfg/def ip-whitelist {:spec (s/coll-of string?)})
ip-whitelist
=> ["1.2.3.4" "4.3.2.1"]
;; ^ not a string, a Clojure data structure
```

#### Prismatic Schema

> [Prismatic Schema]: A Clojure(Script) library for declarative data description and validation.

It is enabled by setting `:schema` key in the parameters:

```clj
(cfg/def http-port {:schema s/Int})
```

Also, it's necessary to include `[squeeze "0.3.1]` library in project dependencies to use `:schema` key.

Besides supporting all atomic types (`s/Int`, `s/Num`, `s/Keyword`, etc.), complex types can be coerced:

```
IP_WHITELIST=$(cat <<EOF
- 1.2.3.4
- 4.3.2.1
EOF
)
FOO='foo:\n  bar: 1\n  baz: 42'
```

And then define the config constants as following:

```clj
(cfg/def ip-whitelist {:spec [s/Str]})
ip-whitelist
=> ["1.2.3.4" "4.3.2.1"]

(cfg/def foo {:schema {:foo {:bar                  s/Str 
                             :baz                  s/Int
                             (s/optional-key :opt) s/Keyword}}})
foo
=> {:foo {:bar "1" 
          :baz 42}}

```

Additionally, when using REPL-driven development, you can provide unstringified values in the overrides:

```clj
(cfg/reload-with-override! {"IP_WHITELIST" ["1.2.3.4" "4.3.2.1"]
                            "FOO"          {:foo {:bar "1" 
                                                  :baz 42}}})
```

## Rationale

Let's assume, we are following [12 Factor App] manifesto and read the configuration only from environment variables.

Imagine you have a HTTP server that expects a port:

```clj
(defn start-server []
  (server/start-server {:port (System/getenv "HTTP_PORT")}))
```

But the library expects integer, not string:

```clj
(defn start-server []
  (server/start-server {:port (Integer/parseInt (System/getenv "HTTP_PORT"))}))
```

We need a default value in case when `HTTP_PORT` is not set:

```clj
(defn start-server []
  (server/start-server {:port (or (some-> (System/getenv "HTTP_PORT") (Integer/parseInt)) 8080)}))
```

Environment variables cannot be changed when the process is running. In order to make our application testable
 (for example, to try starting the server on different ports), we need an abstraction layer.
For example, we can read the entire environment to a map and then redefine it for tests:

```clj
(def ^:dynamic environment (System/getenv))

(defn start-server []
  (server/start-server {:port (Integer/parseInt (get environment "HTTP_PORT"))}))

(deftest test-start-server
  (alter-var-root #'environment (constantly {"HTTP_PORT" "7777"})))
  (start-server)
  ...
  (stop-server)
```

This can be further improved by keeping the original environment intact and changing only the overriding:

```clj

(def environment (System/getenv))
(def ^:dynamic env-override {})

(defn reload-overrides! [overrides]
  (alter-var-root #'env-override (constantly overrides)))

(defn get-config [var-name]
  (get (merge environment env-override) var-name))

(defn start-server []
  (server/start-server {:port (Integer/parseInt (get-config "HTTP_PORT")}))
```

If you do REPL-driven development, overrides can be read from a file and applied without restarting the REPL:

```clj
(reload-overrides! (clojure.edn/read-string (slurp "dev-env.edn")))
```

It is a lot of hassle already, while we just wanted to read one simple value and have some basic REPL support.
This solution still has drawbacks:

* If the variable is used in many places, its name has to be repeated, which adds risk of typos ("HTPT_PORT").
* Errors are found late, only when first accessing the value.
* Errors are only given one at a time: if the application fails to start because of one invalid variable, it does not check
  other values which are needed later.
* We might want to have a print a summary of configuration values when the application starts.
    * Some variables should not be printed (passwords, keys, etc.)

This list can be continued, and the requirements are common to many projects, hence this library.

## License

Copyright © 2017 Dmitrii Balakhonskii

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.

[12 Factor App]: https://12factor.net/
[spec]: https://github.com/clojure/spec.alpha
[Prismatic Schema]: https://github.com/plumatic/schema
