#Queries

## @google-cloud Query

gstore is built **on top of @google-cloud/datastore** so you can execute [any query from the Google library](https://googlecloudplatform.github.io/google-cloud-node/#/docs/datastore/master/datastore/query).  

1. Create a query object `const query = MyModel.query(namespace /*optional*/, transaction /*optional*/)`
`
2. and chain the operators to build the query. `query.filter(...).order(...).start(...)`
3. call query.run() to execute the query. `query.run([options]).then( ... )`


### Create the query

```js
const query = MyModel.query(
    /* {string}. -- optional. A namespace to execute the query */
    <namespace>,
    /* {object} -- optional. A transaction to execute the query from */
    <transaction>
);
```

### Chain query operators

Refer to [@google-cloud/datastore](https://googlecloudplatform.github.io/google-cloud-node/#/docs/datastore/master/datastore/query) for list of operators

```js
query.filter(...).order(...).groupBy(...).start(...);
```  

### Run the query
To execute the query call `query.run(options)`

**@Returns** -- the response contains both the entities and the cursor if there are more results

```js
query.run(
    /* {boolean}. -- optional. Default: false
       If set to true will return all the properties of the entity, regardless of the *read* parameter defined in the Schema */
    <readAll>,
    /* {string} -- optional. "JSON"|"ENTITY". Default: "JSON"
       Format of the response, either plain object or entity instances  */
    <format>
);
```

Example:
```js
const User = require('./user.model');

// 1. Build the query
const query = User.query()
            .filter('firstname', '=', 'John')
            .filter('age', '>=', 4)
            .order('lastname', { descending: true })
            .limit(10);

// 2. Execute the query.

// with Promise
query.run().then((response) => {
    const entities = response[0].entities;
    const nextPageCursor = response[0].nextPageCursor; // not present if no more results
});

// or with a callback
query.run(function(err, response) {
    if (err) {
        // deal with err
    }

    const entities = response.entities;
    const nextPageCursor = response.nextPageCursor;
});

// You can then use the "nextPageCursor" when calling the same query and pass it as start value
const query = User.query()
                  .filter('name', '=', 'John')
                  .filter('age', '>=', 4)
                  .order('lastname', { descending: true })
                  .start(nextPageCursor);
                  
// Query on namespace
const namespace = 'com.dev.server';
const query = User.query(namespace)
                .filter('name', '=', 'John');

// Query in a transaction
const transaction = gstore.transaction();
transaction.run().then(() => {
    // Create the query inside the transaction
    const query = User.query(null, transaction)
                      .filter('name', '=', 'John');
   
    query.run().then(() => {
        // other operations inside the transaction
        ...
  		
        transaction.commit().then( ... )
    });	
});

// run options
query.run({ readAll: true, format: 'ENTITY' })
     .then( ... )
```