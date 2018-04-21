# Technical Sharing on Serverless Architecture
This repository contains visual aids and example scripts/manifests for a technical sharing done on serverless architectures, first presented on the 27th of April 2018 for the Agile Consulting & Engineering monthly technical sharing.

We utilise the [open-source fnproject](https://github.com/fnproject) for examples herein.

## Pre-Reading TODO

- Clone this repository locally.
- Run all commands from a terminal/command prompt at this repository.
- Commands are catered for *nix environment, if you can contribute the Windows equivalents, please do with a PR.
- Any `curl`-ed scripts here pose a security threat to your system. While I have verified at point of publishing that none of them are malicious, time and open-source contributions may change that.
- We shall refer to all shells, command prompts and terminals universally as terminal.

## Getting Started
### Download Fn
> **NOTE**: the Linux/OS X `curl` commands may take awhile to complete depending on your internet connection. At the end when it's done, you should see the following:

```
fn version 0.4.74

        ______
       / ____/___
      / /_  / __ \
     / __/ / / / /
    /_/   /_/ /_/`
```

#### Linux
Run the following script in a terminal:

```sh
curl -LSs https://raw.githubusercontent.com/fnproject/cli/master/install | sh
```
#### Mac OS X
Run the following script in a terminal:

```sh
brew install fn
# OR
curl -LSs https://raw.githubusercontent.com/fnproject/cli/master/install | sh
```
#### Windows
Download the latest version from [https://github.com/fnproject/cli/releases](https://github.com/fnproject/cli/releases).

- - -

### Fire Up Fn
Open a dedicated terminal at the repository's directory and run the following command to start Fn (dedicated because that terminal will not longer be used for anything other than Fn logs output):

```sh
fn start
```

This pulls the `fnproject/fnserver:latest` Docker image from the public Docker registry and runs it.

Verify that you can access it by visiting http://localhost:8080 and yielding a `{"goto":"https://github.com/fnproject/fn","hello":"world!"}` response:

```sh
curl -sSL http://localhost:8080 | grep 'hello":"world!' && echo 'fn is accessible!';
```

- - -

### Initialise a Project
Run the following script to create an `example` directory to store your function:

```sh
fn init --runtime node example
```

Observe the newly created directory `example` and its contents:

```md
- example
  - func.js       (-> code)
  - func.yaml     (-> fn definitions)
  - package.json  (-> node package management)
  - test.json     (-> contract test schema)
```

- - -

### Run the Project Locally

Navigate into it (`cd example`) and test drive it:

```sh
fn run
```

This initiates a Docker build process (explained more later).

You should see:

```
Building image example:0.0.1 ............................
{"message":"Hello World"}
```

#### Quirk #1: Build-Time Errors
Let's see what happens when we inject a build-time error. Modify `func.js` so that it looks like:

```js
// func.js
var fdk=require('@fnproject/fdk');

fdk.handle(function(input){
  var name = 'Worl
  if (input.name) {
    name = input.name;
  }
  response = {'message': 'Hello ' + name}
  return response
})
```

Run `fn run` again. Observe the error. Note that while running `node func.js` would have yielded the error immediately, Fn's build process still runs successfully despite language syntax errors.

#### Quirk #2: Run-Time Errors
Now let's see what happens when we inject a run-time error. Modify `func.js` so that it looks like:

```js
// func.js
var fdk=require('@fnproject/fdk');

fdk.handle(function(input){
  throw new Error('what now doc?')
  return response
})
```

Do an `fn run` and observe the behaviour. Nothing special about this one.

#### Quirk #3: Logging Errors
Finally, let us attempt to log a message for easier debugging:

```js
// func.js
var fdk=require('@fnproject/fdk');

fdk.handle(function(input){
  console.info('hi');
  const response = {'message': 'Hello '}
  return response
})
```

Do an `fn run` now, and observe the message:

```
error decoding invalid character 'h' looking for beginning of value
```

In a nutshell, we can no longer use the various `console.*` we are accustomed to. The only method still available is `console.error`. Try change the line `console.info('hi');` to `console.error('hi');` and we'll get what we were looking for:

```sh
# output
Building image example:0.0.1 .
hi
{"message":"Hello "}
```

#### TL;DR
1. you can't rely on the build process, always run your function before dpeloying
2. throw your errors, this way, it'll always be in the logs
3. use `console.error` or `process.stderr.write` for debugging purposes

- - -

### Running the Project on Fn

Revert `func.js` to the original state:

```js
// func.js
var fdk=require('@fnproject/fdk');

fdk.handle(function(input){
  var name = 'World';
  if (input.name) {
    name = input.name;
  }
  response = {'message': 'Hello ' + name}
  return response
})
```

Now let's deploy it to an example application, `example`, using:

```sh
fn deploy --app example --local;
```

Observe the output:

```
Deploying example to app: example at path: /example
Bumped to version 0.0.2
Building image example:0.0.2 .
Updating route /example using image example:0.0.2...
```

Let's test it by curling it now:

```sh
time curl http://localhost:8080/r/example/example
```

The first time we run this, we notice a lag time. Your output probably looks like:

```
{"message":"Hello World"}curl http://localhost:8080/r/example/example  0.02s user 0.01s system 21% cpu 3.083 total
```

Note the last value (`3.083`) is in seconds. It took us 3 seconds to run the function.

Now run the same command again and notice that that number drops to `0.*`. This is due to the way Fn manages your functions.

- - -

### Testing an Fn function
The Fn Node template comes with a `test.json` that provides an easy way to do contract testing as a unit test. Open the file in a text editor and observe that it defines an array of two test cases, each case containing an `input` and `output`.

When running this in-built contract test mechanism, Fn sends the input to your application and checks to see if the output matches the input. If it does, it passed. If it doesn't, it failed.

Let's try it out:

```sh
fn test
```

Observe the output:

```
Building image example:0.0.2 .
Running 2 tests on /path/to/ts-serverless-20180427/example/func.yaml (image: example:0.0.2):
Test 1
PASSED -    ( 2.846489226s )

Test 2
PASSED -    ( 2.623483408s )

tests run: 2 passed, 0 failed
```

- - -

### Building an Fn function
Should we wish to just build the function, there's a command for that:

```sh
fn build
```

This builds the function using information from `func.yaml` but does not run it. When building a build/release pipeline for deploying functions, remember that **syntax errors will not show up at build-time if we are using an interpreted language**.

So what happens at build-time? Let's challenge it with some unexpected values. Add a new dependency to `package.json` **manually** by opening the file and adding a new line under the `"dependencies"` property:

```json
{
  ...,
  "dependencies": {
    "@fnproject/fdk": "0.x",
    "this-definitely-doesnt-exist": "nope"
  }
}
```

Spoiler: this will error out, so run the build command as such to get logs:

```sh
fn --verbose build
```

Note that the build process is essentially a `docker build .` with special parameters. So Fn uses Docker containers under the hood!

- - -

## Assessing Performance
### Setting up Fn variant
Create a new project using Fn:

```sh
fn init --runtime node withoutserver
```

Change the `func.js` so that it looks like:

```js
// withoutserver/func.js
var fdk=require('@fnproject/fdk');

fdk.handle(function(input){
  response = 'hi';
  return response;
});
```

Deploy it using:

```sh
fn deploy --app example --local
```

Test it is working:

```sh
curl http://localhost:8080/r/example/withoutserver
```

### Setting up Vanilla variant
Create a new directory with a `server.js` file manually:

```sh
mkdir withserver;
touch server.js;
```

Paste the following code into `server.js`: 

```js
// withserver/server.js
const http = require('http');

const server = http.createServer((req, res) => {
  res.end('"hi"');
});
server.listen(8081);
```

Run the server:

```js
node ./server.js
```

Open a new terminal and test that it works:

```sh
curl http://localhost:8081
```

Observer that the output is exactly the same for the Fn and the Vanilla variants. Let's put them to the test.

### Introducing Autocannon
Autocannon is a handly tool for running targetted load tests against an API endpoint. Install it with:

```sh
command -v autocannon || npm i -g autocannon && echo 'autocannon is installed';
```

Let's use Autocannon to simulate a huge load coming into Fn and Vanilla Node:

```sh
autocannon --connections 10 --duration 30 --connectionRate 10 -T withoutserver "http://localhost:8080/r/example/withoutserver";
sleep 1;
autocannon --connections 10 --duration 30 --connectionRate 10 -T withserver "http://localhost:8081";
```

For the variant using Fn, my machine reports:

```
Running 30s test @ http://localhost:8080/r/example/example__withoutserver
10 connections

Stat         Avg     Stdev   Max
Latency (ms) 338.97  799.04  7248.65
Req/Sec      24.37   21      58
Bytes/Sec    4.37 kB 3.78 kB 10.3 kB

731 requests in 30s, 130 kB read
5 errors (5 timeouts)
```

For the variant using vanilla Node, it is:

```
Running 30s test @ http://localhost:8081
10 connections

Stat         Avg     Stdev   Max
Latency (ms) 3.97    5.14    41.65
Req/Sec      100     10.81   133
Bytes/Sec    10.3 kB 1.14 kB 13.7 kB

3k requests in 30s, 309 kB read
```

Note that compared to vanilla node, a serverless architecture:

- has 100x more latency than vanilla node
- processes requests 5x slower than vanilla node
- reads data about 2x as slow

On overall, given no processing and perfect conditions, a serverless architecture performs about 3x worst (3k requests for vanilla node vs 731 requests for serverless).

So why serverless?

- - -

## Serverless Usage


`-- WIP --`

