+++
title = "CosmosDB + Gremlin"
date = "2019-08-12"
tags = ["cosmosDB", "azure", "gremlin", "javascript", "node.js", "graphson"]
categories = ["Graph Database"]
description = "A discussion on Azure's CosmosDB and its graph database offerings."
banner = "img/banners/banner-1.jpg"
+++
I’m writing this article after only two weeks of working with Gremlin and CosmosDB. What I’m writing about could be dead wrong. I honestly hope so, because my job would be much easier if I’m missing something and what little goodwill I had towards Azure before this experience might be restored.

This article assumes that you have an intermediate knowledge of TypeScript and a basic knowledge of Gremlin and CosmosDB. I won’t be stopping to explain the benefits of TypeScript or what Gremlin is and how it works, but I have included links to resources that do. If you’re feeling rusty, feel free to brush up using the following articles.

- [Getting Started with Gremlin](https://tinkerpop.apache.org/docs/3.1.0-incubating/tutorials-getting-started.html)
- [TypeScript in 5 minutes](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes.html)
- [CosmosDB Node.js Quickstart(Gremlin)](https://docs.microsoft.com/en-us/azure/cosmos-db/create-graph-nodejs)

On to business.

# The Players

- **[Gremlin](https://tinkerpop.apache.org/gremlin.html)** - Graph Traversal Machine and Language (how you communicate with some graph databases)
- **[CosmosDB](https://docs.microsoft.com/en-us/azure/cosmos-db/introduction)** - Microsoft Azure’s multi-model database—specifically, its graph database offering
- **[TypeScript](https://www.typescriptlang.org/)** - JavaScript’s sane younger brother
- **[Gremlin-JavaScript](http://tinkerpop.apache.org/docs/current/reference/#gremlin-javascript)** - Apache’s managed Gremlin-JavaScript implementation

<br>
# TypeScript and Gremlin-JavaScript

Before we start using CosmosDB, we need to correct the Gremlin-JavaScript package’s TypeScript type declarations. Don’t skip this section or you’ll learn what happens when type declarations don’t match up with their functionality counterparts.

The [@types/gremlin](https://www.npmjs.com/package/@types/gremlin) package contains incorrect declarations and is currently incomplete. I’m in the process of contributing to the official package, but that is a slow process. In the meantime, the best option is to use [Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html) to augment and correct current type declarations. The snippet below is my current, corrected type declaration file.

```
import * as gremlin from 'gremlin'

// The @types/gremlin package is out of date and/or incomplete. This file allows
// us to extend the gremlin types package so that Typescript can provide adequate
// typing information and linting. PLEASE VERIFY THAT THE ORIGINAL PACKAGE CONTAINS
// THE BEHAVIOR YOU'RE ATTEMPTING TO ADD.
declare module "gremlin" {
    export namespace driver {
        // auth was originally undeclared and its functions included in the driver namespace
        // however, the original gremlin package has the auth functions in a sub-folder of
        // driver, and as such must be declared as a nested namespace.
        namespace auth {

            class Authenticator {
                constructor(options?: any);
                evaluateChallenge(challenge: string): any;
            }

            class PlainTextSaslAuthenticator extends Authenticator {
                constructor(username: string, password: string, authzid?: string);
                evaluateChallenge(challenge: string): Promise<any>;
            }
        }
    }

    export namespace process {
        interface Traverser {
            object: any
        }

        interface Translator {
            of(traversalSource: string): void;
        }
    }

    export namespace structure {
        // CosmosDB returns an uuid for the ID field, not an integer. I've overridden
        // each return object type I'm currently using with the id field. Keep in mind
        // that you can't override constructors from within this file. If you need
        // that behavior, you'll have to find a better solution
        interface Element {
            id: string
            label: string
            value: any
        }

        interface Vertex {
            id: string
            label: string
            properties?: VertexProperty[]
        }

        interface Edge {
            id: string
            label: string
            inV: Vertex
            outV: Vertex
            properties?: Property[]
        }

        interface VertexProperty {
            id: string
            label: string
            value: any
            properties?: Property[]
        }

        interface Property {
            key: string
            value: any
        }

        // io suffered from the same problem as the auth package, the matching functionality
        // lived in a sub-folder in process. Declaring the functions in a nested namespace
        // solves this problem.
        namespace io {

            // these types are only needed if you're using the GraphSON v1 reader/writer
            // directly. You shouldn't need these declarations if you've decided to brave the NPM package.
            class GraphSONReader {
                constructor(options?: any);
                read(obj: any): any;
            }

            class GraphSONWriter {
                constructor(options?: any);
                adaptObject(value: any): any;
                write(obj: any): string;
            }

            class TypeSerializer {
            }

            class VertexSerializer extends TypeSerializer {
                deserialize(obj: any): structure.Vertex
                serialize(item: structure.Vertex): any
                canBeUsedFor(value: object): boolean
            }
        }

    }
}
```

<sub>_This is where my knowledge of TypeScript could use a little help. The least painful way I’ve found to augment declarations is to merge modules, so I don’t have to report and then import Gremlin from different locations. This does have a drawback: you can’t modify any of the constructors and might have to instantiate objects with an empty ID field. This generally isn’t a problem, since the majority of classes have extremely simple constructors, but that won’t always be the case._</sub>

I’ve corrected the most obvious errors for you, and have made a majority of the changes for CosmosDB to work, but these are probably not all the declaration changes you’ll have to make for your project. Don’t just plug this in and expect that all functionality has been covered. Be careful and pay attention.

<br>
# Gremlin and CosmosDB

You have two significant hurdles to overcome when dealing with CosmosDB through the Gremlin-JavaScript library:

#### _**CosmosDB does not support Gremlin bytecode commands**_

Gremlin works best when it can take the user commands and translate them into Gremlin bytecode. This helps avoid issues that can come about because of malformed or unescaped strings, and it allows the developer to use steps and traversal methods that would be too difficult or impossible otherwise. If you want more info, you can read all about Gremlin bytecode and why it’s a Very Good Thing™.

Without bytecode support, the [CosmosDB website](https://docs.microsoft.com/en-us/azure/cosmos-db/create-graph-nodejs) and [example packages](https://github.com/Azure-Samples/azure-cosmos-db-graph-nodejs-getting-started/blob/master/app.js) (even the [Gremlin-Javascript reference documentation](http://tinkerpop.apache.org/docs/current/reference/#_submitting_scripts_4)!) would have you believe that the only way to accomplish communication and queries against Gremlin is through raw script submission.

This is incorrect.

Included in the Gremlin-JavaScript package is a [nifty set of classes](https://github.com/apache/tinkerpop/blob/master/gremlin-javascript/src/main/javascript/gremlin-javascript/lib/process/translator.js#L24) for taking normal, fluent, traversal steps—minus the termination steps—and converting bytecode commands to a Gremlin/groovy script.

Note: Microsoft does not support all traversal steps; check out [this page](https://docs.microsoft.com/en-us/azure/cosmos-db/gremlin-support#gremlin-steps) for supported steps.

[Microsoft says](https://feedback.azure.com/forums/263030-azure-cosmos-db/suggestions/33632779-support-gremlin-bytecode-to-enable-the-fluent-api) they’ve begun work on accepting bytecode and that a public preview will be available in December 2019, but I won’t hold my breath for it becoming quickly available afterwards.

In the spirit of abstraction and longevity of the application, I’d suggest coding your app using the bytecode functionality and methods, and then using the script translator. You’ll thank me if/when CosmosDB enables bytecode support or you decide to find a much better alternative graph database provider. If you’re especially talented, you could probably make a fantastic abstraction layer that makes switching back and forth a breeze!

<br>
#### _**CosmosDB outputs GraphSON 1.0**_

[GraphSON](http://tinkerpop.apache.org/docs/3.4.1/dev/io/#graphson) is like JSON but for graph databases. When an SDK (in this case the Gremlin-JavaScript library) communicates with a Gremlin enabled graph database the data shared is serialized into GraphSON.

Simple.

There are three versions of GraphSON to date. Changes from [1.0](http://tinkerpop.apache.org/docs/3.4.1/dev/io/#graphson-1d0) to [2.0](http://tinkerpop.apache.org/docs/3.4.1/dev/io/#graphson-2d0) were very drastic, changes from 2.0 to [3.0](http://tinkerpop.apache.org/docs/3.4.1/dev/io/#graphson-3d0) not so much. Most modern database providers use either GraphSON 2.0 or 3.0 and the majority of SDK’s can serialize/deserialize 2.0 and 3.0 messages.

This is where things get frustrating.

CosmosDB accepts GraphSON 2.0 format, meaning that the Gremlin-JavaScript package’s serialization of Gremlin scripts will be accepted by CosmosDB right out of the box. However, CosmosDB outputs GraphSON 1.0—and none of the GraphSON serializers included with the Gremlin-JavaScript package accept GraphSON 1.0

Why CosmosDB accepts one version of GraphSON and outputs another is beyond me. I have both a [question on Stack Overflow](https://stackoverflow.com/questions/57174102/unable-to-get-cosmosdb-gremlin-endpoint-to-output-graphson-2-0) and a [post on Reddit](https://www.reddit.com/r/AZURE/comments/chi5ar/unable_to_get_cosmosdb_gremlin_endpoint_to_output/) asking that same question. I continue to hope I’m just missing a setting or not reading the documentation closely enough. But at the time of writing I have yet to receive any response, and, seeing that the UI tool on the Azure site mirrors the same GraphSON 1.0 output I get from communicating with the server directly, I am not filled with confidence that I will.

I’m in the process of writing a [GraphSON 1.0 reader/serializer](https://www.npmjs.com/package/gremlin-graphsonv1) for the Gremlin-JavaScript package, which you are free to use Keep in mind that it’s unfinished, and though it’s tested, I can’t guarantee it’s feature-complete or particularly good. If nothing else, it will be a demonstration on how the serializer/deserializer should function and where to start in writing your own.

<br>
# The End

I hope this article does not age well. Microsoft has stated they’re working on accepting Gremlin bytecode commands, and I hope that actually happens. As CosmosDB evolves, I hope the graph database offering will become more modern, outputting GraphSON 2.0. Most of all, I hope that Microsoft understands that these shortcomings are what’s keeping them from being as competitive in their cloud graph databases as they should be.

My recommendation? Avoid CosmosDB’s graph database service, for now, and try any of these [alternative solutions](http://tinkerpop.apache.org/providers.html).
