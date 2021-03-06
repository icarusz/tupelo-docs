---
layout: default
title: Notebook
parent: Tutorials
nav_order: 2
---

# Building a Notebook
We're going to build a command-line timestamped note-taking application using
the [Tupelo WASM SDK](https://github.com/quorumcontrol/tupelo-wasm-sdk).
When complete, we'll be able to use this app to save timestamped notes.
All of these notes will be signed by the Tupelo TestNet as they
are entered adding a level of outside verification and trust to our simple
application.

## Getting started
Our notebook will be a [Node.js](https://nodejs.org/en/ "Node.js") application.
We'll manage its dependencies with [NPM](https://www.npmjs.com/ "NPM"), so make
sure you have NPM installed before going further.

To start, we will create and enter the directory that will hold our notebook
application's code. We will then initialize a new NPM application with `npm init`.
The `npm init` defaults are fine for our purposes, so we'll use the `-y` option
to skip any confirmation questions.

```bash
# Create the 'notebook' application directory
mkdir notebook

# Change to the application directory
cd notebook

# Initialize a new npm application
npm init -y
```

This gives us a new directory with a `package.json` skeleton to start with.

## The Tupelo WASM SDK
Before we build anything, we need to add the Tupelo WASM SDK to our application's
dependency set. Edit the new `package.json` file in your favorite editor and add a
new dependency set containing the `tupelo-wasm-sdk` library:

In file `notebook/package.json`:
```json
{
  "name": "notebook",
  ...
  "dependencies": {
      "tupelo-wasm-sdk": "latest"      
  },
}
```

Next, run `npm install` from the application directory to install the dependency
to your local cache.

After installing `tupelo-wasm-sdk`, make a new file called
`index.js` with your favorite editor to hold the application code. At the
top of that file, require the Tupelo wasm sdk so we can use it later.

In file `notebook/index.js`:
```javascript
const tupelo = require('tupelo-wasm-sdk');
```

## Creating a new notebook
Our application will organize notes into _notebooks_, and we'll store the data
associated with a particular notebook in its own, unique ChainTree.  You can read more
about ChainTrees [here](/docs/chaintree).  Chaintrees are the flexible datastructure that underlies
Tupelo.  We can store any type of data we want in a ChainTree and organize it within
a path structure.

Before we can start saving notes, we need to lay some groundwork.  We will need to be able
to create keys for the user, connect to a service to sign and track our ChainTrees,
and create a new empty ChainTree to store our notes.

Let's build a `createNotebook()` function, step by step, to do those things.

First we will connect to a service to act as our _community_, in `notebook/index.js`:
```javascript
async function createNotebook() {
    console.log("creating notebook")
    let community = await tupelo.Community.getDefault();  
}
```

The _community_ the SDK is connecting to in this example is the Tupelo TestNet.  
The underlying code creates a p2p node and establishes the connections it needs to submit
transactions and get back confirmations when transactions are finalized.  The Tupelo
network is fast so the user can wait the few hundred milliseconds required to process
requests in real time.  In this way the Tupelo WASM SDK is as easy to use as a standard
database API.

### Generating keys

We will need to generate a new public/private keypair for the user of our wasm app in
the `createNotebook()` function.

In file `notebook/index.js`:
```javascript
async function createNotebook() {
    console.log("creating notebook")
    let community = await tupelo.Community.getDefault();
    const key = await tupelo.EcdsaKey.generate()   // <--- Create a digital signature for the user
}
```

After generating our new key we will use the Tupelo SDK to create a new empty
ChainTree to write our notebook entries into.  Next we will write a setDataTransaction
to that tree which will get signed by the Tupelo TestNet. 

```javascript
async function createNotebook() {
    console.log("creating notebook")
    let community = await tupelo.Community.getDefault();
    const key = await tupelo.EcdsaKey.generate()   
    const tree = await tupelo.ChainTree.newEmptyTree(community.blockservice, key)
    await community.playTransactions(tree, [tupelo.setDataTransaction(CHAIN_TREE_NOTE_PATH, [])]);
}
```

### Storing identifiers

Now that we have generated keys and a ChainTree for our notebook, we will need a way to
persist the required information locally between invocations of the app.

We could use a database for this in a full-fledged production app, but an external file
is enough to serve our purposes.  The `fs` module handles file i/o in Node.js so we add
a filesystem require for that:
```javascript
const tupelo = require('tupelo-wasm-sdk');
const fs = require('fs');
```

We will also need a file and path to write the values to.  We will specify that right
after our module declarations.
```javascript
const tupelo = require('tupelo-wasm-sdk');
const fs = require('fs');

const LOCAL_ID_PATH = './.notebook-identifiers'; // <--- Specify the file to save to
...
```
While we are figuring out where to put things we should also specify where in our ChainTree
we want to store our data.  For more complex applications we will want to use different paths 
to keep our data organized and potentially manage permissions, but for a simple application like our
notebook a single path will do.

In file `notebook/index.js` we will add a constant to store that.
```javascript
...
const LOCAL_ID_PATH = './.notebook-identifiers'; 
const CHAIN_TREE_NOTE_PATH = 'notebook/notes'; // <--- Specify the path in the tree to save to
...
```

Then we will create a few new functions to help us store our identifiers.
The first one will compose our identifier object.  

```javascript
async function identifierObj(key, chain) {
    return {
        unsafePrivateKey: Buffer.from(key.privateKey).toString('base64'),
        chainId: await chain.id()
    };
}
```

Note that we are calling out explicitly with the variable name that our handling of the
users private key is not safe.  

We will be storing it locally in a file and in plain text.  For a real application that
is clearly unacceptable.  Depending on your applications requirements there are a variety
of options to consider concerning the best practices in key management.

The other new function we need will write our identifier object into the file we specified
in our LOAD_ID_PATH.

```javascript
function writeIdentifierFile(configObj) {
    console.log("saving identifierFile: ", configObj)
    let data = JSON.stringify(configObj);
    fs.writeFileSync(LOCAL_ID_PATH, data);
}
```

Back in our createNotebook() function we will call those two new functions with our
key and ChainTree to compose our identifiers and then write the identifiers
into that file.

```javascript
async function createNotebook() {
    console.log("creating notebook")
    let community = await tupelo.Community.getDefault();
    const key = await tupelo.EcdsaKey.generate()
    const tree = await tupelo.ChainTree.newEmptyTree(community.blockservice, key)
    await community.playTransactions(tree, [tupelo.setDataTransaction(CHAIN_TREE_NOTE_PATH, [])]);
    let obj = await identifierObj(key, tree);
    return writeIdentifierFile(obj);
}
```

### Testing notebook creation

Let's test out our progress so far. Start a node repl in your project
directory with the `node` command. Then, from the node prompt run

```javascript
> .load index.js
> createNotebook();
```

This sequence of commands loads our `index.js` file as if we'd typed
each line into the repl and then runs the `createNotebook()` function.

You should see a confirmation that we are saving our writeIdentifier with a PrivateKey
and a chainID in the console.  As long as you see those, everything is on track and
we can run the .exit command in the node repl session.  

Back at the command line we can see a .notebook-identifiers file with our key and
chainID in it.

If your repl session is not responding as expected you can look for differences between
your index.js and this [`index` file](/tutorials/notebook/index1_js).

----------------------------

## Adding a note to the notebook

Now that we have a notebook it is time to start writing our notes into it.

We start by creating an addNote function accepting an argument
of the value for the new note.

```javascript
async function addNote(note) {

}
```

### Retrieving the key and notebook

Before we write our new note we need to make sure we have the identifiers
we need.  We just created and stored those in .notebook-identifiers above so create
a function to grab that.  

We start the readIdentifierFile by opening the file at our LOCAL_ID_PATH
and then grab and parse the key we stored, translating it into the appropriate form.

```javascript
async function readIdentifierFile() {
    let raw = fs.readFileSync(LOCAL_ID_PATH);
    const identifiers = JSON.parse(raw);
    const keyBits = Buffer.from(identifiers.unsafePrivateKey, 'base64')
    const key = await tupelo.EcdsaKey.fromBytes(keyBits)
}
```

Then we connect to the same default community service we had used to create our notebook
ChainTree.

```javascript
...
const community = await tupelo.Community.getDefault()
}
```

We proceed to grab the "tip" of our ChainTree which represents the latest version
of it from the community service.  We need to pass in the chainID from the identifier file
to grab the proper notebook.  We can only update a ChainTree we own. Once we have found
the tree we load it up so we can use it locally.

``` javascript
...
let tree
try {
    const tip = await community.getTip(identifiers.chainId)
    console.log("found tree")
    tree = new tupelo.ChainTree({
        store: community.blockservice,
        tip: tip,
        key: key,
    })
}
...
```

We will want to catch an error and create a new empty ChainTree if we could not
find our existing one for some reason.

``` javascript
...
} catch(e) {
    if (e === 'not found') {
        tree = await tupelo.ChainTree.newEmptyTree(community.blockservice, key)
    } else {
        throw e
    }
}
...
```

Upon success, we return the notebook ChainTree and the key we have retrieved.

``` javascript
  return { tree: tree, key: key }
}
```

The above steps results in the following full readIdentifierFile() function:

``` javascript
async function readIdentifierFile() {
    console.log("reading identifiers")
    let raw = fs.readFileSync(LOCAL_ID_PATH);
    const identifiers = JSON.parse(raw);
    const keyBits = Buffer.from(identifiers.unsafePrivateKey, 'base64')
    const key = await tupelo.EcdsaKey.fromBytes(keyBits)

    const community = await tupelo.Community.getDefault()
    let tree
    try {
        const tip = await community.getTip(identifiers.chainId)
        console.log("found tree")
        tree = new tupelo.ChainTree({
            store: community.blockservice,
            tip: tip,
            key: key,
        })
    } catch(e) {
        if (e === 'not found') {
            tree = await tupelo.ChainTree.newEmptyTree(community.blockservice, key)
        } else {
            throw e
        }
    }

    return { tree: tree, key: key }
}
```

### Writing the note

Now that we have retrieved our key and notebook ChainTree we can proceed
towards inserting new notes.

To build the actual addNote function, we start by grabbing our identifiers and
whatever notes already exist. Because ChainTrees are so flexible, this data can be of
nearly any type.  We will be storing our notes in an array of strings at 'notebook/notes'.

``` javascript
async function addNote(note) {

    let { tree } = await readIdentifierFile(); // Using our new function to retrieve

    const resp = await tree.resolveData(CHAIN_TREE_NOTE_PATH);
    let notes = resp.value,
        noteWithTs = addTimestamp(note); // Add a time and date to our new entry

    if (notes instanceof Array) {
        notes.push(noteWithTs);
    } else {
        notes = [noteWithTs];
    }

}
```

You will note that along with our text values we have decided to add a timestamp.  
This will provide additional immutable information to our notebook to be signed by
the Tupelo TestNet.  Our application will append the current date and time when each of
our notes was entered.

We will need to add a simple function to support that timestamping.

``` javascript
function addTimestamp(note) {
    let ts = new Date().getTime().toString();
    return ts + '::' + note;
}
```

The last step we need to take in our addNote function is to actually submit the
information to be signed!  So we will add the SDK calls to the end of addNote()
to do just that.  The playTransactions call takes the tree we are changing
and the change we want to make to the data as arguments.  

``` javascript
    ...
    console.log("saving new notes: ", notes)
    let c = await tupelo.Community.getDefault()
    await c.playTransactions(tree, [tupelo.setDataTransaction(CHAIN_TREE_NOTE_PATH, notes)])
}
```

We should have our new state confirmed in a second or two.

### Making sure the notebook exists

We want to make sure that everything is done in the right order so we will add a
small function to confirm we have an id file created.

``` javascript
function idFileExists() {
    return fs.existsSync(LOCAL_ID_PATH);
}
```
We will want to call that check at the very beginning of our addNote() function.  If
there is no ID file we will warn the user and end there.

``` javascript
async function addNote(note) {
    if (!idFileExists()) {
        console.error("Error: you must register before you can record notes.");
        return;
    }
  // ... additional body of addNote ...  
}
```

After these changes our index.js should look more or less like this
[`index.js` file](/tutorials/notebook/index2_js).

### Testing add note

Now's a good time to test our `addNote()` function. Start another node repl session in
the notebook directory with the `node` command, then run these next few commands:

```javascript
> .load index.js
> createNotebook();
> addNote("Super Awesome Note of Consequence.");
```

Assuming everything worked, you should see a "Saving Registration" message at
the node repl after the createNotebook.

In response to "addNote" you should get a confirmation
`saving new notes: [ '1577129740043::Super Awesome Note of Consequence.' ]`
The first time you save a note you should also get confirmation that "nothing was resolvable"
because its the first time any data was placed there.  We can add additional notes
and the array of notes will grow but we will not see that message again.

Assuming everything worked terminate the node repl session with the `.exit` command.
Our ChainTree has data in it and it was successfully updated, verified and signed off
by the Tupelo TestNet.

### Displaying our notes

Since we can save signed notes to our chain tree, let's add a way to print out all the
notes we've recorded so far. We'll write a `showNotes()` function that fetches
the saved notes using existing functions and print each one to the console.

In many ways showing notes is not all that different from adding notes.  The first
step for both is opening our identifier file and retrieving existing data.

First we want to make sure the user has registered.
Then we populate our ChainTree of notes locally.

It is then a simple matter of resolving the data from our CHAIN_TREE_NOTE_PATH
and cycling through each value in the array printing them to the console.

```javascript
async function showNotes() {
    if (!idFileExists()) {
        console.error("Error: you must register before you can print notes.");
        return;
    }

    let { tree } = await readIdentifierFile();
    let resp = await tree.resolveData(CHAIN_TREE_NOTE_PATH)
    let notes = resp.value

    if (notes instanceof Array) {
        console.log('----Notes----');
        notes.forEach(function (note) {
            console.log(note);
        });
    } else {
        console.log('----No Notes-----');
    }
}
```
If there are no notes we simply output that.

----------------------------

## Creating a command line interface

Now that we have all the background functions we need to manage notes with our
application, let's add a simple command line interface to tie everything together.
We'll use the [Yargs](https://github.com/yargs/yargs "Yargs") library for the
CLI, so let's add that as a dependency to our `package.json` and require it at the
top of our `index.js` file.

In file `notebook/package.json`:
```javascript
{
    "name": "notebook",
    ...
    "dependencies": { 
        "tupelo-wasm-sdk": "latest",
        "yargs": "^12.0.2"
    },
```

Next run `npm install` to get yargs loaded.

In file `notebook/index.js`:
```javascript
const tupelo = require('tupelo-wasm-sdk');
const fs = require('fs');
const yargs = require('yargs');
...
```

Next we'll follow the `yargs` documentation to define 3 commands: `register`,
`add-note`, and `print-notes`.

In file `notebook/index.js`:
```javascript
yargs.command('register', 'Register a new notebook chain tree', (yargs) => {
}, async (argv) => {
    await createNotebook();
    process.exit(0)
}).command('add-note', 'Save a note', (yargs) => {
    yargs.describe('n', 'Save a note')
        .alias('n', 'note')
        .demand('n');
}, async (argv) => {
    await addNote(argv.n);
    process.exit(0)
}).command('print-notes', 'Print saved notes', (yargs) => {
}, async (argv) => {
    await showNotes();
    process.exit(0)
}).argv;
```

## Finishing up

Together we've built a command line notebook that when invoked with `node`, can
record timestamped notes into a ChainTree, have the Tupelo TestNet sign each note
as we save them, and print them out later for prosperity.

You can run `node ./index.js register` to register a new notebook,
`node ./index.js add-note -n <note>` to save a note, and finally,
`node ./index.js print-notes` to print all the saved notes.

Be sure to take a look at the final
[`package.json` file](/tutorials/notebook/package_json) and the final
[`index.js` file](/tutorials/notebook/index3_js) for reference.

This tutorial just scratches the surface of how Tupelo can be used as a building block
of trust.  You can see more examples leveraging the WASM SDK to do **[ownership transfers](https://github.com/quorumcontrol/tupelo-wasm-sdk/tree/master/examples/ownership-transfer), [setting and getting data](https://github.com/quorumcontrol/tupelo-wasm-sdk/tree/master/examples/set-and-get-data) or [issuing and transferring tokens](https://github.com/quorumcontrol/tupelo-wasm-sdk/tree/master/examples/tokens)**.

Check out further [examples](/examples) such as a
[decentralized mobility application](/examples/decentracar) or hop into our
[developer chat]((https://t.me/joinchat/IhpojEWjbW9Y7_H81Y7rAA))
and we will be happy to answer any questions or discuss how Tupelo might help build
the trust a DLT provides into your application.
