# Notes on the Lively Kernel

Lively Kernel tidbits I've picked up from conversation, code reviews, and raw experimentation. Significant credit and thanks to Robert Krahn (RK) for his contributions to this guide.


## General Questions

### What are Lively's guiding principles?

Courtesy of RK:

1. To have a construction environment that is self-sufficient (e.g., able to"construct itself") including mechanisms to explore and modify the existing system.

2. To make the process of creating as concrete as possible, i.e. to represent abstract things as inspectable and modifiable objects, base the user interface on direct manipulation mechanisms.

3. to make the system easy to work with, i.e. it runs in browser so users don't have to install anything but provides enough abstraction that you don't have
   to understand the entire web stack to use it.

### How are these principles exposed in the UI?

1. PartsBin and object-focused development, as presented in ["The Lively PartsBin"](https://www.computer.org/csdl/proceedings/hicss/2012/4525/00/4525a693.pdf).
The idea behind this is to encapsulate components and entire applications into objects that are shared by means of a repository called PartsBin. Development happens concretely by manipulating objects and scripting them in a tool we called ObjectEditor. One advantage if this approach is that the object you are working on is always "at hand", i.e. you can evaluate code using it. This is different to a class-based workflow where you are at least one step (instantiation of the class) removed from the concrete thing.

2. Being able to "lift" available data sources, services, and non-Lively systems into Lively itself to program with, combine them, represent them as concrete objects. There are several facets to this, besides the integration of "web-stuff" such as JavaScript libraries and HTTP services we also make use to control operating system processes via nodejs and provide a remote messaging mechanism called Lively-2-Lively that automatically connects Lively worlds and optionally other systems that implement the [L2L protocol](http://lively-web.org/users/robertkrahn/2013-08-17_Lively2Lively.html).

3. Finding presentations for control flow and dynamic relationships in programs and their execution. One element of this is certainly [how "debugging" works in Smalltalk](https://www.youtube.com/watch?v=1kuoS796vNw) which leads to a process during which users don't have to play "computer" in their heads and which gives direct access to the runtime state during program execution. In Lively we still haven't satisfying support for that and one of the major goals for us is to finally change this. The existing [debugger work](http://lively-web.org/projects.html#show=debugging), to which Chris who worked with us before contributed, is part of the solution we have in mind.

## The Object Model

### What flavor(s) of object orientation does Lively support?

Lively's core is built around a hierarchy of classes and instances, inspired by the class system of Prototype.js. Individual objects may be augmented with new or modified properties using  `Object.extend` (see below). And of course, it's all Javascript under the hood, so custom object models can be built within Lively worlds.

### Can I still use prototypical inheritance in Lively?

Yes. Objects in Lively can have a class (the constructor attribute of an object points to it) but they can still be changed prototypically. This means that methods/attributes of a specific object can be changed without changing its class.

This general facility to provides a way to override the behavior of an instance of a class.

### What's the difference between `Object.subclass` and `Object.extend`?

Properties (including functions) defined using `Object.subclass` apply to the instances of a class, i.e., they are added to the prototype property of the class. In contrast, properties (including functions) defined using `Object.extend` are added directly to a given object, not its prototype.

### Does Lively support multiple inheritance?

Not currently. That said, you can copy functions and properties from one class to another (thoughbe advised that $super, instanceof may not work as expected).
```javascript
Object.subclass('ClassA', {
			             m1: function() { return 1},	
			             m2: function() { return 2}	,
			    });
			    
 Object.subclass('ClassB', {	
			      		  m2: function() { return 22 },
				          m3: function() { return 3 }
				});

ClassA.subclass('ClassC', ClassB.prototype);

x = new ClassC();
x.m1() // 1
x.m2() // 22
x.m3() // 3
x instanceof ClassA // true
x instanceof ClassB // false
```

### How do Lively modules work, at a high level?

Modules are used to identify Javascript and other source files. Consider an example:

```javascript
module('lively.ide').requires('lively.Tools', 'lively.Ometa').toRun(function() {
// Code depending on lively.Tools and lively.Ometa
}) // end of module
```

This definition does two things:
* Define the module 'lively.ide'. This module can then be required by other parts of the system, e.g.,:

```javascript
require('lively.ide').toRun(function() { new lively.ide.SystemBrowser().open() })
```

require() will check if the 'lively.ide' module is already loaded. If not, the module name will be resolved to a URL, e.g., {Config.codeBase}/lively/ide.js. This URL will then be used to load lively/ide.js asynchronously.
Once the file has been loaded, code inside toRun() will be executed.

* Define the namespace 'lively.ide'. Once the module has been loaded its members are accessible in Javascript (e.g., lively.ide.SystemBrowser).

## Morphic 

### How can I override Morph mouse behavior, say, to capture a drag event?

First, make sure that dragging is enabled: morph.enableDragging().

There are three drag-related handlers:
* onDragStart
* onDrag
* onDragEnd

```javascript
morph.addScript(function onDrag(evt) {
    var globalPosition = this.worldPoint(pt(0,0)),
        nowHandAngle = evt.getPosition().subPt(globalPosition).theta(),
	newRotation = this.startRotation + (nowHandAngle - this.startHandAngle);
	this.setRotation(newRotation);
});

morph.addScript(function onDragStart(evt) {
    var globalPosition = this.worldPoint(pt(0,0));
    this.startRotation = this.getRotation();
    this.startHandAngle = evt.getPosition().subPt(globalPosition).theta();
});
```

### Why doesn't my morph rotate about its center?
Morphs rotate around their origin, which appears as a small red dot in the halo menu, and may not be identical to the morph's centroid. You can move this origin by grabbing the red dot, or by executing `morph.setOrigin(morph.innerBounds().center())`.


## The PartsBin

### How can I open a part programmatically?

```javascript
$world.openPartItem('Triangle', 'PartsBin/Basic');
```


### Is it possible to add parts stored in remote PartsBins to my world?

You can teach your PartsBinBrowser about remote PartsBins dynamically. A few options:
All examples assume you're attempting to to a remote PartsBin at http://my-lk-server:9001/PartsBin/ and that your firewall doesn't block traffic on port 9001.


* Teach your PartsBinBrowser about a new PartsBin:

    ```javascript
    lively.PartsBin.getPartsBinURLs = lively.PartsBin.getPartsBinURLs.wrap(function(proceed) {
        return proceed().concat([new URL("http://my-lk-server:9001/PartsBin/")])
    });
    lively.PartsBin.discoverPartSpaces(function() { /*when done*/});
    ```

* Open a PartsBinBrowser on a remote PartsBin:
    ```javascript
    $world.openPartsBin();
    var pb = $morph("PartsBinBrowser").targetMorph;
    pb.setPartsBinURL(new URL("http://my-lk-server:9001/PartsBin/"));
    pb.reloadEverything();
     ```

* Programmatically load a remote Part, "Triangle", stored in the "Basic" category, from the remote PartsBin:
    ```javascript
    // Load a "PartsSpace" = a category of another PB
    var ps = lively.PartsBin.partsSpaceWithURL(new URL("http://my-lk-server:9001/PartsBin/Basic")).load()
    // list of items:
    ps.getPartItems()
    // load a part
    ps.getPartItemNamed("Triangle").loadPart().part.openInWorld();
    ```

## Connecting to the outside world

### Can I use it to talk to hardware?

Absolutely. You could use a NodeJS subserver to dispatch requests to the OS and return responses. For low-level access to ports on embedded hardware (e.g., a Raspberry Pi), you could also use a library like [PI-GPIO](https://github.com/rakeshpai/pi-gpio).

### How can I embed third-party HTML, possibly with external Javascript libraries?

The HTMLWrapperMorph (available in the default PartsBin) is good starting point. You can set the the HTML contents of this morph directly. If your HTML depends on external libraries (e.g., D3, THREE.js), you can include these by overriding the morph's
default onLoad() method:

```javascript
 // run on part / world load
   function onLoad() {
       this.loadLibs()	
           .then(function() { render(); })
	   .catch(function(err) { $world.inform("load error " + err) });
	}

  // load d3 + verious vega libs
    function loadLibs() {
        var libs =  [
	      "//d3js.org/d3.v3.js",
	      "//vega.github.io/vega/vega.js",
	      "//vega.github.io/vega-lite/vega-lite.js",
	      "//vega.github.io/vega-editor/vendor/vega-embed.js"
	      ];
		 
	      return lively.lang.promise.chain(libs.map((url, i) =>
			 function() { return new Promise((resolve, reject) => JSLoader.loadJs(url, resolve)); }))
					         .then(function() { return show('Loaded!'); })
						 .catch(function(err) { return $world.logError('Load failed: ' + err); });
    }
```

Note that some third party libraries load asynchronously, so you may need to add a timeout function to loadLibs() to ensure all libraries load before passing control to the next function in the promise chain:

```javascript
function loadLibs() {
    var libs =  [
      "//d3js.org/d3.v3.js",
      "//vega.github.io/vega/vega.js",
      "//vega.github.io/vega-lite/vega-lite.js",
      "//vega.github.io/vega-editor/vendor/vega-embed.js"
    ]
    
    var libTests = [
      () => typeof d3 !== "undefined",
      () => typeof vg !== "undefined",
      () => typeof vg !== "undefined",
      () => typeof vg.embed !== "undefined"
    ]
      
    lively.lang.promise.chain(libs.map((url, i) => () => {
      JSLoader.loadJs(url);
      return lively.lang.promise.waitFor(3000/*ms timeout*/, libTests[i])
    }))
    .then(() => show("Loaded!"))
    .catch(err => $world.logError("Load failed: " + err))
  }
```

### What's the preferred way to make an ajax call?
Probably using a WebResource:

```javascript
new URL('http://google.com').asWebResource().beAsync().noProxy().withJSONWhenDone(function(content) {
    console.log(content) });
```

## Workflow

### Where are my `console.log` messages redirected to?

You can keep an eye on them by keeping an instance of the System Console running ($world right click, Tool > System Console). 

### How can I get the user's attention with a dialog box?

Try `$world.inform`. 

```javascript
$world.inform("Unable to retrieve remote resource.")
```

### Can I accept input with a popup?

Try `$world.prompt`.

```javascript
var response = $world.prompt("Enter a city name")
```

### Can you recommend any useful prototyping patterns?

While I make no claim these techniques are the most effective way to prototype tools inside Lively, I have found them helpful:

* If you don't know exactly how your morph should behave, or what protocol it should support, start with a Rectangle morph. Use the Object Edtitor
to attach methods to it. Use the Inspector to send messages to the morph. Use this mechanism to refine method signatures until the morph behaves as you expect.
Save it in its own world so you have a dedicated workspace as you develop it, or publish to the PartsBin with the caveat that it's not feature complete.

* Create `aboutMe()` scripts that take no arguments and return docstrings for parts.

* Place code to restore a morph to its initial state in a method or script called "reset". This will automatically add a "Reset" item to the morph's context menu.

* If you override a morph's `onLoad()` method, you can make a call to `this.reset()` to ensure newly created morphs (say, pulled from the PartsBin) appear in the world 
in their 'default' state.


## Deployment

### Can I use lively in a multitenant environment?

Definitely. This is how lively-web is set up. Common practice is to give each user a directory to keep his worlds and to share parts via the PartsBin.


### Can I change the server configuration (host, port) Lively runs on?

es, this option (along with many others) can be configured by bin/lk-server.js. To make your lively instance accessible to other users on your local network on port 8080, try the following from the LivelyKernel root directory:

```sh
$ node bin/lk-server.js --host 0.0.0.0 --port 8080
```


## TODO

### Can I run a Lively world without persistent storage?

This used to be possible. Can we still do this? If so, what are the constraints?

### What's the difference between `$morph('aMorphName')` and `this.get('aMorphName')`?

### How can I use HTTPS?
