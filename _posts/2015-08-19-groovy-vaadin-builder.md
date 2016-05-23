---
layout: post
title:  "A Groovy Builder for Vaadin"
date:   2015-08-19 14:26:32
categories: Groovy Grails Vaadin builder-pattern
---
I’ve been working with Groovy and the [Vaadin][vaadin] UI framework for several years now, but not, until recently, in a Grails 
environment. Vaadin works well in a standard servlet container, but until recently, it wasn’t possible to take 
advantage of all of Grails goodness as well. So I was very excited to see the release of the [Vaadin plugin for Grails][vplugin], 
which allows these two great technologies to work together.

Vaadin is by nature a Java oriented product, but will of course play well with Groovy. It’s a server-side framework, 
and presents an event-driven interface to the application, much like GUI application frameworks such as Swing, Qt, etc. 
It also suffers from one of the drawbacks of this, in that building a complicated UI can lead to some very verbose code, 
as widget definitions are embedded within others, along with parameter settings and other configuration items. UI’s such 
as this are by nature hierarchical, and unfortunately, Java does not allow this hierarchy to be directly visible in the 
code structure. This is very desirable from a maintenance and understanding point of view.

Groovy can come to the rescue here, in the form of the Builder pattern. This pattern is extremely well suited to 
expressing such hierarchical structures, and is deeply embedded in the Groovy ecosystem. It is well-tested and well-used, 
with great support for creating new types of builders. What we need a Groovy builder for Vaadin.

Looking around the net I found a couple of existing implementations, but neither supported Vaadin 7. That, along with a 
desire to ultimately implement as a Grails plugin argued for a new implementation.

The builder allows the definition of a Vaadin UI with a builder-implemented DSL, where the instances of particular UI 
components are defined by the DSL verbs, and the containment hierarchy is defined by child code blocks attached to these verbs. 
The verbs themselves are named directly from the components they create. It allows a definition to change from this:

{% highlight groovy %}
def panel = new Panel();
panel.content = new VerticalLayout();
def label = new Label();
panel.content.addComponent(label);
panel.content.setAlignment(label,Alignment.TOP_LEFT);
{% endhighlight %}

to this:

{% highlight groovy %}
def panel = new VaadinBuilder().build {
    panel() {
        verticalLayout() {
            label('Hi there',alignment: Alignment.TOP_LEFT)
        }
    }
}
{% endhighlight %}

I hope you’ll agree that’s a great improvement to both maintenance and understanding.

There are some other significant improvements that a builder can bring. These include:

- Passing component property values as keyword parameters to the DSL verbs
- Allowing attributes that apply to container children to be placed on the children, and not the parent e.g. ‘expandRatio’, ‘alignment’
- Adding a bind() operator that allows dynamic binding between a model variable and a UI component, including forms

This last point is especially useful. Much of the task of building a Vaadin UI is handling update events from the UI, 
and updating component data sources when the application data changes. With this last operator, a components data source 
can be bound to a property on an Object (or a Map key), and will automatically update when the model value changes. 
The binding can be bi-directional, and is most useful when used with a form component to bind form fields to 
properties on a model object.

You can find the code on GitHub here: [https://github.com/wizardpb/vaadin-builder][repo]. The repo contains a couple of simple 
web-app examples which show the builder usage in both a servlet and Grails application.

[vaadin]: http://vaadin.com
[vplugin]: http://grails.org/plugin/vaadin
[repo]: https://github.com/wizardpb/vaadin-builder
