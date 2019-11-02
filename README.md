# JsAction

JsAction is a tiny event delegation library that allows decoupling the DOM nodes
on which the action occurs from the JavaScript code that handles the action.

The traditional way of adding an event handler is to obtain a reference to the
node and add the event handler to it. JsAction allows us to map between events
and names of handlers for these events via a custom HTML attribute called
`jsaction`.

Separately, JavaScript code registers event handlers with given names which need
not be exposed globally. When an event occurs the name of the action is mapped
to the corresponding handler which is executed.
                                        
Finally, JsAction uncouples event handling from actual implementations. Thus one
may late load the implementations, while the app is always able to respond to
user actions marked up through JsAction. This can help in greatly reducing page
load time, in particular for server side rendered apps.

JsAction是一个很小的事件委托库，它允许在将处理这个动作的JS中进行解耦DOM节点。

添加事件处理程序的传统方法是获取对该节点的引用，然后将事件处理程序添加到该节点。 JsAction允许我们通过称为jsaction的自定义HTML属性在事件和这些事件的处理程序名称之间映射。

另外，JavaScript代码使用不需要全局公开的给定名称注册事件处理程序。 发生事件时，操作的名称将映射到执行的相应处理程序。

最后，JsAction将事件处理与实际实现分离。 因此，人们可能会延迟加载实现，而应用程序始终能够响应通过JsAction标记的用户操作。 这可以帮助大大减少页面加载时间，尤其是对于服务器端渲染的应用程序。
JsAction

## Building

JsAction is built using the [Closure
Compiler](http://github.com/google/closure-compiler). You can obtain a recent
compiler from the site.

JsAction depends on the [Closure
Library](http://github.com/google/closure-library). You can obtain a copy of the
library from the GitHub repository.

The compiler is able to handle dependency ordering automatically with the
`--only_closure_dependencies` flag. It needs to be provided with the sources and
any entry points.

See the files dispatch_auto.js, eventcontract_auto.js, and
eventcontract_example.js for typical entry points.

Here is a typical command line for building JsAction's dispatch_auto.js:

<pre>
find path/to/closure-library path/to/jsaction -name "*.js" |
    xargs java -jar compiler.jar  \
    --output_wrapper="(function(){%output%})();" \
    --only_closure_dependencies \
    --closure_entry_point=jsaction.dispatcherAuto
</pre>

## Using drop-in scripts

If you would like to test out JsAction, you can link precompiled scripts into
your page.

```html

<script src="https://www.gstatic.com/jsaction/contract.js"></script>

...

<script src="https://www.gstatic.com/jsaction/dispatcher.js" async></script>
```

## Usage

You can play around with JsAction already set up with the following directions
at https://jsfiddle.net/q2eacgs7/.

## In the DOM

Actions are indicated with the `jsaction` attribute. They are separated by `;`,
where each one takes the form:

```
[eventType:]<namespace>.<actionName>
```

If an `eventType` is not specified, JsAction will assume `click`.

```html
<div id="container">
  <div id="foo"
       jsaction="leftNav.clickAction;dblclick:leftNav.doubleClickAction">
    some content here
  </div>
</div>
```

## In JavaScript

### Set up

```javascript
const eventContract = new jsaction.EventContract();

// Events will be handled for all elements under this container.
// 所有元素的事件将针在此容器下被处理。
eventContract.addContainer(document.getElementById('container'));

// Register the event types we care about.
// 注册事件类型
eventContract.addEvent('click');
eventContract.addEvent('dblclick');

const dispatcher = new jsaction.Dispatcher();
eventContract.dispatchTo(dispatcher.dispatch.bind(dispatcher));
```

### Register individual handlers

```javascript
/**
 * Do stuff when actions happen.
 * @param {!jsaction.ActionFlow} flow Contains the data related to the action
 *     and more. See actionflow.js.
 */
const doStuff = function(flow) {
  // do stuff
  alert('doStuff called!');
};

dispatcher.registerHandlers(
    'leftNav',                       // the namespace
    null,                            // handler object
    {                                // action map
      'clickAction' : doStuff,
      'doubleClickAction' : doStuff
    });
```

## Late loading the JsAction dispatcher and event handlers

JsAction splits the event contract and dispatcher into two separably loadable
binaries. This allows applications to load the small event contract early on the
page to capture events, and load the dispatcher and event handlers at a later
time. Since captured events are queued until the dispatcher loads, this pattern
can ensure that user events are not lost even if they happen before the primary
event handlers load.

JsAction将事件协议和调度程序分为两个可分别加载的事件二进制文件。 这使应用程序可以在早期加载小事件。
页面以捕获事件，并在以后加载调度程序和事件处理程序时间。 由于捕获的事件将等待直到调度程序加载后才发生捕获，
因此此模式可以确保即使在主事件之前发生主处理程序加载，用户事件也不会丢失。

Visit http://jsfiddle.net/880m0tpd/4/ to try out a working example.

### Load the contract early in the page

Just like in the regular example, in this example the event contract is loaded
very early on the page, ideally in the head of the page.

```html
<script id="contract" src="https://www.gstatic.com/jsaction/contract.js"></script>
<script>
  const eventContract = new jsaction.EventContract();

  // Events will be handled for all elements on the page.
  eventContract.addContainer(window.document.documentElement);

  // Register the event types handled by JsAction.
  eventContract.addEvent('click');
</script>

<button jsaction="button.handleEvent">
  click here to capture events
</button>
```

The event contract is configured to capture events for the entire page. Since
the dispatcher and event handlers are not loaded yet, the event contract will
just queue the events if the user tries to interact with the page. These events
can then be replayed after the dispatcher and event handlers are loaded, which
will be shown in this example next. This will ensure that no user interaction is
lost, even if it happens before the code is loaded.

### Loading the dispatcher and replaying events

At any point later in the page, the dispatcher and event handlers can be loaded
and any queued events can be replayed.

After the dispatcher and event handler code loads, you will configure the
dispatcher just like in the regular example:

```javascript
// This is the actual event handler code.
function handleEvent() {
  alert('event handled!');
}

// Initialize the dispatcher, register the handlers, and then replay the queued events.
const dispatcher = new jsaction.Dispatcher();
eventContract.dispatchTo(dispatcher.dispatch.bind(dispatcher));
dispatcher.registerHandlers(
    'button',
    null,
    { 'handleEvent': handleEvent });
```

There is some new code to replay the queued events:

```javascript
// This code replays the queued events. Applications can define custom replay
// strategies.
function replayEvents(events, jsActionDispatcher) {
  while (events.length) {
    jsActionDispatcher.dispatch(events.shift());
  }
}

// This will automatically trigger the event replayer to run if there are
// queued events.
dispatcher.setEventReplayer(replayEvents);
```

Now any events that happen during page load before the JS has loaded will be
replayed when the primary JS does load, ensuring that user interactions are not
lost.
