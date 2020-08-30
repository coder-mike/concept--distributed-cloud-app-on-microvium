# The Idea: A Distributed App on Microvium

2020-08-30

I [posted recently](https://coder-mike.com/2020/08/a-distributed-language/) about the idea of creating a distributed application, possibly in a new programming language. Here I'm going to consider whether Microvium is a suitable platform for this.

The objective here is to define a distributed application in a single "program". Distributed applications are normally more difficult to write, in part because of the plumbing required to connect up all the pieces running on different physical machines, and the separation between IaC code and application code. Some things that should be simple become more complicated than they need to be. I won't rehash the whole post.

The distinction between IaC code and application code is reminiscent of the distinction between compile time and runtime languages used in a typical C++ application (the compile time languages include the preprocessor language, template language, linker script, and make files). I've said many times how I don't like this. Microvium solves this with the [snapshotting feature](https://coder-mike.com/2020/05/snapshotting-vs-bundling/), allowing the same code to run across "compile time" and runtime, and the [Microvium Boost](https://coder-mike.com/2020/06/microvium-boost/) optimizer which performs accurate "tree shaking" to remove parts of code that don't apply to runtime.

I believe the same principle could possibly be applied to the division between IaC and application code in a cloud application. It might also be useful in closing the gap between code running on multiple different machines and services.

## The Vision

I imagine that the IaC code and runtime code are the same code, with execution split across multiple hosts, in the same way that Microvium was originally designed to split execution between a desktop machine and a microcontroller.

Let's say we want this:

 - A "program" that consists of a database and a lambda
 - The lambda inserts received messages into the database
 - The lambda is bound to an internet-visible endpoint at `https://mycompany.com/insert-object-into-my-database`

Here's a simplified example of what the code might look like (I'll need to work out some more of the details later):

```js
import { CosmosDB, FunctionsApp, Internet }  from 'azure-microvium';

// Create a new database
const db = new CosmosDB('my-database', { api: 'mongo-db' });

// Create a new Azure Functions app
const fn = new FunctionsApp('my-function', aMessage => {
  // Let's say we just want the Azure Functions App to capture the message into the database
  db.insert(aMessage);
};)

// When someone does an HTTP POST to the following URL, invoke the function
Internet.listenAt('https://mycompany.com/insert-object-into-my-database', 'POST', fn);
```

The key points:

  - This script includes both IaC code (e.g. `new CosmosDB`) and application code (e.g. `db.insert(aMessage)`) side by side. The former is executed as part of a deployment, and the latter is executed as part of the running application.

  - Because of the inclusion of both IaC and app code in the same program, there is a degree of type safety (assuming this is written in TypeScript) between them. In this example, the type safety doesn't give us much, but one can imagine situations where it's useful, as discussed in my previous blog post.

  - The assumption here is that this script is first executed at deployment time, establishing all the infrastructure in an idempotent way. In this example, it creates a database, Azure Function, and a public HTTP binding. Then the running script is deployed (copied by snapshot) to all the places it needs to be to resume execution in those environments.

  - I've chosen for the Functions App _not_ to provide binding information. This is normal in programming languages: a function declaration doesn't specify what should call the function. Rather, something outside the function declaration calls the function as it pleases.

  - Information about the created database is communicated privately from the IaC code to the application code by means of a closure. That is, the non-exported variable `db` is still in scope at the time that the Functions App handler is invoked. There is no need for a key vault or other external configuration system -- the connection information is passed implicitly here (I'll elaborate more on how the physical mechanisms might work).

There are some disadvantages of this approach, since JavaScript/TypeScript is not a language designed for distributed programming like this.

  - I think the part that disturbs me most about this hypothetical example is that it incorrectly implies that there is shared state between multiple invocations of the Functions App, and similarly between multiple Functions Apps and other service. In reality, there may or may not be, depending on how the infrastructure chooses to deploy these functions. But this is already an issue with Azure Functions Apps -- state may or may not be shared between multiple running instances of the Function (they are allowed to be in separate processes or the same process).

  - There are probably some other things which would be nice to include such as resource tagging, to make it easier to keep track of Azure costs. So the example itself might get a little bit larger when it's real than here in the minimalist ideal.

  - An elephant in the room is that Microvium is designed for microcontroller scripts. It only supports programs up to 64kB of memory, and the language feature set is limited. Nevertheless it would be quite reasonable for someone to make something _like_ Microvium that is more suited to the purpose, so I think that exploring this thought-experiment (and maybe POC) is worthwhile.

What about Azure Resource Groups?

I think that with azure resource groups, it probably makes sense to assume a "root" resource group into which resource are created. By default then, all resources will land up in the one resource group, which I personally think is fine. I don't really know any advantage of resource groups, other than as an organizational mechanism, but this entire proposed model offers a potentially-better way of organizing things anyway, so resource groups may not be needed (unless they offer some other advantage I haven't thought of).

## Mechanisms

It's all very well to define how an example _should_ look, but quite a different thing to define how one might get it to work that way. Here I'll go through some possible mechanisms.

### The Entry Point

Let's start by thinking about the entry point. In a perfect world, I can image that Azure natively supports this kind of application. In the Azure UI, you might say "Create Microvium Repo", or whatever Azure's trademarked name for this kind of thing will be.

And like the Heroku model, let's assume that this creates a new git repository whose master branch _is_ the deployed system. By default, the repo is empty (no commits, no files), so nothing is deployed, but you can add the repo as a git `remote` and push to it (or clone the empty repo and add files).

One file is special, because it marks the entry point. For a node application on heroku, the special file is `package.json`, which points to the entry script. We might as well copy this mode -- let's assume that an app must have a `package.json` in order for Azure to know what to do with it. This JSON file has only one requirement: that it have a "main" field that references the entry file.

```json
{
  "main": "main.js"
}
```

The difference between an "Azure Microvium" app and a Heroku app here is that the entry point is invoked _at deployment time_, not at runtime. It's essentially the entry to the IaC script.

Short of having this magic backed into Azure one day, it would be easy enough to set up a deployment pipeline manually that does exactly this. Unlike most deployment pipelines, this would be something that only needs to be set up once, and it only needs to do one thing: execute `main.js` with deployment-level privileges (authority to create Azure resources). The rest of the cloud application should "unfold" from this, including all the resources in the system, the databases, queues, networks, services, etc. One pipeline and one repository to describe the whole system (or at least, once per instance of the system.. of which there might be separate instances for production, UAT, etc). I suspect that in most cases, this is all that's needed.

### IaC Mechanisms

When `main.js` runs, it presumably imports its transitive dependencies, some of which may be the "services" in the system, and executes their IaC code as well. Dependencies can be passed down naturally in code and injected into the various services. For example `new MyService(theDatabase)` to inject the database into a service (where the service constructor is the IaC code for that service).

The `azure-microvium` library I've used in my example would encapsulate the details about creating Azure resources. This could either puppeteer the Azure CLI or Azure API, to create all the resources required.

The individual resource types would be handled by the corresponding constructors, such as `CosmosDB` and `FunctionsApp`, which I'll get to in a moment.

### The Database Connection

Let's think for a moment about the database connection specifically.

The constructor `new CosmosDB` can contact the Azure API to check if the database named `my-database` exists and, if not, create it. Whether it's newly created or pre-existing, the `CosmosDB` class can retrieve the sensitive connection string for the database and store it in a private field on the class -- it never needs to be accessible by anything outside the class.

When `publishChanges` captures a snapshot of the application and compiles it to bytecode, the `CosmosDB` instance will still be there, existing in the snapshot with its private connection string. The snapshot will then be distributed to various locations, including the hosted Function App `my-function`, where it carries this sensitive information inside it.

When the `my-function` function app is invoked, it calls `db.insert` which has access to the private db fields, including the connection string, giving it opportunity to establish the connection to the database. It's perfectly reasonable and feasible for it to handle connection pooling so that the connection is sustained across multiple calls to the function app.

### The Functions App

The constructor `new FunctionsApp` should create a new Azure Functions App, or updates the existing one with the same name.

There are a couple of ways this could work, but for simplicity let's say it works like this:

  - The `FunctionsApp` constructor itself contacts the Azure API to create a new Functions App with that name if it does not already exist. The returned information about the Functions App is stored in private fields in the `FunctionsApp` class instance.

  - A reference to the callback function (in this case the one that takes `aMessage` as a parameter) is also recorded as a private field on the class instance.

  - The constructor then forks the current program, by taking a snapshot of the current state of the program. The snapshot is then deployed to the Azure Cloud to act as the handler when the Functions App is invoked.

  - More concretely, it exports the entry point from the Microvium "assembly" (program) at a predefined export address, and then deploy to Azure a wrapper node.js application which has the capability to be invoked by Azure and in turn run the exported entry point.

The end result will be that the `FunctionsApp` constructor actually forks the process, with the second fork continuing in the `aMessage => ...` callback (on a separate machine), and the first fork continuing with the deployment/IaC code (in the deployment environment).

This is a little weird, but it may just be a consequence of trying to use a programming language like JavaScript to represent a program that runs across multiple machines.

Note that if we used this mechanism, then we need all the information about the Functions App upfront, such as its binding information. We can't determine its binding information upon _usage_, like I was hoping to do earlier. Maybe this isn't serious.

## Conclusion

I'm actually quite comfortable with this, and think it could be implemented on top of Microvium today (if I had time!), without any changes to the engine. It would be an interesting proof of concept, but because of the limitations of Microvium at this time, it wouldn't be something I would use in production.

The example, if it were to work, _is_ a whole cloud application, with multiple components interacting with each other, all within the same "program" code.

  - Dependencies can be injected directly from the IaC code into the application code, with type checking to validate that the contracts are correct.

  - Contracts can be defined in TypeScript (e.g. as interface types) and shared between the dependents and dependencies.

  - Services and sub-services can be organized however you want them in code, dividing the application up into modules (or you can keep everything in one file, like the example, but this will get out of control quickly). Organization does not need to be done by multiple distinct mechanisms (distinct resource groups, repositories, pipelines).

  - The definition of a new "service" can be done in a single line of code, in a high level language, with no interaction through a UI, nothing outside of source control, and everything described cohesively in a single language with the ability to pass anything around between code in that language (e.g. passing the new service to another service that calls it). There is a minimum of "plumbing code".



