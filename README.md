# Klang - A Clojurescript logging libary/viewer

This library offers a simple logging interface in clojurescript for
usage in the browser.
Note: You can also use this to send your server logs to a browser window for
easier inspection. So you backend folks might also be interested in this.
It allows for powerful (user defined) log filtering and syntax highlighting for
clojure data structures:

![Example](https://github.com/rauhs/klang/blob/master/docs/img/example.png)


[Demo][].

# Features

* Central hub to push all log messages in your client app
* Define multiple tabs that filter only the messages that you're interested in
  with transducers (allows things like: "wait for event X, then show the next
  10 events")
* **Zero overhead and removal of all log function calls for production by eliding
  all calls with macros. No Klang code goes into your app.**
* Enter a search term to quickly find log messages (Spaces will map to `.*`, so
  similar to Emacs fuzzy search)
* Pausing the UI in case of many logs arriving. This will not discard
  the logs but buffer them.
* **Customize rendering** of any log data (attach render function to any log
  message that emits hiccup)
* Clone existing tabs to quickly apply different search terms
* Logging is asynchronous so you don't have to worry about blocking
* Click on any log message and dump the object to your javascript console (as an
  object). This allows you to inspect the object and even log functions and
  invoke them in your javascript console. See the demo.
* No global state, you *could* create multiple completely independent loggers
  and have mutliple overlays. For instance if somebody wanted to have one
  browser window to display the logs of the server and browser

# Motivation
By now (2015) the javascript and clojurescript community seems to have arrived
at the fact that javascript applications need to be asynchronous throughout.
React's Flux architecture, core.async, re-frame (which uses core.async), RxJS,
CSP etc etc. are all examples of such design decisions.

Asyncronous systems have many advantages but also make reasoning about the
system harder.
One very simple and effective way to cope (or rather: "help") with understanding
behavior is extensive logging from the start of the development.
This projects the concurrent nature of the control flow into a single, easy to
understand and linear trace of events.
The javascript console is not powerful enough, hence this library.

# Clojars

Warning: I've never deployed to clojars.

[![Clojars Project](http://clojars.org/klang/latest-version.svg)](http://clojars.org/klang)

# Usage

The following is the simplest usage:
```clj
(ns your.app.somens
  (:require [klang.core :as k]))

;; We could potentially use multiple independent loggers. Single mode
;; puts everthing in a local *db*
(k/init-single-mode!)
(k/init!)
;; Setups reasonable default colors for :INFO etc
(k/default-config!)

;; Define a logger that always logs as ::INFO
;; Note how we abuse namespaced keywords here to get more detailed logging
;; information than just :INFO
(def lg (k/logger ::INFO))

;; Now call the ::INFO logger with whatever parameters you like
(lg :db-init "Db initiaized" 'another-arbitrary-param)
(lg :validation :ok {:user userid})
;; A timestamp will be automatically added.

;; Or without the indirection of k/logger:
(k/log! ::WARN "User failed to in")

;; Or the low level raw log:
;; Can be useful if we get log message externally from a server.
(k/raw-log! {:time (goog.date.DateTime.)
             :type :INFO
             :ns "whatever.ns.you.like" ;; Doesn't have to exist
             :msg [:one :two "foo"]})

;; Show the logs in an div overlay:
(k/show!)
;; Or just press `l` if you applied (default-config!)

(k/hide!) ;; hide it again.

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;; Note we can also create some tabs which hold only certain events:

;; Setup some new tabs (left menu)
;; Only hold events from one namespace
(k/tab->ns! k/*db* :my-tab-name "my.ns" "some.other.ns")
;; Only hold events from ns and it's children
;; so here: my.ns.* and other.ns.*
(k/tab->ns*! k/*db* :my-ns* "my.ns" "other.ns")
;; Or only hold certain types. Yes same tab name as tab->ns! call:
(k/tab->type! k/*db* :my-tab-name :INFO :WARN)

```

There is nothing special about `::INFO`, you can use any arbitrary keyword.
The `default-config!` sets up some default color rendering and also
registers the keyboard shortcut `l` to view/hide the logs.
See the source code of `default-config!`, it only calls a handful of functions.
If you want to change --for instance-- the key shortcut you can just grab the
code and change it.

There is also many ways you can customize the rendering yourself. See
the section below for more details on this.

Note that if you use this in production facing code then you'll want
to use this library somewhat differently in order to elide any logging
for production (see below).

# Log message layout
Each log message *internally* has the following fields:

* `:time` is a `goog.date.Date` instance. Either user supplied or filled when
  logged
* `:ns` is a `string` holding the namespace where the log came from. User
  supplied or the empty string "".
* `:type` is a `keyword` specifying the "type" of a log message. This can really
  be anything you want. But most people will use `:INFO`, `:ERRO` or `:WARN`
  and the like.
* `:msg` the actual log message. Is always a vector, even if a single item. This
  allows for arbitrary parameters passed to the various logging functions.
  The default renderer always calls `js->clj` on the message
* `:uuid` is a UUID that the log message is identified with.

The only required fields are `:msg` and `:type`, all others can be omitted.
The `:time` defaults to the current time when the message is added.

# Use cases
The library can be used for different use cases which are described in the
following sections:

## Web browser app development
This is likely the most common use case. You're developing a Clojurescript app
for the browser. You want powerful logging. Simply require the library and call
the API functions to log.

### With deployment to clients
In most use cases for web app development you'll want to remove logging data and
the overhead of Klang from your JS code for deployment.
In this case you should use the macros of klang which introduce a level of
indirection that allows you to elide whatever logs you don't want to make it
into function calls.

Personally, I wouldn't even want any Klang code to stay in a production app
since the logging code of Klang isn't too small.
Hence, in production I want to log a subset of messages (such as warn/error/info
but *not* trace/debug) to be pushed into a global core.async channel where I can
send them to my server (or wherever) in case an error occurs.

The `klang.macros` namespace offers a few additional features to the normal
function calls:

* Setup of whitelist and blacklist to elide specific log calls
* Add line number and file name in your cljs file to every log call
* Add local bindings of your log call
* Change the actual log function being called (for instance your own function
  instead for production)

A quick code example:

```clj
(ns your.ns.core
  (:require-macros
   [klang.macros :as macros]))

;; NOTE:
;; Most of the following macro calls emit ZERO code so these are just macros
;; that drive your compilation.

;; By default this is setup:
(macros/logger! 'klang.core/log!)

;; But you could also have your own log function being called:
(macros/logger! 'your.app.logging/send-to-server-on-error!)

;; This adds the filename & line number of every log call:
(macros/add-form-meta! :line :file)

;; This adds the environment (local bindings) to every log call
(macros/add-form-env! true)

;; Sets the default wheater to emit or elide a log call. This is only used when
;; neither the whitelist nor the blacklist matches anything. If both match then
;; the blacklist will win.
(macros/default-emit! true)


;; We then have a blacklist and a whitelist too:
;; This would elide all :*/DEBG messages
(macros/add-blacklist! "*/DEBG")

;; debg! is just like (log! ::DEBG ...)
;; This will not result in any code now:
(macros/debg! :YOU_SHOULD_NOT_SEE_THIS)

;; However non-namespaces keywords don't match the above, so this still calls
;; log:
(macros/log! :DEBG "This will be logged")

;; This can be achieved by the following:
(macros/add-blacklist! "*DEBG")
;; (not the above will be made into a regex (.*)DEBG$


;; We can also add namespaces to the blacklist:
(macros/add-blacklist! "one.bad.ns*")
(macros/log! :one.bad.ns/INFO :YOU_SHOULD_NOT_SEE_THIS)

;; The following log functions exist:
(macros/log! ::INFO :test-this)
(macros/log! ::WHATEVER "you don't have to use :INFO etc...")
(macros/trac! :some "log message")
(macros/debg! :some "log message")
(macros/info! :some "log message")
(macros/warn! :some "log message")
(macros/erro! :some "log message")
(macros/crit! :some "log message")
(macros/fata! :some "log message")

;; A special one is the following:
(macros/env!)
;; It will "catch" the local binding (via &env in defmacro) and 
;; log it.

;; To see the effect you'd have to use it like this:
(let [x :foo
      this-is [:lots :of 'fun]]
  (macros/env! :I-can-dump-local))

```

**More about env**:

Note that if you called `add-form-env!` then you'll get the environment with
every log call. It is not added to the message itself but instead is added as
meta data to the log call. This meta data is invisible to the GUI. You can
see it by clicking on a log message and checking your browser console.

If this approch of Klang's macros is too inflexible to you then you can
[Wiki Macros](head over to the wiki) where an approach with transducers is
shown.

**Note**: The macros are completely optional and the library itself does not use
  any of them. So you can always just use the normal function call.

## Server mode
In this use case you're only interested in viewing logs in a browser window but
the browser window itself does not host a client app that is interested in
logging.

For instance, you're forwarding all your logs from your clojure
backend app to Klang. You may also have multiple backends or even your
browser app to also forward the logging to one klang instance.

This mean that you'll have a central location (the browser app running
klang) where you display your client and server-side logging data in
realtime.

* TODO: Add recipe to receive logs over websocket and push them in with `raw-log!`

### Timbre
For instance writing a custom appender in timbre (TODO) could push the log
messages to a browser window and then displaying them with Klang. The times of
your log message wouldn't be touched since you can supply a date/time with
`raw-log!`.

* TODO: Code a timbre appender that also sends the log messages to a websocket
  for displaying with Klang.

# Customizing
In general Klang allows for a lot of customization. In fact, most of fanzy
coloring you see in the above screenshot is added by the standard API. It's not
baked in. The default renderes wouldn't even render the DateTime of a log
message in a decent format. 

I encourage people to check out the source of `klang/core.cljs`, right at the
end of the file you'll find a comment `functionality through the standard API`.

## Renderers for logs
In the above section I talked about the log message layout (`:ns, :time, :type,
:msg`) but that was only half the truth.
The renderer (reagent) also looks for `[:render :type], [:render :ns] ...`.
Check out the function `render-msg`.

These renderers will be `comp` and called. You may choose to supply your own
render instead of the syntax highlighting one.
You can also change the DateTime layout and coloring.
Some convenience functions are supplied to make it easy to color types,
namespaces etc.

## Tabs
The example code above already defined some code that showed off defining custom
tabs.
You're not limited to filter by `:type` and `:ns` however, you can register
arbitrary transducers for a tab.
This allows you to filter and even modify the log messages.
See the function `tab->transducer!`.
You can find an example of how to use that function in the source code of this
library.
In fact, `tab->type!` and `tab->ns!` are implemented using `tab->transducer!`.

## Message rendering
By default the messages get transformed with `js->clj` function and then get
syntax highlighted by highlight-js.
You can change this by not applying the default-config 

You can find an example of how to use that function in the source code of this
library.

You can change the types colore like so:

```clj
(type->color! db :TRAC "lightblue")
```

You can change the color of namespaces with the `ns->` functions:
```clj
(ns*->color! db "my.ns" "red") ;; my.ns.*
(ns->color! db "my.other" "blue") ;; my.other only, no child NS
```

In general you can map a predicate to a color, in fact the above function are
implemented with this function:
```clj
(defn type->color!
  "Given a type keyword (like :INFO), render the type in color."
  [db type color]
  (pred->color! db (partial = type) :type color))
```

## Pit falls
Since HTML is XML is nested. You have to watch out when you attach multiple
renderer. The second render will receive the previous render result and you'll
have to deal with hiccup data. I'm not sure if there is a way to elegantly
solve this.
Watch out if you get an error saying something about some hiccup data like
`[:span ...]`.

A solution is to keep the maximum number of renderer that act on a field of the
log message to one.

This can be achieved by de-normalizing the renderers and attach fewer but more
powerful renderers to the log messages.
You can always just attach exactly once rather complicated renderer instead of
multiple small ones.

Another strategy is to apply your custom fancy renderer to only a tab (as a
`map` transducer) and leave the *normal* tabs render it normally.

## Listening for logs
Every log is pushed on a `mult` channel (see core.async docs) which you can tap
into. The channel is in `:log-pub-ch` of the `db` atom.
This allows you to also forward the logs to other places (localStorage or a
server).

## Reacting to klang actions
Most actions are pushed on an the action channel `actions-pub-ch` in the `db`
atom. Which again, is a `mult`.
You can `tap` into it and react however you like.
Events include showing/hiding the log overlay, new log, freezing the channel etc
etc.

## Increasing freeze buffer
If you freeze the UI (aka "pause") you'll fill up a `core.async` sliding
buffer. This means you will lose log messages eventually unless you thaw the
UI.
The current log buffer is 1000 messages but you may increase it by manually
`swap!`ing the db field `:freeze-buffer`

# Suggested log types
I'll suggest those log types in order to have same string lengths (similar to
supervisord):

* `:TRAC` -- trace, use it if you dump vars
* `:DEBG` -- debug, very fine grained. Low level
* `:INFO` -- Default logging for all other
* `:WARN` -- Something isn't pretty but the app is ok
* `:ERRO` -- We have an error but we can continue with program execution.
* `:CRIT` -- critical, cannot continue normal program execution but the code
             itself is fine.
* `:FATA` -- fatal, We cannot continue with execution. Code needs fixing.

# TODO

* Improve performance by only rendering the subset of log messages that are seen
  for the current scrolling position.
* Run the tab transducers as xform argument in core.async channel. This would
  allow an easy profiling transducer. The `single-transduce` silliness prevents
  this from working.
* Go through the `TODO:` items in `klang/core.cljs`
* We should probably cut off ridicioulsy long messages and just allow the user
  to click on it to dump the object to the console.
* We should dump all meta information of the log message when user clicks on it.
  But not display it in the UI. This would allow for compact yet rich messages.
  Macros should attach line number and filename to the meta ifno.
* Clicking on the time of a log message should go to the `:all` tab and show
  the message in the context of all log messages. Prereq: Performance with
  partial rendering.

## Ideas

We can render some common data structures in an arbitrary way. If you have an
idea, feel free to submit a pull request. We can always just leave it in the
source for others to use (but not used by the default renderer).
For instance: How to render large data structures? Popup? How to render
structs?

Offer a standalone HTML file that includes all the code, css etc of klang and
that can be started with something simple like pythons SimpleHTTPServer.
The have that listen on a few websocket connections where clients (another
browser window or server) can connect to and push logs.

# Why "Klang"?
Timbre, the de-facto clojure logging library has do with sound.
*Klang* is the German word for *tone/sound*.

## License

Copyright &copy; 2015 Andre Rauh. Distributed under the
[Eclipse Public License][], the same as Clojure.


[Eclipse Public License]: <https://raw2.github.com/rauhs/klang/master/LICENSE>
[Wiki Macros]: <https://github.com/rauhs/klang/wiki/Flexible-macros-recipe>
[Demo]: <http://rauhs.github.io/klang/>

