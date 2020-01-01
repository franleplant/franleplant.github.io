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

This Post is meant to be exhaustive so if you find anything I have missed please
report it and I will make the required changes.

This Post assumes ES6 knowledge.

Let's dive in!

Update August 2016  <br>
We’ve been [translated to Japanese!](http://postd.cc/react-higher-order-components-in-depth). <br>
Thanks to all for your interest!

Update December 2016  <br>
I moved to a my new blog, and with that event came some changes to the post.
Improved the Parent Component section, and moved it out of Appendixes and into a main section.
Added a comparative table.
Fixed some typos and improved some parts of the text.
Added more examples.


# Table of Contents

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

## What are Higher Order Components?

> **A Higher Order Component is just a React Component that wraps another one.**

This pattern is usually implemented as a function, which is basically a class factory (yes, a class factory!),
that has the following signature in pseudo-Typescript (don't worry, it'll be last typescript you will see in this post)


```typescript
type hocFactory: (W: React.Component) => React.Component
```

Where W (`WrappedComponent`) is the `React.Component` being wrapped and then returned Component, E `Enhanced Component`,
is the new, HOC, `React.Component` being returned.

The “wraps” part of the definition is intentionally vague because it can mean one of three things:

1. Props Proxy: The HOC renders the `WrappedComponent` W,
2. Inheritance Inversion: The HOC extends the `WrappedComponent` W.
3. Parent Components: The HOC is a regular parent of the `WrappedComponent` W.

**Note**: Parent Components is not implemented with the type signature previously discussed.
In fact, one could argue they are not HOC at all, but in my experience I've found that there are a lot
of things that you can achieve with Parent Components withouth the need of any fancy class factories, so that's
why we are going to include Parent Components in our HOC study.

We will explore these patterns in more detail.

## What can I do with HOCs?

At a high level HOC enables you to:

- Code reuse, logic and bootstrap abstraction
- Render Highjacking
- State abstraction and manipulation
- Props manipulation

We will see this items in more detail soon but first, we are going to study the ways of implementing HOCs
because the implementation allows and restricts what you can actually do with an HOC.


## HOC implementations

In this section we are going to study the three main ways of implementing HOCs in React:
Props Proxy (PP), Inheritance Inversion (II) and the special case of Parent Components (PC).
All enable different ways of manipulating the `WrappedComponent`.



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

Note that PP is an interface for normal component composition!

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
also adds more props which are functions (via `mapDispatchToProps`), while also
encapsulating the process of hooking into the store and other stuff. 
Read more in the Case Studies sections.

### What can be done with Props Proxy?

- Manipulating `WrappedComponent` 's props
- Accessing the instance via Refs
- Abstracting State
- Wrapping the `WrappedComponent` with other elements
- Abstract life cycle hook and other routinary stuff (i.e. React-Redux's `connect`)

### Manipulating `WrappedComponent` 's props

You can read, add, edit and remove the props that are being passed to the `WrappedComponent`.


Be careful with deleting or editing important props,
you should probably namespace your Higher Order props not to break the `WrappedComponent`.

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


**Note** how we are only changing the props that `WrappedComponent` receives.
This is an important difference with Inheritance Inversion.


Example: Encapsulate data fetching
{% highlight javascript %}
function ppHOC(WrappedComponent) {
  return class PP extends React.Component {
    // initialize state
    state = {
      loading: true,
      data: {},
    };

    async componentDidMount() {
      // Fetch data
      const myData = await API.getMyData();
      // Update state;
      this.setState(state => {
        state.data = myData;
        state.lading = false;
        return state;
      })
    }

    render() {
      return (
      <WrappedComponent
        {...this.props}
        loading={this.state.lading}
        data={this.state.data}
      />
      );
    }
  }
}
{% endhighlight %}

In this last simple, and very rough, example we are encapsulating data fetching and providing inside
a HOC, which means that we can create generic HOCs that data to our dumb[er] components, that don't
need to worry about data fecthing, maintaining state or else.

In a way, when you start using Redux and React-Redux you basically implement this pattern with your `connect`
function beeing the HOC that provides data and useful methods (pieces of your app state and action dispatchers).



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
class Example extends React.Component {
  render() {
    return <input name="name" {...this.props.name}/>
  }
}

const EnhancedExample = ppHOC(Example)
{% endhighlight %}



The input will be a [controlled input](https://facebook.github.io/react/docs/forms.html) automagically.

For a more general two way data bindings HOC see this [link](https://github.com/franleplant/react-hoc-examples/blob/master/pp_state.js)


### Wrapping the `WrappedComponent` with other elements

You can wrap the `WrappedComponent` with other components and elements for styling, layout or other purposes.
This is why we studied that PP is just another interface for normal component composition.

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


### Accessing the instance via Refs

**Important** Most of the time, if you want to access the `WrappedComponent` instance, you should **not**
use this methods as it's inefficient and you should use Inheritance Inversion.
The only case I see for using this pattern is for some interaction with DOM elements, so you could probably create
an HOC that access the native DOM element via refs and calls some native methods such as `focus`.

You can access `this` (the instance of the `WrappedComponent`) with a `ref`,
but you will need a full initial normal render process of the `WrappedComponent` for the `ref` to be calculated,
that means that you need to return the `WrappedComponent` element from the HOC `render` method,
let React do its reconciliation process and just then you will have a `ref` to the `WrappedComponent` instance.

**Note** That in the case your `WrappedComponent` is a DOM element you will get the native dom element
object. Also note that you cannot use refs with Stateless Functional Components.

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

If you need to interact with the Instance I would suggest you use Inheritance Inversion.



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

### Reconciliation process

Before diving in let’s summarize some theory.

React Elements describe what is going to be rendered when React runs it’s [reconciliation](https://facebook.github.io/react/docs/reconciliation.html) process.

React Elements can be of two types: String and Function.
The String Type React Elements (STRE) represent DOM nodes and the
Function Type React Elements (FTRE) represent Components created by extending `React.Component`.
For more about Elements and Components read this [post](https://facebook.github.io/react/blog/2015/12/18/react-components-elements-and-instances.html).

FTRE will be resolved to a full STRE tree in React’s [reconciliation](https://facebook.github.io/react/docs/reconciliation.html)
process (the end result will be always DOM elements).

This is very important and it means that
**Inheritance Inversion High Order Components don’t have a guaranty of having the full children tree resolved.**


> **Inheritance Inversion High Order Components don’t have a guaranty of having the full children tree resolved.**


This is will prove important when studying Render Highjacking.

### What can you do with Inheritance Inversion?

- Render Highjacking
- Manipulating state
- Manipulating the instance via `this`
- Bonus: Class Based Mixin


### Render Highjacking


It is called Render Highjacking because the HOC takes control of the
render output of the `WrappedComponent` and can do all sorts of crazy stuff with it.

In Render Highjacking you can:
- Read, add, edit, remove props in any of the React Elements **outputted** by `render`
- Read, and modify the React Elements tree outputted by render
- Conditionally display the elements tree
- Wrapping the element’s tree for styling purposes (as shown in Props Proxy)

*`render` refers to the `WrappedComponent.render` method

You **cannot** edit or create props of the `WrappedComponent` instance,
because a React Component cannot edit the props it receives,
but you **can** change the props of the elements that are outputted from the **render** method.
If you only want to change the props of elements, you should probably just use Parent Components
and `cloneElements` their children with new props. This is a very powerful technique, see it's power
[here](https://github.com/franleplant/reform/blob/31bb72710ecb83b007750ca432ff3ad3481a89de/src/Reform.js#L52-L114)
(a prototype of a form validation library I've been working)


Just as we studied before, II HOCs don't have a guaranty of having the full
children tree resolved, which implies some limits to the Render Highjacking technique.
The rule of thumb is that with Render Highjacking you will be able to manipulate the element’s tree
that the `WrappedComponent` render method outputs no more no less.
If that element’s tree includes a Function Type React Component then you won't be able to manipulate
that Component’s children. (They are deferred by React’s reconciliation process until it actually renders to the screen.)

Example 1: Conditional rendering.
This HOC will render exactly what the `WrappedComponent` would render unless
`this.props.loggedIn` is not true. (Assuming the HOC will receive the loggedIn prop)


{% highlight javascript %}
{% raw %}
function iiHOC(WrappedComponent) {
  return class Enhancer extends WrappedComponent {
    render() {
      if (this.props.loggedIn) {
        return super.render()
      } else {
        return null
      }
    }
  }
}
{% endraw %}
{% endhighlight %}

Something very similar can be achieved with PP.


Example 2: Modifying the React Elements tree outputted by render.
{% highlight javascript %}
{% raw %}
function iiHOC(WrappedComponent) {
  return class Enhancer extends WrappedComponent {
    render() {
      const elementsTree = super.render()
      let newProps = {};
      if (elementsTree && elementsTree.type === 'input') {
        newProps = {value: 'may the force be with you'}
      }
      const props = Object.assign(
        {},
        elementsTree.props,
        newProps
      )

      const newElementsTree = React.cloneElement(
        elementsTree,
        props,
        elementsTree.props.children
      );
      return newElementsTree
    }
  }
}
{% endraw %}
{% endhighlight %}


**Note** That we are only cloning the top level element of the tree, more sophisticated techniques
include a recursive traversal and cloning of the tree.

The advantage here over using Parent Components (see below in it's own section) is that
you not only have access to the render tree but to instance and it's methods.


In this example, if the rendered output of the `WrappedComponent` has an input
as its top level element then we change the value to “may the force be with you”.

You can do all sorts of stuff in here, you can traverse the entire elements
tree and change props of any element in the tree.
This is exactly how Radium does its business (More on Radium in Case Studies).


> NOTE: You **cannot** Render Highjack with Props Proxy.
While it is possible to access the render method via `WrappedComponent.prototype.render`,
you will need to mock the `WrappedComponent` instance and its props, and potentially handle the component lifecycle yourself,
instead of relying on React doing it.
In my experiments it isn’t worth it and if you want to do Render Highjacking
you should be using Inheritance Inversion instead of Props Proxy.
Remember that React handles component instances internally and your only way of dealing with instances is via this or by refs.
**And also Remember that Instances are different from React Elements.**


### Manipulating state

The HOC can read, edit and delete state of the `WrappedComponent` instance,
and you can also add more state if you need to.
Remember that you are messing with the state of the `WrappedComponent` which can lead to you breaking things.
Mostly the HOC should be limited to read or add state, and the latter should be namespaced not to mess with the `WrappedComponent` ’s state.

Other important thing is that you should **avoid** doing this in a II HOC
{% highlight javascript %}
{% raw %}
// DON'T DO THIS!!!!
export function WRONGIIHOC(WrappedComponent) {
  return class II extends WrappedComponent {
    constructor(props) {
      super(props);

      // This will DELETE all the state that the
      // WrappedComponent might have stored!
      this.state = {  }
    }

    render() {  }
  }
}
{% endraw %}
{% endhighlight %}


Example: Debugging by accessing WrappedComponent’s props and state
{% highlight javascript %}
{% raw %}
export function IIHOCDEBUGGER(WrappedComponent) {
  return class II extends WrappedComponent {
    render() {
      return (
        <div>
          <h2>HOC Debugger Component</h2>

          <p>Props</p>
          <pre>{JSON.stringify(this.props, null, 2)}</pre>

          <p>State</p>
          <pre>{JSON.stringify(this.state, null, 2)}</pre>

          {super.render()}
        </div>
      )
    }
  }
}
{% endraw %}
{% endhighlight %}


This HOC wraps the `WrappedComponent` with other elements, and also displays the `WrappedComponent` ’s instance props and state.
The `JSON.stringify` trick was taught to me by [Ryan Florence](https://twitter.com/ryanflorence) and [Michael Jackson](https://twitter.com/mjackson).
You can see a full working implementation of the debugger [here](https://github.com/franleplant/react-hoc-examples/blob/master/ii_debug.js).

### Bonus: Class Based Mixins

This is not exclusively useful in React land.

You can implement the old Mixin pattern with a Class Based solutions.
The inspiration is drawn from [here](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Classes)


Example: Class Based Mixins

{% highlight javascript %}
{% raw %}
export function myMixin(OtherClass) {
  // anonymous class :)
  return class extends OtherClass {
    newMethod() {}
    otherNewMethod() {}
  }
}
{% endraw %}
{% endhighlight %}

With this technique you can add new methods and new attributes to classes dynamically, exactly as you
will use the old mixins.

Becareful not to overwrite existing methods or attributes in `OtherClass`


## Parent Components

Parent Components are just React Components that have some children.
React has APIs for accessing and manipulating a component’s children.

In a sense, we are cheating when we say that they are a form of HOC but you can achieve a lot of things
that you could achieve with the other HOC implementations.

Example 1: Parent Components accessing its children.

{% highlight javascript %}
{% raw %}
class Parent extends React.Component {
    render() {
      return (
        <div>
          {this.props.children}
        </div>
      )
    }
  }
}

render((
  <Parent>
    {children}
  </Parent>
  ), mountNode)
{% endraw %}
{% endhighlight %}


Example 2: Render Highjacking with Parent Components!
{% highlight javascript %}
{% raw %}
class Parent extends React.Component {
    render() {
      const elementsTree = this.props.children;
      if (elementsTree && elementsTree.type === 'input') {
        newProps = {value: 'may the force be with you'}
      }

      const props = Object.assign(
        {},
        elementsTree.props,
        newProps
      )

      const newElementsTree = React.cloneElement(
        elementsTree,
        props,
        elementsTree.props.children
      )

      return newElementsTree
    }
  }
}
{% endraw %}
{% endhighlight %}

**Note** That we do not have access to `this` of the `WrappedComponent` (which in this case is simply `this.props.children`)
plus we are super assuming that `this.props.children` is simply an input element, otherwise we would need
to do further work, but you get to see the point.


This is a brief list of things you can achieve with Parent Components:

- Render Highjacking (Example 1)
- Manipulate inner props (Example 2)
- Abstract state and send it to the `WrappedComponent` 's children via props.
- Wrap with new React Elements.
- Children manipulation (same as II)
- Parent Components can be used freely in an Elements tree, they are not restricted to once per Component class as HOCs are.

**Note** that when manipulating the Children you potentially will need to wrap your childrens in a single parent root element since it's a restriction in JSX.

Powerful stuff can be made with Parent Component so feel free to experiment.
I've been working on a [Form Validation](https://github.com/franleplant/reform/blob/31bb72710ecb83b007750ca432ff3ad3481a89de/src/Reform.js#L52-L114)
lib for React that exploits Parent Components and
it achieves a lot of a nice stuff. (The lib is in alpha and ongoing a mayor refactor a simplification, so don't use it in production)




## Comparison between techniques


Let's do a quick summary of core features and techniques available for the different implementations
of HOCs. Note that all other techniques not mentioned in this table are different use cases of
one or more techniques in the table.


| HOC                   | Manipulate Props                                    | Render Highjacking | Access the Instance | Regular Component Composition |
| ----------------------|-----------------------------------------------------|--------------------|---------------------|-------------------------------|
| Props Proxy           | Yes. Only of `WrappedComponent`                     | No                 | No*                 | Yes                           |
| Inheritance Inversion | Yes. Only of `WrappedComponent.render()` (children) | Yes                | Yes                 | Yes                           |
| Parent Components     | Yes. Only of `WrappedComponent.render()` (children) | Yes                | No*                 | Yes                           |

*While you can use `ref` to access the instance you should use Inheritance Inversion, which offers full control over it.


## Naming

When wrapping a component with an HOC you lose the original `WrappedComponent` ’s name which might impact you when developing and debugging.

What people usually do is to customize the HOC’s name by taking the `WrappedComponent` ’s name and prepending something.
The following is taken from `React-Redux`.

{% highlight javascript %}
{% raw %}
HOC.displayName = `HOC(${getDisplayName(WrappedComponent)})`
//or
class HOC extends ... {
  static displayName = `HOC(${getDisplayName(WrappedComponent)})`
  ...
}
{% endraw %}
{% endhighlight %}



The `getDisplayName` function is defined as follows:

{% highlight javascript %}
{% raw %}
function getDisplayName(WrappedComponent) {
  return WrappedComponent.displayName || 
         WrappedComponent.name || 
         ‘Component’
}
{% endraw %}
{% endhighlight %}



You actually don't need to rewrite it yourself because [recompose](https://github.com/acdlite/recompose) lib already provides this function.


## Case Studies


### [React-Redux](https://github.com/rackt/react-redux)

React-Redux is the official Redux bindings for [React](http://redux.js.org/).
One of the functions it provides is `connect` which handles all the bootstrap necessary for listening to the store and cleaning up afterwards.
This is achieved by a Props Proxy implementation.

If you have ever worked with pure [Flux](https://facebook.github.io/flux/docs/overview.html)
you know that any React Component that is connected to one or more stores
needs a lot of bootstrapping for adding and removing stores listeners and selecting the parts of the state they need.
So React-Redux implementation is pretty good because it abstract all this bootstrap.
Basically, you don’t need to write it yourself anymore!

So, by using a HOC React-Redux `connect` abstracts:

- listening to the store and re render when it changes.
- cleanup listeners when component is not used anymore.
- getting data from the store (via new props).
- dispatching actions (via new props).


### [Radium](http://stack.formidable.com/radium/)

Radium is a library that enhances the capability of inline styles by enabling
[CSS pseudo selectors](https://developer.mozilla.org/en-US/docs/Web/CSS/Pseudo-classes)
inside inline-styles. Why inline styles are good for you
is subject of another discussion, but a lot of people are starting to do it and
libs like radium really step up the game. If you want to know more about inline
styles start by this presentation by [Vjeux](https://medium.com/u/46fa99d9bca4)

So, how does Radium enables inline CSS pseudo selectors like hover? It implements
an Inheritance Inversion pattern to use Render Highjacking in order to inject proper
event listeners (new props) to simulate CSS pseudo selectors like hover. The event
listeners are injected as handlers for React Element’s props. This requires Radium to
read all the Elements tree outputted by the `WrappedComponent` ’s render method and whenever
it finds an element with a style prop, it adds event listeners props. Simply put, Radium
modifies the props of the Element’s tree (what Radium actually does is a little bit more
complicated but you get the point)

Radium exposes a really simple API. Pretty impressive
considering all the work it performs without the user even noticing.
This gives a glimpse of the power of HOC.

Note that superficially Radium could be implemented as a regular Parent Component,
the limitations or technical decisions behind the use of II are beyond me.



## Appendix A: HOC and parameters

_The following content is optional and you may skip it._


Sometimes is useful to use parameters on your HOCs.
This is implicit in all examples above and should be pretty
natural to intermediate Javascript developers, but just for
the sake of making the post exhaustive let's cover it real quick.

Example: HOC parameters with a trivial Props Proxy. The important thing is the HOCFactoryFactory function.


{% highlight javascript %}
{% raw %}
function HOCFactoryFactory(...params){
  // do something with params
  return function HOCFactory(WrappedComponent) {
    return class HOC extends React.Component {
      render() {
        return <WrappedComponent {...this.props}/>
      }
    }
  }
}
{% endraw %}
{% endhighlight %}


You can use it like this:


{% highlight javascript %}
{% raw %}
HOCFactoryFactory(params)(WrappedComponent)
//or
@HOCFatoryFactory(params)
class WrappedComponent extends React.Component{}
{% endraw %}
{% endhighlight %}


## Closing Words

I hope that after reading this post you know a little more about React HOCs.
They are highly expressive and have proven to be pretty useful in different libraries.

React has brought a lot of innovation and people running projects like Radium,
React-Redux, React-Router, among others, are pretty good proofs of that.

If want to contact me please follow me on twitter @franleplant.
Go to [this repo](https://github.com/franleplant/react-hoc-examples) to play with code I’ve played
with in order to experiment some of the patterns explained in this post.

## Credits
Credits go mostly to React-Redux, Radium, this [gist](https://gist.github.com/sebmarkbage/ef0bf1f338a7182b6775)
by [Sebastian Markbåge](https://twitter.com/sebmarkbage) and my own experimentation.


