# Fauna Schema Migrate (Unofficial)

## Introduction

####  Goals

The Fauna Schema Migrate tool is an opinionated tool with two goals:

- Set up Fauna resources as code.
- Provide support for fault-proof schema migrations for bigger teams. 
- Provide guidance on how to manage fauna resources by showing the Let statements that can be generated to transactionally update databases.

####  What is a 'schema' migration? 

Schema is a confusing name but typically what we search for for this sort of thing. The Fauna resources such as Functions, Collections, Indexes, Roles and AccessProviders are considered schema. Although **data migrations** can be done as well, this is a very different problem, see the section on data migration for more information. 

#### Setup

Install the tool in a repository that has the faunadb javascript driver installed. The javascript driver is a peer dependency and will therefore not be pulled in by the tool. It makes more sense to rely on the same fauna driver as the repository to allow users to be in control of that version. T

```
npm install fauna-schema-migrate
```

Run interactively with npx:

``` 
npx fauna-schema-migrate run
```

Or with:

```
node_modules/.bin/fauna-schema-migrate
```

The **minimum** driver version should be: 

```
"faunadb": "^4.0.3"
```

This driver contains some minor fixes to the FQL formatting which are required for the tool to work properly. 

#### Available commands

Run `fauna-schema-migrate` to see all commands and extra options.

```shell
Commands:
  run                             Run interactively
  init                            Initializing folders and config
  state                           Get the current state of cloud and local migrations
  generate                        Generate migration from your resources
  rollback [amount] [childDb...]  Rollback applied migrations in the database
  apply [amount] [childDb...]     Apply unapplied migrations against the database
```

Or run `fauna-schema-migrate run` to test it out interactively. All commands can be run interactively as well by using run. The command parameters from rollback and apply are not (yet) accessible interactively though. 

#### Flow

1. **Initialize** **standard folders** with `fauna-schema-migrate init`  or by using the interactive `run` command.

   Init will generate a *fauna* folder and a *.fauna-migrate.js* config file. If you don't want to change anything to the default folders  or default collection to store migration state, you can remove the .fauna-migrate.js file. 

   The default folder structure which will be generated by init looks as follows:

   ```
   fauna > resources
         > migrations
   ```

2. **Write resources** 

   Create your resources by adding .fql or .js files with a default export to the fauna > resources folder. Any file, even in sub-folders will be picked up by the tool so you can organize your files as you wish. For example you could organize files as follows:

   ```
   fauna > resources > users_collection.fql
   				  > users_by_name_index.fql
   				  > logged_in_users_role.fql
   ```

   Or use folders and order by resource type or domain: 

   ```
   fauna > resources > collections > users.fql
   				  > indexes     > users_by_name.fql
   				  > roles       > logged_in_users.fql
   ```

   It doesn't matter,  there is only one caveat and that is to avoid the foldername 'dbs' unless otherwised configured since that name is reserved to support child database migrations. Resource files always have to contain one (and only one) of the following functions.

   ```js
   CreateCollection(...)
   CreateIndex(...)
   CreateFunction(...)
   CreateRole(...)
   CreateAccessProvider(...)
   ```

   For example, the following is a valid **.fql** file:

   ```js
   CreateIndex({
     name: 'users_by_alias',
     source: Collection('users'),
     terms: [
       {
         field: ['data', 'alias']
       }
     ],
     unique: true,
     serialized: true
   })
   ```

   When using **.js** files, you can import/write anything you want as long as the default export is one of the above. Below is an example from Fwitter that creates a function based on an external file: 

   ```js
   import faunadb from 'faunadb'
   
   import { CreateFweet } from '../fauna/src/fweets'
   const q = faunadb.query
   const { CreateFunction, Query, Var, Lambda, Role } = q
   
   export default CreateFunction({
     name: 'create_fweet',
     body: Query(Lambda(['message', 'hashtags', 'asset'], CreateFweet(Var('message'), Var('hashtags'), Var('asset')))),
     role: Role('functionrole_manipulate_fweets')
   })
   ```

3. **Generate migrations** with `fauna-schema-migrate generate` or via the interactive tool started by `run` 

   Generate will look at the resources folders and the migrations folder and determine what has changed in the resources folder since the last time we ran generate. Generate will generate the FQL for the migration in .fql files. The tool will verify whether all resources that are referenced are present in the resources. For example, you can't currently create a role that secures a collection which is not present in your resources folder. 

   *Why this intermediate step?* This is useful since migrations can also contain JavaScript which might be imported from other files to maximize reuse and FQL composition. We would risk to change old migrations if we change external JavaScript files. Generate essentially sets the migration in stone by taking the FQL that the JS file generated and turning it into a pure standalone .fql file. Changing externally imported files will no longer impact the migration (but when running generate, will trigger a new migration if the resulting query has changed)

4. **Verify the migration** 

   he tool is still in beta and relies heavily on comparison of FQL json formats. It has been tested in several advanced scenarios. When you start using it early on. Verify whether the migration results are correct in the migrations folder. The migration folder will contain different migrations folders with a timestamp as name. Each generated migration contains an .fql file for each resource that was created/updated/deleted. Below is an example structure: 

   ```
   fauna > migrations > 2021-01-25T20:49:16.074Z > create-collection-accounts.fql
   											  > create-collection-comments.fql
   											  > create-index-accounts-by-name.fql
   				   > 2021-02-09T15:49:58.115Z > create-collection-fweets.fql
   				                              > create-index-all_fweets.fql
   ```

   Do not edit migrations by hand except if you are never going to use **generate**. If there is a desire for writing migrations by hand, let us know, we can add another modus for that later on. Migrations can contain the following top-level statements.

   ```
   CreateCollection(...)
   CreateIndex(...)
   CreateFunction(...)
   CreateRole(...)
   CreateAccessProvider(...)
   Update(...)
   Delete(...)
   ```

5. **Apply or rollback as desired.** 

   Once there are migrations, use apply and rollback to apply or roll back your migrations one by one on the database.

   `fauna-schema-migrate apply`

   `fauna-schema-migrate rollback`

   In contrast to existing tools to aid managing fauna resources, these functions will combine all to be executed resource migrations in one transaction by using a Let statement. Either the full transaction will fail or complete, nothing will be applied partially. The migration query will be printed for educational purposes, if the print becomes big you can opt to remove it by passing -n or --no-print .

   Both rollback as apply can be run from the interactive tool as well and both take two options (which are not available in the interactive tool). The first option is the amount of migrations to apply/rollback, the second option is to support child databases as explained further. Applying multiple migrations at once will combine these multiple migration steps in **one** transactions intelligently. This is an optimization intended to speed up setting a database from scratch in development scenarios. 

6. **Optionally** **verify the state.** 

    `fauna-schema-migrate state`

## Extras

##### ENV variables:

- FAUNA_ADMIN_KEY: provide the admin key
- FAUNA_CHILD_DB: set a child database (see next section) for development purposes
- FAUNA_LEGACY: remove fancy output for legacy terminals that might not support it or if you just don't like it or want to copy the generated FQL query without the box.
- FAUNA_NOPRINT: don't print the FQL migration.

##### Faster local development with a child database

One of the parameters, FAUNA_CHILD_DB is useful in development in case you often find yourself completely nuking the database and reapply everything from scratch. In that case, you might bump into the cache which requires you to wait for 60 seconds before recreating the resources. FAUNA_CHILD_DB will create your resources in a child database instead of in the database that the admin key points at. When using FAUNA_CHILD_DB and rolling back before the first transactions (e.g. with `fauna-schema-migrate rollback all`) the rollback will nuke the database instead of applying the rollback which essentially avoids the 60 seconds cache. 

##### Managing multiple databases

Support for child databases is provided by adding the folder **dbs** in the resources directory. This allows to define resources in child databases (recursively). A potential structure could be: 

```
fauna > resources > ... resources for the root db ...
                  > dbs > nameOfchildDb1 > ... resources ...
                        > nameOfChildDb2 > ...
                        > dbs > nameOfChild1OfChildDb2 > ...
```

The 'dbs' name can be configured in the config. When running `fauna-schema-migrate generate` the tool will generate migrations to create child databases as well as generate migrations for all resources within the child databases. The above would result in a migration on the root database to create the two child databases and a dbs folder within the migrations folder that will contain migrations for all child database etc. Migrations of child databases are applied completely separate from the root database.  We can currently not update multiple databases transactionally and some generate steps might not have a new migration for a specific database. Therefore, it might not make sense to have an apply command over multiple databases. 

When running `fauna-schema-migrate apply` or `fauna-schema-migrate rollback` only the database migrations of one database will be applied depending on the parameters, by default that will be the root database which the admin key points at. Therefore, running `fauna-schema-migrate apply` without parameters will run the migrations of the root database. 

Running `fauna-schema-migrate apply 1 nameOfChildDb1` will apply a migration on the first child database. Running `fauna-schema-migrate apply 1 nameOfChildDb2 namdOfChild1OfChildDb2` would apply a migration on the child of the second child databases. This functionality is **not (yet)** available in the interactive version.

## Where to go from now?

### Potential extensions

- **GraphQL schemas,** this could easily be added. The complicated part is that we would have to either make sure we know what resources the GraphQL schema already created in order to avoid conflicts or transform Create statements to CreateOrUpdate statements in case there is a GraphQL schema in the resources folder.  

- **Copy schema from an existing database**: users often start by testing fauna in the dashboard, it would make sense to take an existing database and download the schema as resources to continue building upon. Currently this was out of scope.

- **Expose functions to use it as a library**: for integration testing purposes it might make sense to be able to run part of the functions programmatically instead of having to call it as a commandline tool The [https://github.com/fauna-brecht/fauna-schema-migrate/blob/master/tests/_helpers.ts](https://github.com/fauna-brecht/fauna-schema-migrate/blob/master/tests/_helpers.ts) itself are a good example. You could easily provide a resource folder and use the functions to set up an integration test against Fauna such as in the snippet below. 

  ```
  sinon.stub(Config.prototype, 'getResourcesDir')
              .returns(Promise.resolve(path.join(dir, folder)))
  const planned = await planMigrations()
  const migrations = await generateMigrations(planned)
  const time = new Date().toISOString()
  await writeMigrations([], migrations, time)
  const res = await apply(1)
  ```

  which takes a custom resource folder (set by stubbing the config) and applies whatever is in that folder for this test. I assume many people would like to set up tests with Fauna UDFs/Roles where they could benefit from the library if it would expose this functions in a simplified format (generate/apply/rollback/...) and therefore also behave as a library next to being a commandline tool. 

- **Plan step**: it makes perfect sense to save the FQL that will be applied to a plan (similar to terraform). That way we could verify the plan and run the plan both locally, on staging as in production with less risk that there could be a difference. 

- **Migration only mode:** do you only use .fql files and don't care about the separate resources folder to nicely organize resources then you might want to write migrations directly. We could offer a modus to do so to be defined in the configuration. 

- **No migration mode:** or would you prefer operating such a tool without migrations in a similar fashion as terraform, define resources and just update cloud.. whatever has to be done. This could also be a modus that we provide in the configuration.  

- **Validate schema against cloud**: we could in theory validate whether migrations are valid depending on what is present in cloud to avoid detecting problems at the last possible moment when users have changed their database schema manually. E.g. we could detect when a role is updated that refers to a deleted collection. Currently the transaction would abort and throw an error which is probably good enough. It might still be useful to verify whether someone has manually tampered with the cloud  schema and see the difference and/or fix it if there is a difference. 

Please let us know your requirements so we can decide what would be a good next step. 

### Unofficial

The tool is currently unofficial since it's essentially developed by a Developer Advocate who felt the need to facilitate his own work flow as well as help users along the way. As long as the tool is hosted under my personal user (fauna-brecht) it will remain unofficial. Once the tool seems to fulfill the need there is for managing fauna resources and we have gather some feedback we will look into more official support/maintenance. 



## Philosophy

These are some extensive notes on why the tool is designed as it is. 

#### Keep FQL composable

The FaunaDB Query Language (FQL) is very different than other query languages like SQL, Cypher or ORM approaches. One of the features that are worth embracing in FQL is that writing FQL queries is essentially function composition. For example:

```javascript
Collection("accounts"), "268431417906561542")
```

is just a reference to a collection. Throw that in another function to get a reference to a document within that collection.

```javascript
Ref(
  Collection("accounts"),
  "268431417906561542"
)
```

That beautiful part is that these are just functions that define your query and we can use the host language (in this case JavaScript) to start saving snippets of queries for reuse. A very silly example would be: 

```javascript
const accountCollection = Collection("accounts")
const account = Ref(accountCollection,"268431417906561542")
...
```

But you can imagine when queries implement complex transactional logic or start joining extensively that this becomes very useful. It's something we take advantage of in many examples such as [Fwitter](https://css-tricks.com/rethinking-twitter-as-a-serverless-app/) or [the authentication skeletons](https://github.com/fauna-brecht/skeleton-auth) to define return formats only once or write reusable snippets that can be easily mixed into other queries. For example,  a query that creates a new user but also implements rate limitation and adds validation could be abstracted away in JavaScript as follows:

```javascript
function CreateUser(email, password) {
	let FQLStatement = <query to create a user> 
	FQLStatement = ValidateEmail(FQLStatement, email, ['data', 'email'])
	FQLStatement = ValidatePassword(FQLStatement, password, ['data', 'password'])
	return AddRateLimiting('register', FQLStatement, email)
}
```

What ValidateEmail, ValidatePassword or AddRateLimiting do is not important, but it's important that this composability might be used and we want to make sure it doesn't conflict with our migrations. If we directly use the widely used approach of many migration frameworks of writing up/down statements as popularized by Rails (as far as I know) .  For example: 

```javascript
module.exports.up = () => {
  return CreateCollection({ name: 'Users' })
}

module.exports.down = () => {
  return Delete(Collection('Users'))
}
```

This might be a bad idea in case the up definition would have looked like: 


```javascript
import { CreateOrUpdateFunction } from './../helpers/fql'

module.exports.up = () => {
  CreateFunction(
    'register',
    Query(Lambda(['email', 'password'], 
      RegisterAccountWithEmailVerification(Var('email'), Var('password'))))
  )
}
...
```

Your migration is no longer **fixed** as this code that was important can evolve separately from the migration and that's probably hard to manage. This is something that's quite unlikely for the creation of indexes and collections but becomes a real problem for User Defined Functions that want to share some logic. 

### Do less work and reduce the possibility of errors. 

The idea of this schema framework library is not only to  help support stable schema migrations. First and foremost it's intended to help express FaunaDB resources as code. In essence support the resources equivalent of Infrastructure as Code (IaC)  but you can't really call this 'infrastructure' since FaunaDB handles all infrastructure for you. 

If we define a function, index or collection as follows

```javascript
module.exports.up = () => {
  CreateFunction(name, body)
}
```

The down function can easily be derived so it would be a bit silly to ask the user to write this. Especially since down migrations are often not tested thoroughly so it's easy to get them wrong.  

Instead we chose to let the user define the Create statement and will derive the down statement automatically when rolling back or even derive Update statements when we noticed that resources has changed since the last migration. 

### Detect dependencies.

Functions can depend on roles or can depend on collections and typically you would define these in one migration. This requires you sometimes to: 

- Define the Role
- The define the function that uses the role
- Define the role that can call the function.

Instead of requiring the user to think of this order we'll detect it and order the different resources in **one migration** automatically.

### A nice overview of resources

When writing fauna examples myself, I like my resources to be nicely presented making easy to find resources by type or by domain. Up/down migrations don't deliver that and scatters your resources in an unorganized series of files. Having a resource folder with simple names without timestamps which I can organize how I want, in either FQL or JS files brings more clarity to a repository. 

### A format to exchange Fauna functionality? 

If the approach is embraced by our community, we could have an easy uniform format to share schema files in repositories. I can now package a few fauna repositories in a GitHub repository that tackle specific functionality. For example, 

- Authentication skeletons which set up the functions, roles, indexes, collections necessary for basic or advanced authentication scenarios as a starting point. Leaving the implementation of a frontend/backend implementation up to the user (of course, we will continue delivering accompanying applications)
- A repository with a User Defined Function to apply rate limiting on other functions with the indexes and collections it requires. 
- Packing up user contributed functionality in a repository such as: https://forums.fauna.com/t/logging-function/1513/3.

## Notes on data migration

This library is **not** intended for data migration. Thats essentially since pagination kicks in when one tries to write huge data migrations which means that one huge data migration can't be performed in one transaction. Such huge data migrations rarely have a rollback script specified. Instead developers typically take a backup before they do it, test it extensively on real-life data or ideally.. avoid the need for big data migrations completely. If necessary, it's often a sign that the migration strategy of an application should  be revised since a huge migration transaction that runs for several hours is also a big risk. 

##### Alternative strategies:

Instead you could start the migration at the moment you deploy the new code, keep the timestamp when the migration started and after completing the batches, verify which new data that arrives that was still inserted with the old code and should be migrated and go through the pages again. Until we arrive at a point that the 'new data' is an empty page in which case we successfully migrated all data. 

This clearly requires your code to be able to deal with both the old format as the new format of the data in given certain timespan which is something that's advisable regardless of the database. There are moments where this is simply not possible due to a difficult implementation process that involves many services. In that case there are two things you can do:

- Temporary change your queries to work on a snapshot using FaunaDB's temporality and pause/block new creates/writes (allow reads, block writes)
- Temporary take down the application until the migration is complete
  (block both reads and writes)

##### Need it anyway?

Maybe there is a need to have small seed data in your database or there might be value in looking into a separate npm library to help with such complex data migrations. Let us know what you need, your feedback is important to us. 

## Inspiration

This library was inspired by excellent database libraries and efforts of community users such as 

- https://github.com/Plazide/fauna-gql-upload

- https://github.com/ptpaterson/faunadb-graphql-schema-loader
