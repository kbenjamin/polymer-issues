# Polymer 1.0 / 1.x Issues, Problems, and Pain Points
##Lessons Learned the Hard Way

_It Just Doesn't Work™_

## Data-driven Programming Model

Polymer works best when you leverage data binding rather than events, giving you greater independence between elements in your application but that scenario comes with some challenges, primarily due to the asynchronous nature of web components and Polymer.

Events are, generally, most useful for UI interactions and, critically as you'll see below, for knowing when the application is ready by listening for `WebComponentsReady` e.g.

```
window.addEventListener('WebComponentsReady', function() {
  //do something
}.bind(this));
```

## Reference Example

__We'll refer to this example throughout so be sure you understand it before continuing:__

```
<div id="wrapper">
 <first-el  id="first"  item="{{myitem}}" state="{{mystate}}"></first-el>
 <second-el id="second" item="{{myitem}}" state="{{mystate}}"></second-el>
</div>


Polymer({
  is: 'first-el',
  properties: {
    item: {
      type: Object,
      notify: true,
      value: function(){ return {}; },
      observer: 'itemChanged'
    },
    state: {
      type: String,
      notify: true,
      value: '',
      observer: 'stateChanged'
    }
  },
  itemChanged: function(newVal, oldVal){
    // do something
  },
  stateChanged: function(newVal, oldVal){
    // do something
  },
});


Polymer({
  is: 'second-el',
  properties: {
    item: {
      type: Object,
      notify: true,
      // Important not to create another Object so don't do this:
      // value: function(){ return {}; },
      observer: 'itemChanged'
    },
    state: {
      type: String,
      notify: true,
      value: '',
      observer: 'stateChanged'
    }
  },
  itemChanged: function(newVal, oldVal){
    // do something
  },
  stateChanged: function(newVal, oldVal){
    // do something
  }
});
 ```
### Binding Values
It is important to understand how data binding works. While the Polymer documentation describes it, what's not obvious, or at least wasn't to me, is how simple it really is. Looking at the example, you will see the following relationship between the various Objects:

`first.item = wrapper.myitem = second.item`

`first.item === wrapper.myitem === second.item // => true`

In other words, it's the exact same object.

#### Default Values
It's critical that you only set the default value of an Object or Array in ___one___ element or you will end up with two different objects being bound and, therefore, not communicating. e.g.

```
Polymer({
  is: 'second-el',

  properties: {
    item: {
      type: Object,
      notify: true,
      // Oops...
      value: function(){ return {}; },
      observer: 'itemChanged'
    }
  },
```
Creates this instead:

`first.item = wrapper.myitem`

`second.item = wrapper.myitem`

Note: `wrapper.myitem` could be either value, depending on the timing of the last setter.

`first.item === wrapper.myitem === second.item // => false`

When we talk about communication via data binding, all we're really talking about is pointer references to the exact same Object. No _magic_ is involved.

## Observer(s)
### Observers fire on transition from `undefined` to an initial value
This is both a feature and a bug, depending on your needs. In any case, it's important to know so you can handle either scenario.

#### Example
Using our example, imagine this situation:

```
itemChanged: function(newVal, oldVal){
  doAmazingThingsWithItem(this.item);
}
```
The problem is that the first time this code runs, `this.item === {}`. It is initialized when `first-el` is being created. For reference, `newVal === {}` and `oldVal === undefined`.

This is not a problem if you know about it and can even be a good way to leverage data-driven programming to start something up once the property is initialized. Here are two examples:

##### Avoiding the initialization transition:

```
itemChanged: function(newVal, oldVal){
  if( oldVal === undefined ){ return; } // Just the initialization. Ignore it.
  doAmazingThingsWithItem(this.item);
}
```

##### Using the initialization transition:

```
itemChanged: function(newVal, oldVal){
  if( oldVal === undefined ){
    initializeSomething();
  } else {
    doAmazingThingsWithItem(this.item);
  }
}
```

I'm finding this pattern useful to initialize key application elements and, in fact, to kick off the app based on a particular element being selected via `iron-selector`:

```
<iron-selector attr-for-selected="active" ...>
  <first-el></first-el>
...

// In first-el:

properties: {
  ...
  active: {
    type: Boolean,
    value: false,
    notify: true,
    observer: 'activeChanged'
  }
},
activeChanged: function(newVal, oldVal){
  if( newVal === true ){
    doStuffWhenActive();
  }
}
...
```

There are some issues, however...

### Co-dependent properties may not all be set before the `observer` executes

Whenever you are using two or more data-bound properties together, you have to make sure both have changed before you can use either of them.

The precise timing of when binding change observers execute is asynchronous. This creates some issues when there are dependencies between bound properties.

```
// first-el
this.set('item', someValue);
this.set('state', 'ready');

// second-el
itemChanged: function(){
  if( this.state=='ready'){
    // You may not get here!
  }
}
```

#### Workarounds
* `observers: [ 'myObserver( propA, propB ) ]`

This requires both properties to be defined.

It's useful in the case of initialization but still does not resolve the problem of co-variant properties.

It's not much help if any of the properties might be allowed to not have a value or that value isn't set yet so we haven't solved all our problems.

A better solution is to rely on `Polymer.prototype.async` when calling any method that depends on more than one data bound property:

```
// first-el
this.set('item', someValue);
this.set('state', 'ready');

// second-el
itemChanged: function(){
  this.async( // wait for microtasks...
    function(){
      if( this.state=='ready'){
        // This works so do something
      }
    },
    1 // delay - optional
  );
}
```

Async does two things:
* Awaits completion of microtasks
* Optionally, delays for a period of time. How beneficial this feature is, I don't know. Timing can vary by platform and program so it's not a reliable technique.

We've solved our timing issues with `async` within our element and in relation to its children.

Sibling elements, however, are another matter...

## Initialization of Elements is Asynchronous
### Timing Problems with Siblings

Interdependent elements may bind and modify property / attribute values prior to a sibling element being ready to listen for those changes.

Where the `async` method shown above is useful within an element and its children, between siblings, it's not reliable because elements load and are created asynchronously. How, then, do we resolve the resulting timing problems?

One partial solution is to not use property bindings before `WebComponentsReady` has fired. How, exactly, to do this I don't know since bindings begin flowing during element creation, which may well occur prior to `WebComponentsReady`.

My solution to these timing problems has been to create an execution queue which dequeues upon the `WebComponentsReady` event. It defers all execution until that time for any element that incorporates and uses the `enqueue`, `setDeferred`, or `notifyPathDeferred` methods:
  * `this.enqueue('someFunction' [arg1, arg2], this);`
  * `this.setDeferred('propertyName', value, this);`
  * `this.notifyPathDeferred('pathName', value, this);`

These methods are useful if you know your element will need to interact with siblings, as in our example. For interactions with children or internally co-dependent properties, you can use `async` instead which will be slightly more performant.

Going back to our example, this may fail:

```
// first-el
ready: function(){
  this.set( 'state', 'doIt');
  this.set( 'item', {key: 'Surprise!'} );
}

// second-el
stateChanged: function(newVal, oldVal){
  if(newVal === undefined){ return; }
  if( this.state === 'doIt'){
    // This might not run as 'state' might have an old value still
    doSomethingAmazingWithItem(this.item); // Or maybe it will but 'item' still has an old value
    this.set('state', 'done');
  }
}
```

## Timing Issues Resolved

To resolve these issues, both internally and externally, i.e. between siblings, I've created this Behavior:

### Execution Queue Code (ReadyQueueBehavior)

```
var MyBehaviors = window.MyBehaviors = window.MyBehaviors || {}; // Global variable

MyBehaviors.ReadyQueueBehavior = {

  properties: {
    appReady: {
      type: Boolean,
      value: false,
      notify: true,
      observer: 'appReadyChanged'
    },
    queue: {
      type: Array,
      value: function(){ return []; },
      notify: false
    },
  },

  created: function(){
    window.addEventListener('WebComponentsReady', function() {
      this.set('appReady', true);
    }.bind(this));
  },

  appReadyChanged: function(){
    if( this.appReady === true ){
      this.dequeue();
    }
  },

  setDeferred: function(prop, val, scope){
    scope = scope || this;
    this.async(function(){
      this.enqueue( 'set', [prop, val], scope );
    });
  },

  notifyPathDeferred: function(prop, val, scope){
    scope = scope || this;
    this.async(function(){
      this.enqueue( 'notifyPath', [prop, val], scope );
    });
  },

  enqueue: function( functionName /*string*/, valuesArray /*array?*/, scope /*object?*/ ){
    scope = scope || this;
    valuesArray = valuesArray || [];
    if( typeof scope[functionName] === 'function' ){
      this.push( 'queue', {fn: functionName, values: valuesArray, scope: scope} );
      this.dequeue(); // attempt to run immediately - only happens if appReady===true
    }
  },

  dequeue: function(){
    var item, scope;
    if( this.appReady === true ){
      while(this.queue.length > 0){
        item = this.shift('queue'); // FIFO
        scope = item.scope;
        scope[item.fn].apply( scope, item.values );
      }
    }
  }

};

```

By using the queue, action is deferred until `WebComponentsReady` fires but all other element creation methods can continue. While it might seem that we can use the `ready` or `attached` lifecycle methods, when components are data-driven, that model doesn't work. The queue does.

I've found this technique valuable for preserving the independence of elements so as not to have to rely on some master controller to be aware of their state.

##Computed Properties
When using computed properties, we can extend what we've learned about co-dependent properties to ensure they work correctly.

Take this example:
```
<template>
  <first-el hidden="{{getComputedState(item)}}"></first-el>
</template>
Polymer({
  is: 'first-el',
  properties: {
    item: {
      type: Object,
      notify: true,
      value: function(){ return {}; }
    },
    state: {
      type: Boolean,
      notify: true,
      value: false
    }
  },
  getComputedState: function(i){
    // Hide first-el under certain conditions...
    if( this.state === true && i.someProperty === 'foo' ){
      return true;
    }
  }
});
```

This looks like it should work but _It Just Doesn't Work™_.

Instead, we get:

`Uncaught ReferenceError: state is not defined`

The problem is that the code fails during the initialization of the element. It's not as simple as detecting an `undefined` state, either. The computed property fires every time _all_ the listed properties change, an only then. Therein is the clue to resolving this problem but first...

### Computed Properties: What Doesn't Work
We can't use our previous tricks so easily here. The problem is that the return value from the function is what is needed in the computed property so using the `enqueue` or `async` methods becomes harder to work with as the return values from these functions aren't, without some effort and refactoring, what we are looking for. Fortunately, there is a very simple solution.

### The Correct Way to Use Co-Dependent Computed Properties
Simply include all the dependent properties in the function call:

```
<template>
<first-el hidden="{{getComputedState(item, state)}}"></first-el>
</template>
Polymer({
  is: 'first-el',
  properties: {
    item: {
      type: Object,
      notify: true,
      value: function(){ return {}; }
      },
    state: {
      type: Boolean,
      notify: true,
      value: false
    }
  },
  getComputedState: function(i, s){
    // Hide first-el under certain conditions...
    if( s === true && i.someProperty === 'foo' ){
      return true;
    }
  }
});
```
Note the inclusion of `state` in the function call.

While I also the changed the function to remove a dependence on `this.state` and instead use the local variable `s`, that is optional. `this.state` will be properly initialized so you can reference it either way since it's just a pointer to the same object, i.e. `s === this.state // ==> true`

This works because, as I said, the computed property fires every time _all_ the listed properties change. While I haven't investigated this, I assume that computed properties rely on the same code that `observers: [ 'doSomething(item, state)' ]` does.

