+++
title = "Typescript and Gremlin: Part 2"
date = "2019-09-23"
tags = ["gremlin", "javascript", "node.js", "typescript"]
categories = ["Typescript"]
banner = "img/banners/banner-3.jpg"
+++
This article demonstrates a pattern for accessing graph databases in TypeScript. Make sure you read [Part 1](https://notyourlanguage.com/post/gremlin-typescript-1/) - without it, these next steps won’t make sense. This article assumes an intermediate knowledge of TypeScript and how generics and interfaces work within the language. This article also mentions the [Data Mapper](https://www.js-data.io/docs/data-mapper-pattern) and [Factory](https://en.wikipedia.org/wiki/Factory_method_pattern) patterns.

# Graph Database Interface

Most books about program design will spend some time talking about the benefits of separation between application logic and storage logic. In the case of our application, we do not want to be bound to a single graph database provider (CosmosDB, Neptune, etc). To accomplish this we will provide our application with a graph database `interface`.

An interface tells a caller the _shape_ of the value that’s being passed to it. It describes what methods exist on that value and how to treat it overall (see [Interfaces](https://www.typescriptlang.org/docs/handbook/interfaces.html) ).

Our interface describes a graph database provider. We will also declare interfaces to represent the `Vertex` data type along with its sub-types.

```
export interface GraphAdapter {
    addVertex(type: string, input: any): Promise<Result<Vertex>>
}


export interface Vertex {
    id: string
    label: string
    properties?: VertexProperty[]
}
   
export interface VertexProperty {
    id: string
    label: string
    value: any
    properties?: Property[]
}

export interface Property {
    key: string
    value: any
}
```


</br>
In the last article we created a class called `GremlinAdapter` - [here](https://gist.github.com/DnOberon/bd02a94afffa9fc9a6af13a1b7645286#file-gremlin_adapter-ts). The `GremlinAdapter` class “satisfies” our new interface in that the class contains all the methods declared in the `GraphAdapter` interface along with their parameters and return values.

Now, any function that takes or returns a value of type `GraphAdapter` can accept or return a `GremlinAdapter` as well - without having to explicitly tell the application which graph database provider we’ve decided to use. As long as the application uses and refers to the `GraphAdapter` interface when interacting with the graph database we’ll have gained the ability to switch our graph database provider _without_ having to modify a large amount of code.

<br>
# Asset
Before we go any further, let’s create a quick class to represent the data we’ll be storing along with a simple constructor which accepts a name and an object of properties. We’ll also create a `factory` function, wrapping the record’s constructor.

```
export default class Asset {
    static readonly label = 'asset'
    id: string
    name: string

    constructor(id: string, properties: any) {

        this.id = id
        this.name = properties.name || "" 
    }

    public static factory(id: string, properties: any): Asset {
        return new Asset(id, properties)
    }
}
```

<br>
# Asset Storage

An easy way to create separation between data types and a storage layer is to follow the [Data Mapper Pattern](https://www.js-data.io/docs/data-mapper-pattern). We’ll create a basic implementation of this pattern by creating a class called `AssetStorage` and ensuring that any code related to how an `Asset` interacts with our graph database is kept here and not in the `Asset` class itself.

`AssetStorage` should contain three things. First, we need to hold an instance of our graph database adapter. We _could_ do something like this -

```
class AssetStorage {
    public db: GremlinAdapter
}
```


AssetStorage can now access `GremlinAdapters` methods internally, so we can do things like adding a vertex or an edge etc. However, we don’t want `AssetStorage` to have any knowledge of mechanics behind the graph database data storage (like what graph provider), only that it’s being stored in a graph database. We also want to keep the ability to quickly switch out graph database providers if needed.

Enter our `GraphAdapter` class.

```
export default class AssetStorage {
    public db: GraphAdapter

    public constructor(db: GraphAdapter) {
        if (db === null) throw new Error('you must provide a graph database adapter')
        this.db = db
    }
}
```

<br>
Now let’s add a method for adding a new Asset record into our graph database. 
```
export default class AssetStorage {
    public db: GraphAdapter

    public constructor(db: GraphAdapter) {
        if (db === null) throw new Error('you must provide a graph database adapter')
        this.db = db
    }

    Create(name: string): Promise<Asset> {
        return new Promise((resolve, reject) => {
            this.db.addVertex(Asset.label, { name })
                .then((result) => {
                    resolve(new Asset(result.id, properties))
                })
                .catch(e => reject(e));
        })
    }
}
```

Note how the function’s parameter isn’t an `Asset` class. Instead, we’re requiring that the caller give us all the information needed to create a new `Asset`. As long as our insertion of the record is successful, we return a concrete type - the new `Asset`.

<br>
Fantastic! We’ve successfully implemented a data mapper class and integrated it with our graph database interface and adapters. Now, with a little setup, we can start working with our Asset.

```
let g = new GremlinAdapter()
let assetStorage = new AssetStorage(g)

g.Create("Test Asset") // Promise<Asset>
```

<br>
# Storage

You can get a lot of mileage from this pattern by stopping here. You’ve created a solid barrier between application logic and storage logic - and then went one better by separating your storage logic from the means by which the data is stored.

There is a downside however. Implementing the `*Storage` class pattern with other record types means you must rewrite the `Create` function for each type. This isn’t the end of the world, and in other cases it might be something you have no choice but to live with. Because we’re working with graph databases - we can do something different.

In a graph database a Vertex record is always the same thing. It consists of an ID, label, and a set of properties. No matter what kind of data the vertex is holding, it must always conform to that structure. With this in mind, we can cut down the amount of boilerplate that would have to be written for basic CRUD functions for each record type.

Let’s create a Storage class - of which every record Storage class (e.g AssetStorage) will inherit from. We’ll also remove the constructor from `AssetStorage` as it now inherits from Storage.

```
export default class Storage {
    public db: GraphAdapter

    public constructor(db: GraphAdapter) {
        if (db === null) throw new Error('you must provide a graph database adapter')
        this.db = db
    }
}
```

We want to pull the bulk of the `Create` function from `AssetStorage` into the parent `Storage` class - but we also want to maintain the behavior of passing parameters in and getting a concrete type out.

Enter [Generics](https://www.typescriptlang.org/docs/handbook/generics.html) and [Factories](https://en.wikipedia.org/wiki/Factory_method_pattern).

<br>

# Generic Create & Record Factory

Before we go on, let’s revisit the `factory` method on our Asset class - basically a wrapper over the `constructor` function for an Asset.

```
 public static factory(id: string, properties: any): Asset {
        return new Asset(id, properties)
    }
```

<br>
In order to maintain the behavior of accepting parameters and returning a concrete type, the `Create` function we’re adding to the Storage class needs to include an additional parameter - a factory function. Now, once a vertex is created, we use the factory function to return a concrete type specified by the caller.

```
create<T>(factory: (id: string, properties: any) => T, label: string, properties: any): Promise<T> {
        return new Promise((resolve, reject) => {
            this.db.addVertex(label, properties)
                .then((result) => {
                    resolve(factory(result.id, result.properties))
                })
        })
    }
```

<br>
The power of this pattern becomes apparent when we modify the `AssetStorage` class to reflect the new functionality.

```
export default class AssetStorage extends Storage {
    public Create(name: string): Promise<Result<Asset>> {
        return super.create<Asset>(Asset.factory, Asset.label, { name })
    }
}
```

<br>

Abstraction should offer flexibility. I believe the patterns shown in this two part article set represent a powerful, flexible way of interacting with graph databases in TypeScript. I encourage you to experiment and expound on my findings - and reach out if you have any questions or suggestions.
