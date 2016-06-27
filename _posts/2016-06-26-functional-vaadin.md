---
layout: post
title:  "Functional Vaadin - a better way of using Vaadin from Clojure"
date:   2016-06-26 16:39:00
categories: Clojure Vaadin builder-pattern
---
I'd like to announce functional-vaadin, a library for building Clojure webapps
using the Vaadin UI framework. Since Vaadin is Java-based, this can already be
done, but is clunky: lots of doto and setter calls, temporaries to hold parts
of the UI being constructed,etc. etc. This library is aimed at making that a
thing of the past. 

The code is open-source, and is here:
<https://github.com/wizardpb/functional-vaadin>, Jars are deployed to Clojars:
<https://clojars.org/com.prajnainc/functional-vaadin>. There is still work to be
done, mostly on the documentation, as well as some final components and event
handling

The primary goal is to use Clojure homoiconic nature to make the code that
builds a UI as structurally similar as possible to the UI itself. The library
provides builder functions for each component, these take constructor arguments
and/or a configuration Map, as well as any children components. This
effectively eliminates the clutter.

The other main feature is an integration with RxClojure, which makes explicit
event handling a thing of the past. Instead, sequences of Observers can be
created, allowing arbitrary processing of events from the UI. It is quite
possible have these chains receive an event, process it with arbitrary code,
and return new data to other parts of the UI.

Other features include a component naming scheme that allows access to
components via their ID (eliminates temporaries), conversion functions that
interface Clojure immutable data structures to Vaadin data binding objects,
and a better Form mechanism, integrating FieldGroups, layouts and
function-based validation and conversion.

I hope you find this useful! Please feel free to contact me with any further questions.