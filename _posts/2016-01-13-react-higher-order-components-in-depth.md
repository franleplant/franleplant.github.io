---
layout: post
title:  "React Higher Order Components in depth"
date:   2016-01-13 10:00:09 -0300
tags: [react, javascript, higher order components, HOC]
excerpt: >
  React Higher Order Components, Inheritance Inversion, Render Hijacking, Render Highjacking, Props Proxy, franleplant
---


## Abstract


This post is **targeted to advanced users** that want to exploit the HOC pattern.
If you are new to React you should probably start by reading [React’s Docs](https://facebook.github.io/react/docs/hello-world.html).

Higher Order Components is a great Pattern that has proven to be
very valuable for several React libraries.
In this Post we will review in detail what HOCs are,
what you can do with them and their limitations and, how they are implemented.

In the Appendixes we will review related topics that are not core to HOC study
but I thought we should have covered.

This Post is meant to be exhaustive so if you find anything I have missed please
report it and I will make the required changes.

This Post assumes ES6 knowledge.

Let's dive in!

Update August 2016  <br>
We’ve been translated to Japanese! [http://postd.cc/react-higher-order-components-in-depth](http://postd.cc/react-higher-order-components-in-depth). <br>
Thanks to all for your interest!


## What are Higher Order Components?

> **A Higher Order Component is just a React Component that wraps another one.**

This pattern is usually implemented as a function, which is basically a class factory (yes, a class factory!),
that has the following signature in haskell inspired pseudocode


```haskell
hocFactory:: W: React.Component => E: React.Component
```

Where W (`WrappedComponent`) is the `React.Component` being wrapped and E (`Enhanced Component`) is the new, HOC, `React.Component` being returned.

The “wraps” part of the definition is intentionally vague because it can mean one of two things:
1. Props Proxy: The HOC manipulates the props being passed to the `WrappedComponent` W,
2. Inheritance Inversion: The HOC extends the `WrappedComponent` W.

We will explore this two patterns in more detail.

## What can I do with HOCs?

At a high level HOC enables you to:
- Code reuse, logic and bootstrap abstraction
- Render Highjacking
- State abstraction and manipulation
- Props manipulation

We will see this items in more detail soon but first, we are going to study the ways of implementing HOCs because the implementation allows and restricts what you can actually do with an HOC.


## HOC factory implementations

In this section we are going to study the two main ways of implementing HOCs in React:
Props Proxy (PP) and Inheritance Inversion (II). Both enable different ways of manipulating the `WrappedComponent`.




## Props Proxy

Props Proxy (PP) is implemented trivially in the following way:

{% highlight javascript %}
function ppHOC(WrappedComponent) {
  return class PP extends React.Component {
    render() {
      return <WrappedComponent {...this.props}/>
    }
  }
}
{% endhighlight %}



The important part here is that the render method of the HOC **returns** a React Element
of the type of the `WrappedComponent`.
We also pass through the props that the HOC receives, hence the name **Props Proxy**.

**Note that**
{% highlight javascript %}
<WrappedComponent {...this.props}/>
// is equivalent to
React.createElement(WrappedComponent, this.props, null)
{% endhighlight %}

They both create a React Element that describes what React should render in its reconciliation process,
if you want to know more about React Elements vs Components see this [post by Dan Abramov](https://facebook.github.io/react/blog/2015/12/18/react-components-elements-and-instances.html) and see
the docs to read more about the [reconciliation process](https://facebook.github.io/react/docs/reconciliation.html).

A good example of this pattern is React-Redux `connect` HOC.
Simply put, it proxies the props that the component receives (via `ownProps` parameter)
and also adds more props that come from the state (via `mapStateToProps`) and
also adds more props (which are functions) (via `mapDispatchToProps`), while also
encapsulating hooking into the store and other stuff. Great!

### What can be done with Props Proxy?

- Manipulating props
- Accessing the instance via Refs
- Abstracting State
- Wrapping the `WrappedComponent` with other elements
- Abstract life cycle hook and other routinary stuff (i.e. React-Redux's `connect`)

### Manipulating props

You can read, add, edit and remove the props that are being passed to the `WrappedComponent`.

Be careful with deleting or editing important props, you should probably namespace your Higher Order props not to break the `WrappedComponent`.

Example: Adding new props. The app’s current logged in user will be available in the `WrappedComponent` via `this.props.user`
{% highlight javascript %}
function ppHOC(WrappedComponent) {
  return class PP extends React.Component {
    render() {
      const newProps = {
        user: currentLoggedInUser
      }
      return <WrappedComponent {...this.props} {...newProps}/>
    }
  }
}
{% endhighlight %}

### Accessing the instance via Refs

You can access `this` (the instance of the `WrappedComponent`) with a `ref`,
but you will need a full initial normal render process of the `WrappedComponent` for the `ref` to be calculated,
that means that you need to return the `WrappedComponent` element from the HOC `render` method,
let React do its reconciliation process and just then you will have a `ref` to the `WrappedComponent` instance.

Example: In the following example we explore how to access instance methods and the instance itself of the `WrappedComponent` via refs
{% highlight javascript %}
function refsHOC(WrappedComponent) {
  return class RefsHOC extends React.Component {
    proc(wrappedComponentInstance) {
      wrappedComponentInstance.method()
    }

    render() {
      const props = Object.assign({}, this.props, {ref: this.proc.bind(this)})
      return <WrappedComponent {...props}/>
    }
  }
}
{% endhighlight %}

When the `WrappedComponent` is rendered the `ref` callback will be executed,
and then you will have a reference to the `WrappedComponent` 's instance.
This can be used for reading/adding instance props and to call instance methods.

### State abstraction

You can abstract state by providing props and callbacks to the `WrappedComponent`,
very similar to how smart components will deal with dumb components.
See [dumb and smart components](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0#.o2qmm6j3h) for more information.

Example: In the following State Abstraction example we naively abstract the `value` and `onChange` handler of the name input field.
I say naively because this isn’t very general but you got to see the point.
{% highlight javascript %}
function ppHOC(WrappedComponent) {
  return class PP extends React.Component {
    constructor(props) {
      super(props)
      this.state = {
        name: ''
      }

      this.onNameChange = this.onNameChange.bind(this)
    }
    onNameChange(event) {
      this.setState({
        name: event.target.value
      })
    }
    render() {
      const newProps = {
        name: {
          value: this.state.name,
          onChange: this.onNameChange
        }
      }
      return <WrappedComponent {...this.props} {...newProps}/>
    }
  }
}
{% endhighlight %}




You would use it like this:
{% highlight javascript %}
@ppHOC
class Example extends React.Component {
  render() {
    return <input name="name" {...this.props.name}/>
  }
}
{% endhighlight %}



The input will be a [controlled input](https://facebook.github.io/react/docs/forms.html) automagically.

> Note that the decorator `@ppHOC` is the same as `ppHOC(Example)`

**For a more general two way data bindings HOC see this [link](https://github.com/franleplant/react-hoc-examples/blob/master/pp_state.js)**


### Wrapping the `WrappedComponent` with other elements


You can wrap the `WrappedComponent` with other components and elements for styling, layout or other purposes.
Some basic usages can be accomplished by regular Parent Components (see Appendix B) but you have more flexibility with HOCs as described previously.


Example: Wrapping for styling purposes
{% highlight javascript %}
{% raw %}
function ppHOC(WrappedComponent) {
  return class PP extends React.Component {
    render() {
      return (
        <div style={{display: 'block'}}>
          <WrappedComponent {...this.props}/>
        </div>
      )
    }
  }
}
{% endraw %}
{% endhighlight %}


## Inheritance Inversion

Inheritance Inversion (II) is implemented trivially like this:

{% highlight javascript %}
{% raw %}
function iiHOC(WrappedComponent) {
  return class Enhancer extends WrappedComponent {
    render() {
      return super.render()
    }
  }
}
{% endraw %}
{% endhighlight %}



As you can see, the returned HOC class `Enhancer` **extends** the `WrappedComponent`.
It is called Inheritance Inversion because instead of the `WrappedComponent` extending some `Enhancer` class,
it is passively extended by the `Enhancer`.
In this way the relationship between them seems **inverse**.


Inheritance Inversion allows the HOC to have access to the `WrappedComponent` instance via `this`,
which means it has access to the `state`, `props`, component lifecycle hooks and the `render` method.

I won't go into much detail about what you can do with lifecycle hooks because it’s not HOC specific but React specific.
But note that with II you can create new lifecycle hooks for the `WrappedComponent`.
Remember to always call `super.[lifecycleHook]` so you don't break the `WrappedComponent`.

## Reconciliation process







