Axis Programming Language
==========================

Axis is a simple and intuitive language for describing distributed applications. Axis follows the [actor model](https://en.wikipedia.org/wiki/Actor_model), meaning that every object is independent, like cells in a body or servers in a network. This allows for programs to be fast, flexible, and scalable. However, while most actor-model languages revolve around sending messages or instructions, Axis is designed around [reactive programming](https://en.wikipedia.org/wiki/Reactive_programming): the idea that all actors and values are bound to each other, so that a change in one will automatically update the rest. This allows for the programmer to focus on defining persistent data relationships instead of instantaneous instructions, keeping the language high-level and easy to use while maintaining highly concurrent execution.

Before delving into the mechanics, here's a brief taste of what's possible. The following shows how to return the [height of a binary tree](https://stackoverflow.com/a/2603707/1852456). The syntax should be relatively understandable if you know python or javascript, but don't worry about understanding it fully as it is explained more in depth in later sections.

	binaryTreeHeight: tree >>
		tag #height.           // declare a tag, which can be used to attach attributes to objects

		// calculate height of all nodes
		for node in tree.nodes:
			node.#height: Math.max(node.left.#height | 0, node.right.#height | 0) + 1    // define our height as the height of our left or right subtree + 1

		=> tree.#height   // return height of root node

Notice how we use a simple `for`-loop to define the height of each node based on the heights of its left and right child nodes. This would not work in most languages, because if the for-loop traverses the nodes in the wrong order, we would get the wrong answer. In those languages, we would have to use recursion to specify the exact traversal order, eg "first find the height of the left and right nodes, before finding the height of the parent". However, Axis doesn't need to do this. We don't specify order. Instead, we simply define the relationships between data, and the answer works itself out. One way to think about it is, instead of defining the height of each node one by one, _Axis defines the heights of all nodes at the same time_. This is explored more deeply in the "Feedback" section, where we give more examples showing how algorithms in Axis can be cleaner and simpler than their functional or imperative counterparts.

In addition, because everything is unordered, everything can execute independently and concurrently. This makes Axis perfect for defining web services and applications. A lot of the complications attributed to web programming, like routing, HTTP requests, and asynchronous callbacks, completely disappear when using Axis.

For example, we can host a chat server using the following (also explained more depth in later sections)

	messages: collector(file: 'messages.axis')   // all messages, saved to permanent storage
	activeUsers: collector

	ChatServer: Web.Server
		index: Web.Client template    // main page, a template for the client browser to instantiate

			layout: JSX(file: 'layout.jsx')    // page layout, imported from a JSX file

			// get user inputs from the page
			username: layout.find('input.username').value
			draft: layout.find('text-area.message-draft').value
			
			activeUsers <: username    // insert user into list of active users

			// inserts message into the list of messages
			send: => @timestamp
				messages <: (time: timestamp, user: username, text: draft)

This is all we need for Axis to spin up a database, run the server, and generate the client-side javascript to make it all work. In the later section "Web Applications", we explain in depth about how Axis streamlines web development, and starts to blur the lines between dabatases, servers, and clients.

Mechanics
-----------------

### Basic Syntax

The basic syntax is rather intuitive, and looks similar to javascript objects definitions.

	x: 10     // define variables using ":"
	y: x + 3  // math expressions are supported

	someString: "hello world"
	someBoolean: true

Everything in Axis is an "object", which is just a collection of properties. The language is dynamically-typed, which makes it easy to define and use objects.

	someObject: (name: "Joe", age: 20)      // define an object using parenthesis
	someList: (10, 20, 30)                  // lists are ordered collections of values, but they are also just objects. This list is equivalent to (0: 10, 1: 20, 2: 30)
	someList2: (10 20 30)                   // for lists, commas are optional

	someObject.name                      // use "." to access properties, so this will return "Joe"
	someObject.age = 20                  // "=" is the equality operator, so this will return true

	console.log("hello world")         // this prints "hello world" to the console. This syntax actually uses both "cloning" and "function calling", explained in later sections


	if (someObject.age > 18) "adult" else "child"     // conditionals use if...else, so this will return "adult"

	if (someObject) "someObject is defined"           // if you use an object as a condition, it is considered "truthy" unless it is undefined

	otherObject | "otherObject is not defined"        // short-circuit evaluation, so since otherObject is undefined, this returns the string "otherObject is not defined"

We can use indented blocks to define objects and lists as well

	Bob:
		_age: 25               // define private properties using the `_` prefix
		isAdult: _age > 25     // we are in same scope as _age, so we can reference it
		
		// nested object
		name:
			initials: first[0] + "." +last[0] + "."    // notice that order doesn't matter, we can reference `first` and `last` even though they are defined after
			first: "Bob"
			last: "Smith"

	console.log( Bob )          // Bob is in scope, so this will print out the object Bob
	console.log( isAdult )      // isAdult is not in scope, so this will prints "undefined"
	console.log( Bob._age )     // _age is private, so this won't work, prints "undefined"

We can use loops to define repeated behaviors

	joe: (name: "Joe", age: 20) 

	for key, val in joe:
		console.log(key + ": " + val)    // will print "name: Joe" and "age: 20"

	for key in joe.keys:
		console.log(key)                 // will print "name" and "age"

Lastly, the keyword `undefined` basically means that we are trying to access a value that hasn't been defined. For example, if we try to access `someObject.height`, it will return `undefined`.

Because everything in Axis is data, there are no "errors" like traditional imperative languages. Bad code will never result in a program crash or a runtime error. Instead, a value will simply evaluate to `undefined`. For example, even if we tried to use property access on `undefined`, eg `undefined.height`, it would simply return `undefined`.

### Cloning and Implicit Parameters

Axis is a prototypal language. So any object can be "cloned" to create a new object, where we can specify new properties and overwrite old ones, while inheriting the rest. This is extremely similar to the concept of "extending classes" in object-oriented languages. For example:

	division: p, q >>      // declare two "parameters", p and q
		quotient: p / q
		remainder: p % q

`division` is an object, so we can access its properties. However, since `p` and `q` are undefined, it's pretty useless.

	division.p             // returns undefined
	division.remainder     // returns undefined

However, we can clone `division` and provide values for `p` and `q`.

	result: division(p: 16, q: 7)   // clone the object, overriding p and q
	result.remainder                // returns 16 modulo 7, aka 2

Unnamed arguments are mapped to the parameters in the order they were declared.
	
	result2: division(10, 2)    // 10 gets mapped to p, 3 gets mapped to q
	result2.quotient            // returns 5

Declaring parameters is actually optional. If we referenced variables that are not defined in the scope, we are implicitly declaring parameters. The order in which these variables are referenced, determines the parameter order (when mapping unnamed arguments). For example, we could have also defined `division` like this:

	division:
		quotient: p / q        // since `p` and `q` aren't in scope, they are implicit parameters
		remainder: p % q

	result: division(10, 2)    // 10 gets mapped to p, 3 gets mapped to q
	result.quotient            // returns 5

Cloning can do much more than just provide values for parameters. We can also use cloning to overwrite existing properties, and define new properties and behavior. All of this is done within the arguments of the clone operation.

	Person:
		name >>
		greeting: "hello my name is " + name

	Student: Person
		name, school >>
		greeting: Person.greeting(name) + ". I go to " + school

	alice: Student("Alice", "Harvard", console.log(this.greeting))    // create a new student, and automatically print her greeting to console
	bob: joe(name: "Bob")                                               // creates a student with the same behavior as alice, so he also logs his own greeting

Notice that when we clone `alice` to create `bob`, the function `console.log(this.greeting)` will be called again. This might seem counter-intuitive, because in imperative languages, function calls are executed before they are passed as arguments. However, remember that this is not a function call, we are extending an object, and defining new behavior.

### Insertion and Collectors

We can declare a `collector`, which allows us to insert values to it from anywhere using the `<:` operator. For example:

	foo: collector

	foo <: 1
	foo <: 2

	console.log(foo)       // insertions are unordered, so this could print "(1 2)" or "(2 1)"

Collectors make it easy to "construct" objects like we would in imperative languages. It's also important to remember that insertions are unordered, just like object properties and pretty much everything else in Axis.

We can insert to any object, no restrictions. However, by default, objects ignore insertions. In order for insertions to have an effect, the object has to be a `collector` or any object that extends a `collector`. There are many useful ways we can declare a collector. For example:

	sum: collector(+)     // collector(<some function>) will apply the function to all insertions

	for num in (1 2 3)
		foo <: num * num

	console.log(sum)       // will print 1+4+9, aka "14"

By leveraging insertion, we can also define object methods (like class methods in Python/Java)

	Library:
		songs: hashset     // hashsets are simply collectors that filter out duplicates
		artists: hashset

		song: name, artist >>
			songs <: (name, artist)
			artists <: artist
			console.log("song added")

	song1: Library.song('Never Gonna Give You Up', 'Rick Astley')

There is a small issue with this code. When we clone `song` to create new songs, it automatically inserts the song to our library and prints "song added". But since Axis is a prototypal language, that means the `song` object itself will also insert a song and print "song added". Because `artist` and `name` are undefined for this prototype, this initial `song` object will insert these `undefined` values. Definitely not something we want.

Instead, we want to somehow define a module without actually "executing" it...

### Templates and Functions

Templates are simply a way of defining modules without "executing" them. More specifically, a template will not perform any clones or insertions. In addition, accessing any property of a template will return `undefined`. However, while a template is completely inert, any clones of it will be "active".

So to tweak the example from before, we simply change `song` to a template:

		song: template
			name, artist >>
			songs <: (name, artist)
			artists <: artist
			console.log("song added")

	song1: Library.song('Never Gonna Give You Up', 'Rick Astley')

This way, the initial `song` object won't insert anything, and won't print "song added".

There is another special kind of object called a "function". The purpose of a function is to represent an action. We define the output of the action (called the "return value") using `=>`, and then call the function using `->` to get the output. For example:

	add: a b >>
		console.log("adding numbers")
		=> a+b

	add(10, 7)->     // returns 17

Functions are also templates. One way to think about it is, functions are just templates with a special property defined via `=>`, and then accessed via `->`. This allows for the familiar arrow syntax we have in Java or Javascript

	someList.forEach(elem => console.log(elem))

Some additional notes about functions:

* like any other template, we only need to clone a function to execute it. We don't necessarily need to access the return value.
* if you want to call a function without specifying arguments, instead of `fn()->` you can just do `fn->`
* declaring an object as a `template` is the same as declaring a function with the return value `=> this`

Notice that in many ways, Axis functions work pretty much the same as the functions we are used to in functional and imperative. The only difference is that Axis functions allow us to modify the internal behavior of a function via cloning.

### Tags

Tags are actually just a syntax shorthand for defining and using hashmaps. To illustrate that, take this example:

	isEcoFriendly: Hashmap()

	for car in cars
		isEcoFriendly.add(car, car.fuelEconomy > 30)     // cars over 30 miles/gallon are eco friendly

	console.log(isEcoFriendly[prius])         // will print "true". Assume "prius" is in the set "cars"

This is what it looks like using tags

	tag #isEcoFriendly.

	for car in cars
		car.#isEcoFriendly: car.fuelEconomy > 30

	console.log(prius.#isEcoFriendly)         // will print "true" (assume "prius" is somewhere in the set "cars")

as you can see, often times hashmaps are used to define additional attributes and properties for objects, without modifying the original objects. They are extremely common for data processing. Tag syntax makes it actually look and feel like you are working with properties, even though you aren't actually modifying/retrieving properties from the object. This is why tags can also be thought of as "virtual properties".

### State Variables

State variables are also just syntactic shorthand, useful in event handlers. State variables are just objects that represent an ordered list of states. Because most things in Axis are unordered, we have to explicitly declare order when we want it, and state variables are great for doing so. For example:

	numClicks: var 0      // state variable starting at "0"

	onClick: @timestamp
		numClicks := numClicks + 1

is just shorthand for

	numClicks: hashmap(0: 0)      // use a hashmap to store the value of numClicks at each timestamp. Initialized with value 0 at timestamp 0

	onClick: timestamp >>
		numClicks.add(timestamp, numClicks.before(timestamp) + 1)     // take the previous value of numClicks, and add 1, and store that as the current value of numClicks

### Extra Syntax

I recommend skipping this section and coming back to it, these are just syntax shorthands. The next sections on examples and use cases are much more interesting.

* object deconstruction: `(a, b): someObj` = `a: someObj.a, b: someObj.b`
* statements: `someProp.` = `someProp: true` = `someProp: ()`
* spread operator: `...object`
	
		foo: (10, 20, a: "hello")
		bar: (30, b: "world")

		zed(...foo ...bar) = zed( 10, 20, 30, a: "hello", b: "world" )     // spread operator will combine properties, and append list items

* capture blocks: `...`
	allows us to define functions using indented blocks
* array map access: `list[[prop]]`

		// mapped property access using [[ ]]
		someList: ( (a: 10), (a: 20), (a: 30) )
		extractedValues: someList[["a"]]          // extract the value of property "a" from each list item. So list[[prop]] is equivalent to list.map(item => item[prop])

		// extractedValues = (10, 20, 30)

* dynamic keys: `[key]: val`

		[key]: key*key

* matchers: `[matcher]: val`

Examples and Concepts
------------------------

### Timeless

One of the biggest things to understand is that since everything in Axis is unordered and asynchronous, there is no concept of ordered execution, like in imperative languages. Instead, we think of data as persistent and "timeless", and assume that every variable is constantly re-evaluated to keep the value up to date. To drive this point home, let's look at an example.

Let's say given a set of student test scores, we want to find every test score that is above average. In imperative, this would take two steps, one for-loop to compute the average, and then another for-loop to find the test scores that are above average. In Axis, it would look like this:

	tag #aboveAverage.
	
	sum: collector(+)                          // sums up all insertions
	average: sum / testScores.length

	for score in testScores:
		sum <: score                           // insert score into sum collector
		score.#aboveAverage: score > average   // tag score with "true" if the score is above average

This might seem a little weird. Remember that in imperative code, we would need two loops, one to compute the average, and one to see if each test score is above average. So how are we doing this in one loop? In the first "iteration" of the `for` loop, how can we set `score.#aboveAverage` if we haven't finished computing `average` yet?

This is why we have to think in terms of data relationships, not execution. All we are telling the interpeter, is that `sum` is the sum of all scores, `average` is the sum divided by the number of scores, and a score is `#aboveAverage` if it is bigger than `average`. That is all we need to specify, and the interpreter figures out the rest.

To get a better idea, let's explore how an interpreter _might_ go about computing this. When it reaches the `for` loop, it takes the first `score`, and inserts it into `sum`. `sum` notices the change, and updates accordingly. `average` notices the change in `sum`, and updates as well. Lastly, `score.#aboveAverage` notices the change in `average`, and updates accordingly. Now this is just for the first score. Then the interpreter moves onto the next score, and inserts it into `sum`. Again, `sum` gets updated, and then `average`, and then this updates `#aboveAverage` for **both** the first score and the second score. Remember that bindings are persistent, so the first score is still listening for updates to `average`, even if the interpreter has moved onto the second score. This process will repeat until the interpreter has processed all scores.

This might seem extremely inefficient, with these feedback loops going back and updating previous variables. However, note that the dependencies in this example actually form a DAG (directed acyclic graph). In other words, there is no feedback. The order we should compute the variables should be `all scores => sum => average => all #aboveAverage tags`. This way, nothing needs to be computed twice. And this optimization is exactly what the interpreter would leverage to achieve the same level of efficiency as imperative code, without sacrificing the elegance of Axis syntax. As mentioned in the philosophy section, Axis is about focusing on data relationships, and leaves all execution optimizations to the interpreter.

However, there are times where the code _can_ introduce feedback, covered in the next section.

(Note that to compute the total sum we could have just used `sum: Math.sum(...testScores)`, which is perhaps simpler, but the point of the example is to show the "timeless" nature of Axis)

### Feedback

Feedback is an extremely powerful tool that is only possible using Actor-model languages like Axis. Although simpler forms are relatively common in imperative languages. For example, a cyclic data structure:

	Bob:
		child: Alice
	Alice:
		parent: Bob

	console.log( Bob.child.parent.child.parent )   // we can do this because of feedback

Here, we have two objects referencing eachother. However, this is a pretty boring example of feedback, it's just a static structure.

The real power of feedback becomes apparent in more complicated cases. One example is actually the `binaryTreeHeight` example given in the introduction section. But let's go over a similar but slightly more complex example. What if we wanted to get the [shortest path distance](https://www.geeksforgeeks.org/shortest-path-unweighted-graph/) from a start node to an end node in a given graph?

	shortestDistance: graph start end >>
		tag #distance: (default: infinity)      // stores each node's distance from the start node. Default value is infinity

		for node in graph.nodes:
			if (node = start)
				node.#distance: 0
			else
				node.#distance: node.neighbors[[#distance]].get(Math.min)-> + 1    // find neighbor closest to start (aka shortest #distance), and add one to its distance

		=> end.#distance

In an imperative or functional language, such an example would require recursion and keeping track of visited nodes throughout the traversal. Not in Axis! We can use a simple for-loop to express the behavior. But how does this actually work?

Think back to the `testScores` example in the "Timeless" section. We can analyze how the interpreter _might_ execute something like `shortestDistance`. First, all `#distance` values are by default set to infinity. Then, the first node to be updated will be the start node, whose `#distance` is set to 0. Then, the neighbors of the start node will be notified of the update, and each node will look for the neighbor with the shortest distance,in this case the start node with a `#distance` of 0. Since the shortest distance is 0, each of these nodes will update their `#distance` to be 1. This will in turn notify the neighbors of these nodes, and the cycle will repeat until all shortest distances are computed. Due to the way this function was defined, we know that eventually, every node `#distance` will reduce to some final value, and stop sending updates to other nodes. We call this type of feedback "convergent".

Note that this method was only possible because we had a list of all graph nodes available through `graph.nodes`. This is provided in the `Graph` data structure, but let's look at how we might implement it ourselves. In order to do so, we have to fall back to using recursion:

	nodes: breadthFirstSearch((), this.root)
	breadthFirstSearch: visited, currentNodes >>
		if (current.length = 0)
			=> visited      // no more nodes to process, return all visited nodes
		else
			nextNodes: currentNodes[['neighbors']].subtract(visited)     // get neighbors of current nodes, minus already visited nodes
			=> breadthFirstSearch(visited, nextNodes)

Imperative code requires similar complexity to retrieve all nodes from a graph (either using loops or recursion). The difference is that in Axis, after we retrieve all the nodes, we can use them in a whole variety of use cases, like computing distances (as shown earlier), finding connected pairs, filtering for certain nodes, etc. We only have to define this recursion once. But in imperative languages, in order to do stuff like computing distance or finding connected pairs, we would have to use recursion over and over again.

This just demonstrates the power of abstracting data relationships away from implementation. We define the "implementation" once to retrieve the data (in this case, using breadth first search to get all nodes). Then we can use the data without worrying about the implementation again. We maximize abstraction and reduce code duplication.


Note that while feedback is very powerful, we do have to take care in using it. For example, if we did something like `x: x+1`, this would be a feedback loop that would constantly increment `x` until it reached infinity. Likewise, we would run into similar problems if we tried to modify our `shortestDistance` example to instead look for `longestDistance`. Intuitively, we know this wouldn't work, because imagine trying to find the longest distance in real life. We could just run around in circles before reaching the destination to make the path as long as we wanted. And this is exactly what would happen here with `shortestDistance` if we tried calculating longest distance by replacing `Math.min` with `Math.max` and setting the default distance to `0`. When the `#distance` of any node increases, the `#distance` of all of its neighbors will increase too, and the feedback would cause the distance for all nodes to steadly increase until infinity. This sort of feedback is called "divergent", and is something to watch out for, just like one might want to watch out for infinite loops in imperative languages.

### Web Applications

Let's take the chatroom example from the introduction, and this time include the layout:

	messages: collector(file: 'messages.axis')   // all messages, saved to a file
	activeUsers: collector

	ChatServer: Web.Server
		index: Web.Client template

			layout: JSX    // using jsx syntax
				<input class="username"/>
				for user in activeUsers
					<span> {user} </span>
				for message in messages.orderBy('time')->
					<p> {message.text} </p>
				<text-area class="message-draft"/>
				<button onclick={send}> Send </button>

			username: layout.find('input.username').value
			draft: layout.find('text-area.message-draft').value
			
			activeUsers <: username

			send: => @timestamp
				messages <: (time: timestamp, user: username, text: draft)

One of the difficulties when writing full-stack web applications is, often it can be ambiguous whether or not to put behavior in the server or in the client. For example, imagine if we wanted a web app that displayed all of a user's photos, potentially thousands of photos. On one hand, we could load the photos dynamically as the user scrolls through the album, but if the user has a slow internet connection, they would have to wait for photos to load every time they scrolled. On the other hand, we could proactively cache all photos on the client side so that the user can scroll smoothly, but then that would require a lot of memory on the client side. The best solution would probably be to do a mix of the two, caching only a few photos ahead of time, but this makes for a complicated dynamic caching mechanism. And while there's nothing wrong with complex optimizations like this, the problem is that current imperative languages force us to mix our business logic with our optimizations, so every optimization we make sacrifices flexibility and readability, and reduces code reuse.

Also, current web applications have a very strict separation between the server and client, with all communication handled through clunky HTTP requests or web sockets. Not only do they add lots of overhead, but they also make it difficult to migrate behavior between the server and client when we want to make changes. We are also forced to use HTTP requests when calling 3rd party APIs, which often leads to a complicated mess of callbacks, promises, or `async/await` patterns.

With Axis, everything is kept clean and simple. From the chat server example above, we can clearly see the distinction between database, server, and client, and more importantly, what each component means: the server is simply for data shared across clients, and the database is simply data shared across servers (or multiple runs of a server). In fact, notice that there is not a single line of code specific to the server, everything is either in the client or the global scope. This will actually be true for many web applications, and just goes to show how much of the server code we write today is just optimization or overhead.

In addition, not only is the communication between server and client seamless, but Axis also makes it easy to communicate with 3rd party APIs. Other servers on the internet can be treated like any other object, so programs and access properties, clone behavior, or insert data without worrying about callbacks or promises.

This high level of abstraction allows the Axis interpreter to make decisions on how to execute the code and manage the memory, so we don't have to worry about it. But we can also specify custom optimizations if we want. However, we can define those optimizations separately using meta-programming. We don't need to manually rewriting our code like we do for imperative languages. By treating execution the same as data, we can dynamically reconfigure and optimize our code. For example, for the photo library example talked about in the previous paragraphs, we might do something like

	PhotoLibraryOptimized: PhotoLibraryServer
		index: cacheOptimization template
			target: PhotoLibraryServer.index
			range: 10      // how many photos ahead and behind that we should cache
			cache: target.photos.slice(target.currentPhoto - range, target.currentPhoto + range)   // give a reference to all data that should be cached

From this, the interpreter will know what photos to cache, and might also reconfigure other behavior to fit with this optimization. The key thing to notice here is that the optimization is separate from the application logic. This means that we can read and alter the web application, without worrying about the optimizations. And this also means that we can apply this optimization to many different web applications. In fact, developers can upload their optimizations to public repositories. Then, other developers can import them and see if they help. Optimizations become like any other object, easy to import and tweak for one's own use.

### Testing and Mocking

coming soon!

### Firewalling

coming soon!

### Meta-programming

coming soon!


Acknowledgements
--------------------

Inspirations: Javascript, AngularJS, [Nylo](https://github.com/veggero/nylo), functional programming (Lisp, Ocaml, Haskell), Python, Verilog, Prolog

Similar languages: Smalltalk, Erlang, Pony
