# Cache

![](https://github.com/sebelga/gstore-cache/raw/master/logo/logo.gif)gstore-node integrates fully with the [gstore-cache](https://github.com/sebelga/gstore-cache) manager. Well, let's say that gstore-cache has been created for gstore-node but it has been made available as a separate npm package for those who don't use gstore-node.

The advantage of using gstore-cache from within gstore-node is that all the management of the cache \(setting/clearing\) is done for you. Just initialize gstore with the cache turned on and call it a day :\)

gstore-cache uses underneath the powerful [node-cache-manager](https://github.com/BryanDonovan/node-cache-manager). This means that you can have multiple cache stores with different TTL settings for each one and that you get a [LRU memory cache](https://www.npmjs.com/package/lru-cache) instance for free.

**Important:** Since v2.7.0 of node-cache-manager it is possible to set/get and delete multiple keys at once from the cache stores. Although this feature is available, you still need to provide a store that support it. At the time of this writing, only the _memory_ store and the [node-cache-manager-redis-store](https://github.com/dabroek/node-cache-manager-redis-store) support it. If you provide [another store engine](https://github.com/BryanDonovan/node-cache-manager#store-engines) that does not support _mget_ or _mset_ you will still be able to use the cache but you won't be able to fetch **multiple** keys from the Datastore and cache them.

## Activate the cache

You activate the cache by passing a configuration object during the gstore initialization. You can also just pass **true** and the default cache configuration will be used.

#### Default configuration

```js
// server.js (Application Bootstrap)

require('gstore-node')({ cache: true }); 

// ------------------------------
// Default configuration
// ------------------------------

const default = {
    stores: [
        {
            store: 'memory',
            max: 100, // maximum number of item in the cache
        },
    ],
    ttl: {
        keys: 60 * 10, // 10 minutes
        queries: 5, // 5 seconds
        // the "stores" setting below is only used when there are multiple stores
        stores: {
            memory: {
                keys: 60 * 5, // 5 minutes
                queries: 5,
            },
            redis: {
                keys: 60 * 60 * 24, // 1 day
                queries: 0, // infinite
            },
        },
    },
    cachePrefix: {
        keys: 'gck:', // Gstore Cache Key
        queries: 'gcq:', // Gstore Cache Query
    },
    global: true,
};
```

Refer to the [gstore-cache](https://github.com/sebelga/gstore-cache) for  a detailed explanation of the configuration properties.

##### Extra config

`global (default: true)`

There is a special _gobal_ config only used by gstore-node. It will turn the cache on globally for all keys and queries fetching. If you set global to _false_, then you will have to explicitly set the cache option to true when calling any Model or Entity method.

#### Custom configuration

To change the configuration you only need to provide what you want to override.

```js
// server.js

const cacheConfig = {
    ttl: { keys: 1200 },
    queries: -1, // ===> don't cache queries (can be overriden later on a specific query call)
};

const gstore = require('gstore-node')({ cache: cacheConfig }); 

// --------------------------------------------------------------

// multi stores example

const redisStore = require('cache-manager-redis-store');

const cacheConfig = {
    stores: [
    {
        store: 'memory',
        max: 100,
        ttl: 15,
    },
    {
        store: redisStore,
        host: 'localhost', // default value
        port: 6379, // default value
        // ... any other configuration for Redis
    }],
    ttl: {
        stores: {
            memory: {
                keys: 10,
                queries: 5,
            },
            redis: {
                keys: 60 * 60,
                queries: 60 * 60
            },
        }
    }
};

const gstore = require('gstore-node')({ cache: cacheConfig }); 

// ...
```

## Access the cache instance

You can access at any time the underlying gstore-cache **instance** and call its API. If you need to cache custom data \(other the than _keys_ or _queries_ managed gstore-node\), just call the set/mset/get/mset/del methods directly on the cache instance.  
For more information on the API refer to the [gstore-cache documentation](https://github.com/sebelga/gstore-cache#api).

```js
// Anywhere in your Appication
const gstore = require('gstore-node')();

// The cache is a "gstore-cache" instance
const { cache } = gstore;

// You can then call any method from gstore-cache

cache.get('somekey').then(...);
cache.set('someKey', { name: 'john' }).then(...);
```

## Advanced Cache for Queries

gstore-cache has an advanced cache for queries **when you provide a **_**Redis**_** store** \(either as single store or in a multi-store\). gstore-cache detects the _Kind_ of the entities on which the query is run to be able to not only cache the data of the query but also a reference to it in a Redis _Set_. It creates one Redis Set by _Entity Kind_.

This means that you can have an infinite TTL \(0\) for the queries on the Redis store. Each time an entity is added/updated/deleted gstore-node will automatically remove all the queries references from the Entity Kind Set and clear the cache data from the queries.

### Useful methods

All the cache management of gstore-cache is done for you by gstore-node. There is though one useful method that you might need:

#### `gstore.cache.queries.kset(key, data, entityKinds)`

In case you have a complex query where the data comes from different entity Kinds, `kset()` lets you save its data in the cache and **link** the Entiy Kinds related to it. Later, if _any_ Entity Kind passed here is added/updated or deleted, gstore will remove the query from the cache.

`kset()` \(for **k**ind **s**et\) accepts 3 parameters:

* _key_ : a custom cache key you want to give to this data
* _data_: the data to cache
* _entityKinds_: one or multiple entityKinds related to the data

Let see it with an example:

```js
const gstore = require('gstore-node')();
const { cache } = gstore;

const Posts = require('./posts.model');
const Users = require('./users.model');

/**
 * Handler to fetch all the data for our Home Page
 */
const fetchHomeData = () => {
    // Check the cache first...
    cache.get('website:home').then(data => {
        if (data) {
            // in the cache... great!
            return data;
        }

        // Cache not found, query the data
        const queryPosts = Posts.query()
            .filter('category', 'tech')
            .limit(10)
            .order('publishedOn', { descending: true });

        const queryTopStories = Posts.query()
            .order('score', { descending: true })
            .limit(3);

        const queryProducts = Products.query().filter('featured', true);

        return Promise.all([queryPosts.run(), queryTopStories.run(), queryProducts.run()])
            .then(result => {
                // Build our data object
                const homeData = {
                    posts: result[0],
                    topStories: result[1],
                    products: result[2],
                };

                // We save the result of the 3 queries to the cache ("website:home" key)
                // and link the data to the "Posts" & "Products" Entity Kinds.
                // We can now safely keep the cache infinitely, gstore will take care
                // of clearing the cache when a "Posts" or "Products" is added/edited or deleted.
                return cache.queries.kset('website:home', homeData, ['Posts', 'Products']);
        });
    });
};
```

## Transactions

For the most part you don't have to worry about clearing the cache as gstore-node automatically does it for you each time you add/edit or delete an entity.  
It cannot do it for you though when you are updating entities inside a transaction as it does not know if the transaction succeeded or not. So you will have to clear the cache manually after a transaction succeed.

Let see it with an example.

```js
const gstore = require('gstore-node')();
const transaction = gstore.transaction();

const User = require('./user.model');
const Post = require('../posts/post.model');

transaction.run()
    .then(() => {
        return User.get(123, null, null, transaction)
            .then((user) => {
                const post = new Post({ title: 'My new Blog Post', authorId: user.id });
                post.save(transaction);

                // We update the total of posts for this user
                user.totalPosts += 1;
                user.save(transaction);

                transaction.commit()
                    .then(() => {
                        // Transaction successful... we need to clear the cache now

                        // 1. Delete the cache for "User" Entity Kind
                        // ---> both the key passed **and** the queries linked to "User" will be deleted
                        const promise1 = User.clearCache(User.key(123));

                        // 2. Delete the cache for "Post". Here no key is passed, we just want
                        // to make sure that all query data cached linked to "Post" is deleted
                        const promise2 = Post.clearCache();

                        return Promise.all([promise1, promise2]);
                    });
            });

    })
```



