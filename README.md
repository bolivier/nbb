# Nbb

Not [babashka](https://babashka.org/). Node.js babashka!?

Ad-hoc CLJS scripting on Node.js.

## Status

Experimental.

## Goals

Nbb's main goal is to make it _easy_ to get started with ad hoc CLJS scripting
on Node.js.

Additional goals are:

- Fast startup without relying on a custom version of Node.js.
- Small artifact (current size is around 1.2MB).
- First class [macros](#macros)
- Support building small TUI apps using [Reagent](#reagent).
- Complement [babashka](https://babashka.org/) with libraries from the Node.js ecosystem.

## Community

- Submit bugs at [Github Issues](https://github.com/borkdude/nbb/issues).
- Join [Github Discussions](https://github.com/borkdude/nbb/discussions/categories) for proposing ideas, show and tell and Q&A.
-  Join the [channel](https://app.slack.com/client/T03RZGPFR/C029PTWD3HR) on Clojurians Slack.
- Follow news or tweet using the Twitter hashtag
  [#nbbjs](https://twitter.com/hashtag/nbbjs?f=live).

## Requirements

Nbb requires Node.js v12 or newer.

## Usage

Install `nbb` from NPM:

```
$ npm install nbb -g
```

Omit `-g` for a local install.

Try out an expression:

``` clojure
$ nbb -e '(+ 1 2 3)'
6
```

And then install some other NPM libraries to use in the script. E.g.:

```
$ npm install csv-parse
$ npm install shelljs
```

Create a script which uses the NPM libraries:

``` clojure
(ns script
  (:require ["csv-parse/lib/sync" :as csv-parse]
            ["fs" :as fs]
            ["shelljs" :as sh]))

(println (count (str (fs/readFileSync "script.cljs"))))

(prn (sh/ls "."))

(prn (csv-parse "foo,bar"))
```

Call the script:

```
$ nbb script.cljs
264
#js ["CHANGELOG.md" "README.md" "bb.edn" "deps.edn" "main.js" "node_modules" "out" "package-lock.json" "package.json" "shadow-cljs.edn" "src" "test.cljs"]
#js [#js ["foo" "bar"]]
```

## Macros

Nbb has first class support for macros: you can define them right inside your `.cljs` file, like you are used to from JVM Clojure. Consider the `plet` macro to make working with promises more palatable:


``` clojure
(defmacro plet
  [bindings & body]
  (let [binding-pairs (reverse (partition 2 bindings))
        body (cons 'do body)]
    (reduce (fn [body [sym expr]]
              (let [expr (list '.resolve 'js/Promise expr)]
                (list '.then expr (list 'clojure.core/fn (vector sym)
                                        body))))
            body
            binding-pairs)))
```

Using this macro we can look async code more like sync code. Consider this puppeteer example:

``` clojure
(-> (.launch puppeteer)
      (.then (fn [browser]
               (-> (.newPage browser)
                   (.then (fn [page]
                            (-> (.goto page "https://clojure.org")
                                (.then #(.screenshot page #js{:path "screenshot.png"}))
                                (.catch #(js/console.log %))
                                (.then #(.close browser)))))))))
```

Using `plet` this becomes:

``` clojure
(plet [browser (.launch puppeteer)
       page (.newPage browser)
       _ (.goto page "https://clojure.org")
       _ (-> (.screenshot page #js{:path "screenshot.png"})
             (.catch #(js/console.log %)))]
      (.close browser))
```

## Reagent

Nbb includes `reagent.core` which will be lazily loaded when required. You
can use this together with [ink](https://github.com/vadimdemedes/ink) to create
a TUI application:

```
$ npm install ink
```

`ink-demo.cljs`:
``` clojure
(ns ink-demo
  (:require ["ink" :refer [render Text]]
            [reagent.core :as r]))

(defonce state (r/atom 0))

(doseq [n (range 1 11)]
  (js/setTimeout #(swap! state inc) (* n 500)))

(defn hello []
  [:> Text {:color "green"} "Hello, world! " @state])

(render (r/as-element [hello]))
```

<img src="img/ink.gif"/>

## Startup time

``` clojure
$ time nbb -e '(+ 1 2 3)'
6
nbb -e '(+ 1 2 3)'   0.17s  user 0.02s system 109% cpu 0.168 total
```

The baseline startup time for a script is about 170ms seconds on my laptop. When
invoked via `npx` this adds another 300ms or so, so for faster startup, either
use a globally installed `nbb` or use `$(npm bin)/nbb script.cljs` to bypass
`npx`.

## How does this tool work?

CLJS code is evaluated through [SCI](https://github.com/borkdude/sci), the same
interpreter that powers [babashka](https://babashka.org/). Because SCI works
with advanced compilation, the bundle size, especially when combined with other
dependencies, is smaller than what you get with self-hosted CLJS. That makes
startup faster. The trade-off is that execution is less performant and that only
a subset of CLJS is available (e.g. no deftype, yet).

## Optional dependencies

Nbb depends on React to load the optional [Reagent](#reagent) module. To not
download react when installing nbb, use `npm install nbb --no-optional`.

## Build

Prequisites:

- [babashka](https://babashka.org/) >= 0.4.0
- [Clojure CLI](https://clojure.org/guides/getting_started#_clojure_installer_and_cli_tools) >= 1.10.3.933
- Node.js 16.5.0 (lower version may work, but this is the one I used to build)

To build:

- Clone and cd into this repo
- `bb release`

## License

Copyright © 2021 Michiel Borkent

Distributed under the EPL License. See LICENSE.
