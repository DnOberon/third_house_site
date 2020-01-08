+++
date = "2019-08-20"
tags = ["gremlin", "javascript", "node.js", "typescript"]
title = "TypeScript and Gremlin: Part 1"
categories = ["Typescript"]
banner = "img/banners/banner-2.jpg"
facebook_author = "GolangSociety"
+++
If you’ve read [CosmosDB + Gremlin + TypeScript = :|](https://notyourlanguage.com/post/cosmos_db/) - you’ll know that we’ve recently begun working with graph databases. Specifically graph databases that communicate using [Gremlin](https://tinkerpop.apache.org/gremlin.html).

This article demonstrates a TypeScript pattern for communicating with a graph database through the Gremlin query language. This article assumes you have an intermediate knowledge of TypeScript and know how to create, compile, and run a TypeScript project. In the interest of article length we will only be implementing a record creation method.

<br>
# Graph Databases

The most important thing to know about graph databases for this article is that every data record is a Vertex (or Node). There are no tables. There are no joins. There are only Vertices(Nodes) and Edges (Relationships). Vertices have _labels_ and it’s those labels that specify what type of record a Vertex is.

Here is a very simple example.

Let’s say you have a Person. That Person has a first name and a last name. In a traditional database you would have a table called “Persons” or “People” with the columns of “ID, FIRSTNAME, and LASTNAME”.

In a graph database however, you would simply create the Person record by inserting a Vertex labeled as “person” and with the properties of “id, first name, last name” along with their values.

This is a very simple example. For real life use-cases and example data I suggest [Capabilities of the Neo4j Graph Database with Real-life Examples](https://rubygarage.org/blog/neo4j-database-guide-with-use-cases). If you’re still unsure of what a graph database is or want more information, I recommend [Graph Databases Will Change Your Freakin Life](https://www.youtube.com/watch?v=GekQqFZm7mA).

<br>
# Prerequisite

Before we can start working with Gremlin in TypeScript, we need to fix the type declarations for the Gremlin-Javascript package. See “[TypeScript and Gremlin-Javascript](https://notyourlanguage.com/post/cosmos_db#typescript-and-gremlin-javascript)” in my previous article. Without the changes specified there you will not be able to connect to your graph database using Gremlin-Javascript. I also suggest visiting the links listed in [The Players](https://notyourlanguage.com/post/cosmos_db/#the-players) in the same article to give you a deeper idea of what Gremlin and Gremlin-JavaScript are.

<br>
# Gremlin Adapter Class

Connecting to Gremlin drills down to creating what’s called a Graph Traversal Source. This Graph Traversal Source is the entry point for **every single interaction with the actual data of your graph database**.

For example: The function `g.V()`, which returns all vertices in your graph database, the `g` refers to the GraphTraversalSource and `V()` is one of a special set of functions that creates a new [GraphTraversal](http://tinkerpop.apache.org/docs/current/reference/#traversal) from your Graph Traversal Source.

If it helps, think of the traversal as a kind of transaction, with the Graph Traversal Source representing the information on how and where of transaction execution. This might be an oversimplification of the process, but I’ll let you visit the [documentation](http://tinkerpop.apache.org/docs/current/reference/#traversal) and decide for yourself.

Let’s create a new class called `GremlinAdapter` and add a single property `g`. The type of `g` is `process.GraphTraversalSource` - just like the `g` in the Gremlin query you saw above. _(You \_could_ use something other than `g` but we’re going to follow common gremlin naming practice.)\_

```
import { process } from "gremlin"

class GremlinAdapter {
    public g: process.GraphTraversalSource
}
```

<br>
We’ll create the constructor next - its only job is to create a valid GraphTraversalSource and set the `g` property. I’m following parts of the Gremlin-Javascript [example](http://tinkerpop.apache.org/docs/current/reference/#gremlin-javascript) which you can visit if you’re still a little confused.
```
import { driver, process, structure } from "gremlin"

// we're going to shorten this class down a bit. You can see where this
// comes into play in the Gremlin documentation
const traversal = process.AnonymousTraversalSource.traversal

class GremlinAdapter {
    public g: process.GraphTraversalSource

    public constructor() {
        let config: any = {}
        config.traversalsource = "g"

        // We don't discuss it in the article, but here is an example of what you'd
        // need to do in order to have the GraphTraversalSource you create be able
        // to communicate to a secured Gremlin endpoint
        if (needsAuth) {
            const authenticator = new driver.auth.PlainTextSaslAuthenticator("user", "key")
            config.rejectUnauthorized = true
            config.authenticator = authenticator
        }

        // withRemote specifies that this source is not local and should be considered
        // a network resource
        this.g = traversal().withRemote( new driver.DriverRemoteConnection(
                `wss://url:port`, 
                config))
    }
}
```

<br>
# Add Vertex

Now that the `GremlinAdapter` can create a successful connection to our Gremlin enabled database, it’s time to add the ability to create a new vertex.

Given a Person with first name of John and last name of Doe, here is what the Gremlin command would look like if you entered via the console application.

g.addV(‘person’).property(‘first name’, ‘John’).property(‘last name’, ‘Doe’).

We specify first that the added vertex should be labeled a `person` vertex - `addV(‘person’)`. Then we add properties `.property(key, value).

The equivalent JavaScript code is not much different.

We’ll create a method on our `GremlinAdapter` class named `addVertex` ( I like to pattern the names of my classes and methods after their gremlin counterparts when possible). The caller of `addVertex` must specify a label, or the type of vertex that should be created, and provide an object full of properties. This function will not return a value yet.

```
 public async addVertex(type: string, input: any) {
        let write = this.g.addV(type)

        for (let key in input) {
            if (input.hasOwnProperty(key)) {
                if (typeof input[key] === 'object') {
                    write = write.property(key, JSON.stringify(input[key]))
                    continue
                }

                write = write.property(key, `${input[key]}`)
            }
        }
    }
```

For ease of use we’ll accept `any` as the parameter. I will forgo any validation or type checking in the interest of time and maintaining focus on the parts of the method that truly matter. _(We’re also not going to discuss setting anything other than a string as the value for a property. If you would like information on how to do that, see the bottom of the article for additional resources.)_

In order to complete and submit our Gremlin query we must call a [terminal step](http://tinkerpop.apache.org/docs/current/reference/#terminal-steps) - in this case `toList`. The `toList` method returns `Promise<ResultSet>`. `ResultSet` represents an array of records whose type, in this case, is vertex. Any errors `toList` encounters in attempting your traversal will be thrown as exceptions that you as the developer are responsible for handling.

In this case there should only be a single vertex record returned from this traversal - our newly created vertex. We use `ResultSet`’s 'first()' method to pull the first record, and then cast that to the `structure.Vertex` type included in the Gremlin-JavaScript package. I’d suggest running any additional checks on data structure and existence here as opposed to the caller of the `addVertex` method.

```
  public async addVertex(type: string, input: any): Promise<structure.Vertex> {
        let write = this.g.addV(type)

        for (let key in input) {
            if (input.hasOwnProperty(key)) {
                if (typeof input[key] === 'object') {
                    write = write.property(key, JSON.stringify(input[key]))
                    continue
                }

                write = write.property(key, `${input[key]}`)
            }
        }

        return new Promise((resolve, reject) => {
           write.toList() 
                .then((results) => {
                    let resultSet: driver.ResultSet = new driver.ResultSet(results)

                    let created: structure.Vertex = <structure.Vertex><unknown>resultSet.first()
                    resolve(created)
                })
                .catch((err) => reject(err))
        })
    }
```

# Usage

Usage is fairly simple.

```
let g = new GremlinAdapter()

g.addVertex('person', { "firstName": "John", "lastName": "Doe" })
.then((resp) => console.log(resp))
```

With that, we can successfully add a node to _most_ Gremlin enabled databases.

<br>
## Additional Resources

- [Gremlin-JavaScript](http://tinkerpop.apache.org/docs/current/reference/#gremlin-javascript)
- [Setting Up A New TypeScript Project](https://alligator.io/typescript/new-project/)
- [Practical Gremlin Guide](http://kelvinlawrence.net/book/Gremlin-Graph-Guide.htm)
- [Practical Gremlin Guide: Property Keys and Values](http://kelvinlawrence.net/book/Gremlin-Graph-Guide.html#pkvrevisited)
