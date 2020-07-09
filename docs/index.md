There are many promises of the upcoming [Internet Computer] (IC) platform, and one of them is to allow a simplified app development experience than conventional cloud computing platforms.
*Disclaimer:* I work for [DFINITY], the company that builds IC.
But I'm also a software developer at heart, and I want to evaluate the experience of building on top of IC from the view point of a web developer.

I chose to build a reversi game, not as a sample app, but as a real application complete with all bells-and-whistles that I'd imagine a multi-player reversi game might have.

![Reversi Screenshots](./screenshots.png)

Before I dive into the technical details behind the scene, I want to emphasize that this post is not about blockchains, which many have associated with IC.
Although IC uses blockchain technologies underneath, I'd like focus on the high-level concept: a virtual environment where Internet applications seamlessly connect to each other.

My personal belief is that with the evolution of cloud computing, infrastructure will become a commodity.
In other words, who provides the infrastructure no longer matters, what matters is:

> you write an application, and it runs on Internet.

# Programming Model

If I must find an analogy, the experience of developing a web application on IC is close to that of (the now defunct) [Parse] or [newer](https://en.wikipedia.org/wiki/Platform_as_a_service) [platforms](https://en.wikipedia.org/wiki/Function_as_a_service) of [a similar vein](https://en.wikipedia.org/wiki/Serverless_computing).
The basic premise of such a platform is to hide the complexity of building & maintaining backend services such as HTTP servers, databases, user login, and so on.
Instead, they present an abstract virtual environment that just runs user applications, without users being aware of, or having to pay attention to, where and how their applications are run.

Looking at it from this perspective, IC is both familiar and yet different.
The basic building block of an IC application is a [Canister], which conceptually is a live running process that:
- is 100% deterministic (if all inputs and state are the same, the output must be the same);
- is transparently persisted (also called [Orthogonal Persistence]);
- communicates with users or other canisters via asynchronous messages (remote function calls);
- processes one message at a time (following the [Actor model]).

If we think of a [docker container] as virtualizing an entire operating system (OS), a canister virtualizes a single program, hiding almost all OS details.
This seems overly restrictive since it can't run your favorite OS or database.
*What good does it do?*

I personally prefer to think in terms of *disciplines* rather than *restrictions*.
Just to highlight two properties (out of many) that make the canister model distinct from the usual web services:
- **Atomicity**: Each update to the state of a canister is atomic per message (remote function call).  A call is either successful and state is updated, or it throws an error and the state is not touched (as if the call has never happened).
- **Bi-directional messaging**: A message is delivered at most once, and the message caller is always guaranteed a reply, either success or failure.

Such guanrantees are hard to come by without restricting what a user program can do.
Hopefully by the end of reading this post, you can agree with me that a restricted canister model actually accomplishes a lot by taking a sweet spot between efficiency, robustness and simplicity.

# Client-server architecture

A multi-player game requires exchanging data between players, the implementation of which often follows a client-server architecture:
- A server hosts the actual game and manages communication with game clients;
- Two or more clients, each representing a player, fetch state from server, render game UI, and also take player inputs to forward to the server.

Building a multi-player game as a web application means the client has to run inside a browser, utilizing HTTP protocol for data communication, and using Javascript (JS) to render game UI as a web page.

Specifically for a multi-player reversi game, I want to implement the following functionalities:
- Any two players can choose to play a game against each other;
- Players gain points by winning a game, which also counts towards their accumulated scores;
- A score board shows the top players;
- And of course the usual game flow: taking inputs from each player in turn, enforcing only legal moves, and detecting the end of a game to count points.

Much of this game logic is about state manipulation, and a server side implementation helps to ensure so players can have a consistent view.

# Backend server

In a conventional backend setup, I will have to choose a suite of server-side software, including a database to keep player and game data, a webserver that serves HTTP requests, and then write my own application software connecting to both to implement the full set of server-side logic. 

In a "serverless" setup, usually the webserver and database services are already provided by the platform, I only need to write the application software calling into the platform to make use of these services.
Despite the misleading name "serverless", the application would still take the "server" role as prescribed by the client-server architecture.

Regardless of the backend setup, at the center of my application design is a set of APIs governing the communication between the game server and its clients.
Developing this application on IC is no different.
So I started with the following high-level design on the game flow:

![Game sequence](./game-sequence.png)

After players have registered, if any two of them indicate they want to play with each other (by the *start(opponent_name)* call), a new game will start.
Players then take turns to place their next moves, and the other player will have to periodically call *view()* to refresh its view on the newest game state, then make a next move and so on, until the game is over.
As a simple rule of thumb, a player is only allowed to play one game at any given time.

The server has to keep the following set of data:
- List of registered players, their names and scores, etc.
- List of on-going games, each will include the latest game board, who is playing black/white, who is allowed to make next move, and final result if it is ended, etc.

I chose to implement the server in [Motoko], but in theory, any language that compiles to [Wasm] can be made to work as long as it talks to IC using the same set of system APIs.
As of writing, a Rust SDK is coming *Really Soon Now*.

Being a new language, Motoko has some rough edges (e.g. its [base library](https://github.com/dfinity/motoko-base) is a bit lacking and not yet stable), but it has already gained a [package manager](https://github.com/kritzcreek/vessel) and [language server protocol (LSP) support](https://marketplace.visualstudio.com/items?itemName=dfinity-foundation.vscode-motoko) in VSCode, which makes the development flow rather pleasant (so I heard, because I'm a Vim user).

I will not go into the Motoko language itself in this post, and instead I'll discussion some notable Motoko or IC features that make canister development exciting.

## Stable variables

[Orthogonal Persistence] (OP) is not a new idea.
The new generation of computer hardware like [NVRam] has largely removed the barrier of treating all program memory as persistent, and concepts like file system are not essential to a program any more.
However, one challenge often mentioned in OP literatureis about upgrades, i.e., what happens when a new program version changes its data structure or memory layout?

Motoko answers the question with **Stable variables**.
They survive through upgrades, which in my case are ideal to hold registered player data.
In a conventional server side development, player data will have to be stored in files or in databases (which is an essential service of "serverless" platforms).
Motoko imposes several rules on stable variables, but besides that, they are just like any other variables that store data on heap, and can be used indistinguishably.

That said, there is currently a limitation preventing *HashMaps* from being used for stable variables, so I had to resort to arrays.
Here is an example:

```swift
// Instead of a list of player, we use HashMap for faster lookup.
type Players = {
  id_map: HashMap.HashMap<PlayerId, PlayerState>;
  name_map: HashMap.HashMap<PlayerName, PlayerId>;
};

actor {
  // A stable data structure that holds player data.
  stable var accounts : [(PlayerId, PlayerState)] = [];

  // A more efficient data structure used by the rest of program.
  let players : Players = {
    id_map   = ... // build HashMap<PlayerId, PlayerState> from accounts
    name_map = ... // build HashMap<PlayerName, PlayerId> from accounts
  };

  // before upgrade, we convert data from players to accounts.
  system func preupgrade() {
    accounts := Iter.toArray(players.id_map.entries());
  };
}
```

I hope in a future version of the DFINITY SDK, this limitation can be lifted so I can simply use *stable var players* without going through any conversion.

## User authentication

Every canister as well as every client (e.g., the *dfx* command line, or a browser) is given a **principal ID** that uniquely identifies them (For clients, such IDs are auto-generated from public/private keypairs, and DFINITY JS library manages them in a browser's local storage at the moment).
Motoko allows an canister to identify callers of "shared" functions and we can use them for authentication puroposes.

For example, I define *register* and *view* functions as follows:

```swift
public shared (msg) func register(name: Text): async Result.Result<PlayerView, RegistrationError> {
  let id = msg.caller;
  ...
}

public shared query (msg) func view() : async ?GameView {
  let player_id = msg.caller;
  ...
}
```
The *msg.caller* field returns the principal ID of the caller of an incoming message, which in this case is either *register* or *view*. (*view* is a query call, as marked by the *query* keyword).

Being able to access a unique ID of the caller (message sender) feels familiar.
A close concept is HTTP cookies, except that the underlying system of IC already did the heavy-lifting to make sure principal IDs are cryptographically secure and the user program running on IC can fully trust that their authenticity.

Although personally I think it is perhaps too powerful to let a program know its caller, and too rigid (e.g. what happens when such IDs have to change?).
At the moment it does lead to a very simple authentication scheme that application developers can make use of.
I hope to see more upcoming development in this area.

## Concurrency and atomicity

Game clients may send messages to the game server at any time, so it's the server's responsibility to handle concurrency requests properly.
With a conventional architecture, I will have to build some logic to sequentialize players' moves, usually through a messaging queue or mutex/lock.
However, with the actor programming model employed by canisters, this is automatically taken care of, and I do not have to write any code for it.
Messages are just remote function calls, and the canister is guaranteed to process one message at a time.
This leads to simplified programming logic -- I simply don't worry about concurrency problems.

Also, because canister state is only persisted after a message is fully processed (function call finishes execution), I do not worry about flushing memory to disk, state corruption, or anything like that.
It should be noted that this atomicity is per message, which corresponds to a *public* function that is annotated with an *async* return type (as seen above).
A public function is free to call any other non-async functions, and as long as it completes all execution without error, changed states are persisted (for update calls, more details is given below).

If I were to build this game with a conventional architecture, I would probably choose an actor framework too, e.g., [Akka] for Java, [Actix] for Rust, and so on.
Motoko offers native actors, joining the family of actor based programming languages such as [Erlang] and [Pony].

## Update call vs. Query call

This is a feature that in my opinion really improves the user experience for IC applications.
It brings them on par with those hosted by conventional cloud platforms (or orders of magnitude faster when compared to other blockchains).

It is also simple to understand and to use: any public function that does not need to change program state can be marked as a "query" call, otherwise it remains as an "update" call by default.

The difference is in both latency and concurrency:
- A Query call takes milliseconds to complete; an update call usually takes around 2 seconds or more.
- Query calls can be executed concurrently with great scalability; update calls are sequentialized (based on the actor model) and they offer atomicity guarantees.

As in the above code example, I was able to mark the *view* function as query call because it merely looks up the game that a player is playing and returns its state. 
In fact, most of the time when we browse the Web, we are doing query calls:
data is retrieved from servers but not modified on them.

On the other hand, I had to leave the *register* function as an update call, because it has to add a new player to the player list on a successful registration.
Although 2s may still sound like a lot, I'd like to remind people that many operations on the web today actually take more than 2s to complete, e.g., logging into your bank account, or placing a stock purchase order, just to name a few.

There is but one problem: when a player makes the next move, it has to be an update call too:

```swift
public shared (msg) func move(row: Int, col: Int) : async MoveResult { ...  }
```

If the game only refreshes its screen 2s after a player clicked the mouse, it will feel unresponsive and no one will want to play a game that lags this much.
So I had to optimize this part by reacting to user inputs directly on the client side without having to wait for the server to respond.
This means the frontend UI will have to validate player's move, calculate what pieces would be flipped, and show them on screen immediately.
It also implies whatever the frontend shows will have to match server's response when it comes, or we risk inconsistency.
But again, I believe any reasonable implementation of a multi-player reversi or chess game would do the same, regardless of whether its backend takes 200ms to respond, or 2s.

# Frontend client

DFINITY SDK provides a way to directly load an application's frontend in browsers.
It is not the same as ordinary HTML pages served from web servers, however; communication with backend canisters is via remote function calls, not HTTP.
This part is already handled by a [JS user library](https://www.npmjs.com/package/@dfinity/agent), so a JS programs only has to import a canister as if it is a JS object and call its public functions as if they are regular async JS functions of the object.
The [DFINITY SDK] has a set of tutorials on how to setup JS frontend, so I won't go into the details.
It uses [Webpack] for packaging resources including JS, CSS, image, and other resources you may have.
You can also combine your favorite JS framework such as [React], [AngularJS], [Vue.js], or something similar, with the DFINITY user agent library to develop the frontend.

## Main UI components

I'm relatively new to frontend development with only some brief experience with [React], but this time I chose to learn to use [Mithril] since I've heard many good things about it, especially its simplicity.
Also to keep things simple, I went with a design that has only two screens: 

1. A *Play* screen that allows players to enter their and opponent names, which leads to the Game screen.
   It will also show some tips and instructions, top player charts, recent players, and so on.
2. A *Game* screen that takes player inputs and communicates with the backend canister to render reversi board with latest state.
   It will also show player scores at the end of game, which then leads player back to the Play screen.

The snippet below shows the skeleton of the game frontend in JS.

```javascript
import reversi from "ic:canisters/reversi";
import reversi_assets from "ic:canisters/reversi_assets";
import "./style.css";
import logo from "./logo.png";
import m from "mithril";

document.title = "Reversi Game on IC";

function Game { ... }

function Play() { ... }

m.route(document.body, "/play", {
  "/play": Play,
  "/game/:player/:against": Game
});
```

There are a couple things to note:

- The main backend canister *reversi* is imported just like any other JS library.
  It can be seen as a proxy that forwards function call to the remote server, receives their replies, and transparently handles necessary authentication, signature signing, data serialization/deserialization, error handling, and so on.

- Another *reversi_assets* canister is also imported.
  I use it in order to fetch necessary assets other than JS and CSS, or what Webpack can pack as source.

- A *logo* image is also directly imported.
  This has to be configured in [Webpack] using *url-loader*, which essentially embeds the content of an image as a Base64 string, and use it as an image element.
  Good for small images, but not big ones.

- The final application is setup using [Mithril] with two routes, */play* and */game*.
  The latter takes player and opponent names as two parameters.

## Loading resources from the assets canister

This is something I struggled a bit since I was not familiar with loading DOM elements asynchronously in Javascript.

When *dfx* builds canisters, it also builds a *reversi_assets* canister that basically just packs everything from <i>src/reversi_assets/assets/</i> in it.
I use it to retrieve the sound file, and play it when a player makes a move.
It is not as direct as pointing to a mp3 file in the *src* field of a HTML element.
But if you are a frontend developer, you may already know how to do this from developing JS apps.

As of writing, my *src/reversi_assets/assets/* contains a single file: *put.mp3* which is a sound effect of putting down a reversi piece.
Here is how I load it:

```javascript
var putsound = null;

var start = function(...) {
  ...
  reversi_assets
    .retrieve("put.mp3")
    .then(function(array) {
      let buffer = new Uint8Array(array);
      var context = new AudioContext();
      context.decodeAudioData(buffer.buffer, function(res) {
        putsound = { buffer: res, context: context };
      });
    })
    .catch(function(err) {
      console.log("Asset retrieve error, ignore");
      console.log(err);
    });
  ...
}
```

When the start function is called (from an async context), it will try to retrieve *"put.mp3"* file from the remote canister.
On a successful retrieval, it uses Javascript facility *AudioContext* to decode the audio data and initialize the global variable *putsound*.

A call to *playAudio(putsound)* will play the actual sound if *putsound* has been properly initialized:

```javascript
function playAudio(sound) {
  if ("buffer" in sound) {
    var audioSource = sound.context.createBufferSource();
    audioSource.connect(sound.context.destination);
    audioSource.buffer = sound.buffer;
    if (audioSource.noteOn) {
      audioSource.noteOn(0);
    } else {
      audioSource.start();
    }
  }
}
```
Other resources can be loaded in a similar way.
I didn't use any image other than a logo, which is small enough to embed its source into Webpack
by adding the following configuration to *webpack.config.js*:

```javascript
  module: {
    rules: [
      ...
      { test: /\.png$/, use: ['url-loader'] },
    ]
    ...
  }
```

## Data exchange format

Motoko has a concept of "shareable" data, which means data that can be sent across canisters or language boundaries.
Apparently I wouldn't imagine a heap pointer in C is "shareable", but anything that can be mapped to JSON seems "shareable" to me.
For this purpose, DFINITY has developed an IDL (Interface Description Language) called [Candid] for IC applications.

Candid greatly simplifies how frontend talks to the backend, or how canisters talk to each other.
For example, the following is a snippet (incomplete) of the backend *reversi* canister as described by Candid:

```swift
type ColorCount = 
 record {
   "black": nat;
   "white": nat;
 };

type MoveResult = 
 variant {
   "GameNotFound": null;
   "GameNotStarted": null;
   "GameOver": ColorCount;
   "IllegalColor": null;
   "IllegalMove": null;
   "InvalidColor": null;
   "InvalidCoordinate": null;
   "OK": null;
   "Pass": null;
 };

service : {
  "list": () -> (ListResult) query;
  "move": (int, int) -> (MoveResult);
  "register": (text) -> (Result_2);
  "start": (text) -> (Result);
  "view": () -> (opt GameView) query;
}
```

As we can see, the *move* function is exported as a canister service.
It takes two integers as input (representing a coordinate), and returns a value of type *MoveResult* as result.
The *MoveResult* is a variant (aka. enum) that represents possible results and errors when a player makes a move.
Among them *GameOver* represents a game is finished and carries an argument *ColorCount* that counts the number of black and white pieces.

A candid file is automatically generated from Motoko canister source code, and automatically used by the JS user library, so there is no user involvement.
On the Motoko side, each candid type corresponds to a Motoko type, and each service corresponds to a public actor method.
On the JS side, each candid type corresponds to a JSON object, and each service corresponds to a member function of the imported canister object.

Most candid types have direct JS representations, some require a little conversion.
For example, *nat* is arbitrary precision in both Motoko and Candid, and in JS it is mapped to a [big.js] integer, so one has to use *n.toNumber()* to convert it to JS native number type.

Another gotcha is the *null* value in Candid (and Motoko's Option type) is represented in JSON as empty array *[]* instead of its native *null*.
This is to differentiate the case when we have nested options, e.g. Option<Option<int>>:

|Motoko       |Rust         |JS           |
|-------------|-------------|-------------|
|?(?1)        |Some(Some(1))|[[1]]        |
|?(null)      |Some(None)   |[[]]         |
|null         |None         |[]           |

Candid is very powerful, although on the surface it may sound a lot like Protocolbuf, or JSON, so why re-inventing the wheel? But there are very good reasons that go beyond what I can mention here, and I encourage people who are interested in this topic to read the [Candid Spec].

## Synchronize game state with backend

As mentioned previously, I use a trick to immediately react to valid user inputs without having to wait for the backend game server to respond.
This means the frontend only needs a confirmation from the game server (or rather, error handling if there is any) after the player makes a move.
It will receive the other player's move from the game server by periodically polling it with the *view()* function.

An implication of this design is that I had to repeat some of the same game logic in both backend (Motoko) and frontend (Javascript), which is not ideal.
Since Motoko compiles to Wasm, and Wasm can run in browsers, wouldn't it be nice if both frontend and backend can share the same Wasm module that implements core game logic?
This kind of sharing is only to share code but not state.
It may require some setup, but I think it is entirely possible.
I might give it a try in a future update.

Specifically for the reversi game, in certain conditions a player may be blocked from making any move, so the other player can make 2 consecutive moves or even more. 
In order to show every move a player makes, I chose to implement game state as a sequence of moves instead of just the latest state of the game board.
This also means by comparing the list of moves in the local state of fronend with what's returned by calling the backend *view()* function, we can easily know what has changed since a player made the last move, whose turn it is to make next move, and so on.

## SVG Animation 

The topic of doing animation with SVG perhaps does not really belong to this post, but at one point I was really stuck and couldn't make any progress.
So I think it is also worth sharing the lesson I've learned.

Here is the problem I had: animations do not start despite *repeatCount* setting.
Most online resources on SVG only gives example of \<animate\> that either repeats indefinitely or with a *repeatCount* setting.
They implicitly assumed that if an animation is to be shown once, it is started once the page is loaded (or with some delay setting).
However, with most one-page app frameworks like [React] or [Mithril], the page is often not reloaded but merely re-rendered.
So when I wanted to show a piece flipping side from white to black or black to white, it has to happen when the page re-renders, not when page loads.
I was missing this crucial difference, and only figured it out after many failed attempts.

So here is how I render an *animate* element (as a sub-element of SVG) using Mithril, where *rx* of an ellipse is changed from initial *radius* to *0* and back.

```javascript
m("animate", {
  attributeName: "rx",
  begin: "indefinite",
  dur: "0.4s",
  repeatCount: "1",
  values: [radius, radius, "0", radius].join(";"),
  fill: "freeze"
})
```

- *begin* is set to *indefinite* to mean that the animation has to be manually started.
- *fill* is set to *freeze* to mean that after the animation finishes, its ending state should stay.
- *values* is set to 4 values, where first two are repeated *radius* as a trick to start animation 
  after 0.1s (1/4 of *dur*) delay. This is because *begin* was already set to *indefinite*.

The main point is that animations should be started manually.
I use *setTimeout* to trigger it with 0s delay, which is a trick to wait until new UI elements prepared by Mithril are rendered in browser DOM:

```javascript
setTimeout(function() {
  document.querySelectorAll("animate").forEach(function(animate) {
    if (!animate.id.startsWith("dot")) {
      animate.beginElement();
    }
  })}, 0)
```

The above says any *animate* elements with an id not starting with "dot" will be started now.

# Take Away

[Internet Computer]: https://dfinity.org
[DFINITY]: https://dfinity.org
[Parse]: https://en.wikipedia.org/wiki/Parse_(platform)
[Canister]: https://sdk.dfinity.org/docs/developers-guide/working-with-canisters.html
[Orthogonal Persistence]: https://sdk.dfinity.org/docs/language-guide/motoko.html#_orthogonal_persistence
[Actor model]: https://en.wikipedia.org/wiki/Actor_model
[docker container]: https://en.wikipedia.org/wiki/Docker_(software)
[DFINITY SDK]: https://sdk.dfinity.org/docs/index.html
[Motoko]: https://sdk.dfinity.org/docs/language-guide/motoko.html
[Wasm]: https://webassembly.org
[NVRam]: https://en.m.wikipedia.org/wiki/Non-volatile_random-access_memory
[Akka]: https:;/akka.io
[Actix]: https://actix.rs
[Erlang]: https://www.erlang.org
[Pony]: https://www.ponylang.io
[React]: https://reactjs.org
[AngularJS]: https://angularjs.org
[Vue.js]: https://vuejs.org
[Mithril]: https://mithril.js.org
[Webpack]: https://webpack.js.org
[Candid]: https://github.com/dfinity/candid
[Candid Spec]: https://github.com/dfinity/candid/blob/master/IDL.md
[big.js]: https://github.com/MikeMcl/big.js