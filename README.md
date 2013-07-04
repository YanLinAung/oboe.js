**Oboe.js** takes a fresh appraoch to AJAX for web applications by wrapping a progressive interface
arround https's standard request-response model. It glues a transport that sits 
**somewhere between streaming and downloading** onto to a **JSON parser that sits somewhere between SAX and
DOM**. It is small enough to be a [micro-library](http://microjs.com/#), doesn't have any external dependencies and 
doesn't care which other libraries you need it to speak to.

Oboe makes it really easy to start using json from a response before the ajax request completes; 
or even if it never completes.

# API

Oboe exposes only one globally available object, ```window.oboe```. You can start a new AJAX call and recieve a new Oboe 
instance by calling one of these methods:

```js
   oboe.doGet(    String url, [Function doneCallback(wholeResponse)])
   oboe.doPut(    String url, String body, [Function doneCallback(wholeResponse)])
   oboe.doPost(   String url, String body, [Function doneCallback(wholeResponse)])
   oboe.doDelete( String url, [Function doneCallback(wholeResponse)])
   
// methods also accept an options object:
   oboe.doPost({
      url: String
      body: Object|String
      complete: Function
   })   
```   

```doneCallback``` is passed the entire json when the response is complete.
Usually it is better to read the json one bit at a time than waiting for it to completely download so this parameter 
is optional.

The returned instance exposes four chainable methods:

```js
   .onNode(String pattern, Function callback(thingFound, String[] path, context))
```

```.onNode()``` registers our interest in nodes in the JSON which match the given ```pattern```.
Pattern syntax is for the most part standard [JSONPath](https://code.google.com/p/json-path/). 
When the pattern is matched the callback is given the matching node and a path describing where it was found.
   
```js
   .onPath(String pattern, Function callback(thingFound, String[] path, context))
```

```onPath()``` is the same as ```.onNode()``` except the callback is fired when the *path* matches, not when we have the
thing. For the same pattern this will always fire before ```.onNode()``` and might be used to get things ready for that call.

Alternatively, several paths may be give at once to either ```onPath``` or ```onNode```:

```js
   .onNode({
      pattern : callback,
      pattern : callback,
      pattern : callback
   });
``` 

```js
   .onError(callback(Error e))
```

Supply a callback for when something goes wrong

```js
   .root()
```

At any time, call .root() on the oboe instance to get the json recieved so far. 
If nothing has been recieved yet this will return undefined, otherwise it will give the root Object.
Technically, this could also return an Array but it is unusual for a json file to not have an Object at the
top level.

## Pattern matching

Oboe's pattern matching is a variation on [JSONPath](https://code.google.com/p/json-path/). It supports these clauses:

`!` root object   
`.`  path separator   
`person` an element under the key 'person'  
`{name email}` an element with attributes name and email  
`*`  any element at any name  
`[2]`  the second element (of an array)  
`['foo']`  equivalent to .foo  
`[*]`  equivalent to .*  
`..` any number of intermediate nodes (non-greedy)
`$` explicitly specify an intermediate clause in the jsonpath spec the callback should be applied to

The pattern engine supports 
[CSS-4 style node selection](http://www.w3.org/TR/2011/WD-selectors4-20110929/#subject)
using the dollar (```$```) symbol.

There are also **[some example patterns](#more-example-patterns)** below. 

# Why I made this

Early in 2013 I was working on complementing some Flash financial charts with a more modern 
html5/[d3](http://d3js.org) based web application.
The Flash app started by making http requests for a very large set of initial data. It took
a long time to load but once it was started the client-side model was so completely primed that it wouldn't need to 
request again unless the user scrolled **waaaay** into the past.

People hate waiting on the web so *naturally* I want my html5 app to be **light and nimble** and 
**load in the merest blink of an eye**. 
Instead of starting with one huge request I set about making lots of smaller ones just-in-time
as the user moves throughout the data.
This gave a big improvement in load times but also some new challenges.
 
Firstly, with so many small requests there is an increased http overhead. Worse, not having a model full of data 
early means the user is likely to need more quite soon. Over the mobile internet, *'quite soon'* might mean 
*'when you no longer have a good connection'*.

I made Oboe to break out of this big-small compromise. We requested relatively large data but 
started rendering as soon as the first datum arrived. We have enough for a screenfull when the request is 
about 10% complete. 10% into the download and the app is already fully interactive while the other 90%
steams silently in the background.

Sure, I could have implemented this using some kind of streaming framework ([socket.io](http://socket.io/), perhaps?) 
but then we'd have to rewrite the server-side and the legacy charts would have no idea how to connect to the new server.
It is nice to just have one, simple service for everything.

# Status

Just hitting v1.2, the API has been changing a lot recently but should be settling down now.
Patches and suggestions most welcome.

The project is designed in for easy testability and has [about 140 unit tests](/test/cases), including
a few completely end-to-end tests that rely on a little node server to sends the data in drips.

BSD licenced.
    
# More use cases

Oboe.js isn't specific to my application domain, or even to solving the big-small download compromise. Here are some
more use cases that I can think of: 

**Sarah** is sitting on a train using her mobile phone to check her email. The phone has almost finished downloading her 
inbox when her train goes under a tunnel. Luckily, her webmail developers used **Oboe.js** so instead of the request failing 
she can still read most of her emails. When the connection comes back again later the webapp is smart enough to just 
re-request the part that failed. 

**Arnold** is using a programmable stock screener.
The query is many-dimensional so screening all possible companies sometimes takes a long time. To speed things up, **Oboe.js**,
means each result can be streamed and displayed as soon as it is found. Later, he revisits the same query page. 
Since Oboe isn't true streaming it plays nice with the browser cache so now he see the same results instantly from cache.

**Janet** is working on a single-page modular webapp. When the page changes she wants to ajax in a single, aggregated json 
for all of her modules.
Unfortunately, one of the services being aggregated is slower than the others and under traditional
ajax she is forced to wait for the slowest module to load before she can show any of them. 
**Oboe.js** is better, the fast modules load quickly and the slow modules load later. 
Her users are happy because they can navigate page-to-page more fluidly and not all of them cared about the slow module anyway.

**John** is developing internally on a fast network so he doesn't really care about progressive loading. Oboe.js provides 
a neat way to route different parts of a json response to different parts of his application. One less bit to write.


# Examples

## Using objects from the JSON stream

Say we have a resource called things.json that we need to fetch over AJAX:
``` js
{
   foods: [
      {'name':'aubergine',    'colour':'purple'},
      {'name':'apple',        'colour':'red'},
      {'name':'nuts',         'colour':'brown'}
   ],
   non_foods: [
      {'name':'brick',        'colour':'red'},
      {'name':'poison',       'colour':'pink'},
      {'name':'broken_glass', 'colour':'green'}
   ]
}
```

In our webapp we want to download the foods and show them in a webpage. We aren't showing the non-foods here so
we won't wait for them to be loaded: 

``` js
oboe.doGet('/myapp/things.json')
   .onNode('foods.*', function( foodObject ){
      // this callback will be called everytime a new object is found in the 
      // foods array. We probably should put this in a js model ala MVC but for
      // simplicity let's just use jQuery to show it in the DOM:
      var newFoodElement = $('<div>')
                              .text('it is safe to eat ' + foodObject.name)
                              .css('color', foodObject.colour)
                                    
      $('#foods').append(newFoodElement);
   });
```

## Listening for strings

Want to detect strings instead of objects? Oboe doesn't care about the types in the json so the syntax is just the same:

``` js
   oboeInstance.onNode('name', function( name ){
      jQuery('ul#names').append( $('<li>').text(name) );
   });
```

## Duck typing

Sometimes it is more useful to say *what you are trying to find* than *where you'd like to find it*. In these cases,
[duck typing](http://en.wikipedia.org/wiki/Duck_typing) is more useful than a specifier based on paths.
 

``` js
   oboeInstance.onNode('{name colour}', function( foodObject ) {
   
      // here we'll be given a callback for every object found that has both a name and a colour   
   };
```   


## Reacting before we have the whole object

As well as ```.onNode```, you can use ```.onPath``` to be notified when the path is first found, even though we don't yet 
know what will be found there. We might want to eagerly create elements before we have all the content to get them on the 
page as soon as possible.

``` js
var currentPersonElement;
oboe.doGet('//people.json')
   .onPath({'people.*': function(){
      // we don't have the person's details yet but we know we found someone in the json stream. We can
      // eagerly put their div to the page and then fill it with whatever other data we find:
      currentPersonElement = jQuery('<div class="person">');
      jQuery('#people').append(personDiv);
   })
   .onNode({
      'people.*.name': function( name ){
         // we just found out that person's name, lets add it to their div:
         currentPersonElement.append('<span class="name"> + name + </span>');
      },
      'people.*.email': function( email ){
         // we just found out this person has email, lets add it to their div:
         currentPersonElement.append('<span class="email"> + email + </span>');
      }
   });
```

## Giving some visual feedback as a page is updating

If we're doing progressive rendering to go to a new page in a single-page web app, we probably want to put some kind of
indication on the page as the parts load.

Let's provide some visual feedback that one area of the page is loading and remove it when we have data,
no matter what else we get at the same time 

I'll assume you already implemented a spinner
``` js
MyApp.showSpinner('#foods');

oboe.doGet('/myapp/things.json')
   .onNode({
      '!.foods.*': function( foodThing ){
         jQuery('#foods').append('<div>').text('it is safe to eat ' + foodThing.name);
      },
      '!.foods': function(){
         // Will be called when the whole foods array has loaded. We've already wrote the DOM for each item in this array
         // above so we don't need to use the items anymore, just hide the spinner:
         MyApp.hideSpinner('#foods');
      }
   });
```

## The path passback

The callback is also given the path to the node that it found in the json. It is sometimes preferable to
register a wide-matching pattern and use the path parameter to decide what to do instead of

``` js
// json from the server side. Each top-level object is for a different module on the page.
{  'notifications':{
      'newNotifications': 5,
      'totalNotifications': 4
   },
   'messages': [
      {'from':'Joe', 'subject':'blah blah', 'url':'messages/1'},
      {'from':'Baz', 'subject':'blah blah blah', 'url':'messages/2'}
   ],
   'photos': {
      'new': [
         {'title': 'party', 'url':'/photos/5', 'peopleTagged':['Joe','Baz']}
      ]
   }
   // ... other modules ...
}

oboe.doGet('http://mysocialsite.example.com/homepage.json')
   // note: bang means the root object so this pattern matches any children of the root
   .onNode('!.*', function( moduleJson, path ){
   
      // This callback will be called with every direct child of the root object but not
      // the sub-objects therein. Because we're coming off the root, the path argument
      // is a single-element array with the module name like ['messages'] or ['photos']
      var moduleName = path[0];
      
      My.App.getModuleCalled(moduleName).showNewData(moduleJson);
   });

```

## Css4 style patterns

Sometimes when downloading an array of items it isn't very useful to be given each element individually. 
It is easier to integrate with libraries like [Angular](http://angularjs.org/) if you're given an array 
repeatedly whenever a new element is concatenated onto it.
 
Oboe supports css4-style selectors and gives them much the same meaning as in the 
[proposed css level 4 selector spec](http://www.w3.org/TR/2011/WD-selectors4-20110929/#subject).

If a term is prefixed with a dollor, instead of the element that matched, an element further up the parsed object tree will be
given instead to the callback. 

``` js

// the json from the server side looks like this:
{people: [
   {name:'Baz', age:34, email: 'baz@example.com'}
   {name:'Boz', age:24}
   {name:'Bax', age:98, email: 'bax@example.com'}}
]}

// we are using Angular and have a controller:
function PeopleListCtrl($scope) {

   oboe.doGet('/myapp/things.json')
      .onNode('$people[*]', function( peopleLoadedSoFar ){
         
         // This callback will be called with a 1-length array, a 2-length array, a 3-length array
         // etc until the whole thing is loaded (actually, the same array with extra people objects
         // pushed onto it) You can put this array on the scope object if you're using Angular and it will
         // nicely re-render your list of people.
         
         $scope.people = peopleLoadedSoFar;
      });
}      
```

Like css4 stylesheets, this can also be used to express a 'containing' operator.

``` js
oboe.doGet('/myapp/things.json')
   .onNode('people.$*.email', function( personWithAnEmailAddress ){
      
      // here we'll be called back with baz and bax but not Boz.
      
   });
```  

Like css4 stylesheets, this can also be used to express a 'containing' operator.

``` js
oboe.doGet('/myapp/things.json')
   .onNode('people.$*.email', function( personWithAnEmailAddress ){
      
      // here we'll be called back with baz and bax but not Boz.
      
   });
``` 

## Using Oboe with d3.js

 
``` js

// Oboe works very nicely with d3. http://d3js.org/

// get a (probably empty) d3 selection:
var things = d3.selectAll('rect.thing');

// Start downloading some data.
// Every time we see a new thing in the data stream, use d3 to add an element to our 
// visualisation. This basic pattern should work for most visualistions built in d3.
oboe.doGet('/data/things.json')
   .onNode('$things.*', function( things ){
            
      things.data(things)
         .enter().append('svg:rect')
            .classed('thing', true)
            .attr(x, function(d){ return d.x })
            .attr(y, function(d){ return d.x })
            .attr(width, function(d){ return d.w })
            .attr(height, function(d){ return d.h })
            
      // no need to handle update or exit set here since downloading is purely additive
   });

```

## Error handling

You use the error handler to roll back if there is an error while getting or parsing the json. 
Oboe stops on error so won't give any further callbacks.
 
``` js
var currentPersonElement;
oboe.doGet('people.json')
   .onPath('people.*', function(){
      // we don't have the person's details yet but we know we found someone in the json stream, we can
      // use this to eagerly add them to the page:
      personDiv = jQuery('<div class="person">');
      jQuery('#people').append(personDiv);
   })
   .onNode('people.*.name', function( name ){
      // we just found out that person's name, lets add it to their div:
      currentPersonElement.append('<span class="name"> + name + </span>');
   })
   .onError(function( email ){
      // oops, that didn't go so well. instead of leaving this dude half on the page, remove them altogether
      currentPersonElement.remove();
   })
```
    
## More example patterns:
  
`!.foods.colour` the colours of the foods  
`person.emails[1]` the first element in the email array for each person
`{name email}` any object with a name and an email property, regardless of where it is in the document  
`person.emails[*]` any element in the email array for each person  
`person.$emails[*]` any element in the email array for each person, but the callback will be
   passed the array so far rather than the array elements as they are found.  
`person` all people in the json, nested at any depth  
`person.friends.*.name` detecting friend names in a social network  
`person.friends..{name}` detecting friends with names in a social network  
`person..email` email addresses anywhere as descendent of a person object  
`person..{email}` any object with an email address relating to a person in the stream  
`$person..email` any person in the json stream with an email address  
`*` every object, string, number etc found in the json stream  
`!` the root object (fired when the whole response is available, like JSON.parse())  

## Getting the most from oboe

Asynchronous parsing is better if the data is written out progressively from the server side
(think [node](http://nodejs.org/) or [Netty](http://netty.io/)) because we're sending
and parsing everything at the earliest possible opportunity. If you can, send small bits of the
output asynchronously as soon as it is ready instead of waiting before everything is ready to start sending.

# Browser support

Browsers with Full support are:

* Recent Chrome
* Recent Firefoxes
* Internet Explorer 10
* Recent Safaris

Browsers with partial support:

* Internet explorer 8 and 9
 
Unfortunately, IE before version 10 
[doesn't provide any convenient way to read an http request while it is in progress](http://blogs.msdn.com/b/ieinternals/archive/2010/04/06/comet-streaming-in-internet-explorer-with-xmlhttprequest-and-xdomainrequest.aspx).
While streaming is possible to work into older Internet Explorers, it requires the server-side to write
out script tags which goes against Oboe's ethos of very simple streaming from standard REST services.

The good news is that in older versions of IE Oboe gracefully degrades,
it'll just fall back to waiting for the whole response to return, then fire all the events together.
You don't get streaming but it isn't any worse than if you'd have designed your code to non-streaming AJAX.


## Running the tests

If you want to do hack on Oboe you can build by just running Grunt but sooner or later you'll have to run the tests.

To build and run the tests you'll need:

* The [JsTestDriver](https://code.google.com/p/js-test-driver/) jar installed somewhere, plus a JVM to run it with 
* [Grunt](http://gruntjs.com/) installed globally on your system
* Node
* Some kind of unix-like environment. On Windows, [cygwin](http://www.cygwin.com/) should do.

An [example streaming http server](test/slowserver/tenSlowNumbers.js) to test against can be found in the [test dir](/test). Unfortunately,
JSTestDriver's proxying doesn't support streaming HTTP. To get arround this there is 
[a small proxy](test/slowserver/jstdProxy.js) written in node that sits in front of JSTD.

To start the proxy, streaming server and jstd itself, run:

``` bash
cd test
./jstestdriver-serverstart

```

Now, capture some browsers by visiting [http://localhost:2442](port 2442) and kick off the build like this:

``` bash
cd .. # back to the root dir
./build
```

If all goes well you should see output similar to in the [latest buildInfo](dist/buildInfo).