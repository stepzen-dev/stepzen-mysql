# Overview 

#### This project builds a GraphQL API connected to your MySQL database using StepZen. We will use the @dbquery directive to generate our GraphQL schema. 

# Project Setup Commands To Run First

`git clone PROJECT URL`

`cd PROJECT NAME`

#### First things first, you'll need to deploy a database on Heroku. 

Download MySQL workbench [before you start](https://dev.mysql.com/downloads/). You should also have MySQL installed on your machine in order to use the workbench.

Sign up for [a Heroku account](https://signup.heroku.com/). It can be on the free tier, but you'll need to add a credit card to your account to use the add-ons that you need (it won't be charged as a part of this tutorial). Once you've done that, you can proceed.

Go to your dashboard and click 'New', then 'Create New App'. Type in your app name and hit 'Create App'. 

Click on the 'Resources' tab and search for 'ClearDB MySQL' under 'Add-ons'. Click on it and select the 'Ignite-Free' plan and then 'Submit Order Form'. 

Now click 'Settings' and 'Reveal Config Vars'. For the value of `CLEARDB_DATABASE_URL` you should see a string formatted like `mysql://USERNAME:PASSWORD@HOSTNAME/DB_NAME`. You'll use this info for setting up a new connection in your MySQL Workbench in the next few steps.

#### Seeding your database from MySQL Workbench

Open up MySQL workbench. 
Click the home icon at the top left corner.
Next to 'MySQL Connections' click the + button,
then name your connection (whatever you want) and enter the USERNAME and HOSTNAME from your heroku account. You can leave the port as it is.  
Click 'Store in Keychain'and add your PASSWORD.
Click 'Test Connection'.  You won't need a default schema. 
Once you hit 'OK' on the notification that your connection was successful, hit 'OK' again. 
Click on your connection to open it. 

Click the 'schemas' tab on the top left. Double click the name of your schema on the left to make sure it is selected, and copy and paste this code:

```
CREATE TABLE `authors` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(45) NOT NULL,
  `isPseudonym` tinyint(4) DEFAULT '0',
  PRIMARY KEY (`ID`),
  UNIQUE KEY `ID_UNIQUE` (`ID`)
);

CREATE TABLE `books` (
  `id` int(11) NOT NULL,
  `name` varchar(45) NOT NULL,
  `originalPublishingDate` date NOT NULL,
  `authorID` int(11) DEFAULT NULL,
  PRIMARY KEY (`ID`),
  UNIQUE KEY `ID_UNIQUE` (`ID`)
);

INSERT INTO `authors` 
VALUES 
(1,'Agatha Christie',0),
(2,'Sherlock Holmes',0),
(3,'Edgar Allan Poe',0);

INSERT INTO `books` 
VALUES 
(1,'Murder on the Orient Express','1934-01-01', 1),
(2,'The Mysterious Affair at Styles','1920-10-01', 1),
(3,'The Adventure of the Three Garridebs','1924-10-25', 2),
(4,'The Murders in the Rue Morge','1841-04-04', 3);
```

Make sure it is either 1) all selected or 2) not selected at all and click the little lightning bolt under the Query 1 tab. This will run the SQL code that seeds your database. 


#### Set yourself up with StepZen

_Make sure you're in the folder you cd'd into at the start._ 

`npm install -g stepzen` 

`stepzen login -a [account name]`

...and enter your admin key. 

#### Create .gitignore file
```
touch .gitignore
```

You'll use this to protect sensitive information later. 

#### Let's get started on connecting MySQL

In the same folder as this project, run:

`touch config.yaml`

Then add `config.yaml` to your .gitignore as well. It's private info! :key:

This is what you'll place in it: 

```
configurationset:
  - configuration:
      name: MySQL_config
      dsn: USERNAME:PASSWORD@HOSTNAME/DB_NAME
```
What does this code do? It provides StepZen the information it needs to connect to your database.

Now you just need to upload and deploy to StepZen.

#### Deploying to StepZen

`stepzen upload configurationset libraryapi/config --file=./config.yaml`
^This uploads your config. 

```
stepzen upload schema libraryapi/schema --dir=. &&
stepzen deploy libraryapi/api --schema=libraryapi/library --configurationsets=libraryapi/config
```
^this uploads your schema and deploys to StepZen

#### The StepZen GraphiQL Query Editor

Now you can run `stepzen start libraryapi/api` in your terminal. This will open up a GraphiQL query editor connected to your database! 

Paste this code into your the query editor:

```
{
  booksFromBackend(originalPublishingDate:"1934-01-01")
  
     {  
			id
				}

}
```

And you should get a result like: 
//SCREENSHOT HERE
So how does the magic happen? 


#### GraphQL Files 


Let's get under the hood, starting with book.graphql, which is written in GraphQL Schema Definiton Language.


```
interface Book {
  id: ID!
  name: String!
  originalPublishingDate: Date!
  authorID: ID!
  author: Author
  @materializer(
        query: "author"
        arguments: [{ name: "id" field: "authorID"}]
  )
}
type BookBackend implements Book {
            id: ID!
            name: String!
            originalPublishingDate: Date!
}
type Query {
  book(id: ID!): Book
  books(originalPublishingDate: Date!): [Book]
  bookFromBackend(id: ID!): BookBackend
        @supplies(query: "book")
        @dbquery(type: "mysql", table: "books", configuration: "MySQL_config")
  booksFromBackend(originalPublishingDate: Date!): [BookBackend]
        @supplies(query: "books")
        @dbquery(
            type: "mysql",
            query: "SELECT * FROM books WHERE ? >= DATE '1900-00-00'",
            configuration: "MySQL_config"
        )
}
```

The first eleven lines of code define the interface, which allows us to implement our Book type multiple times. 

Notice the `@materializer` directive. When we query `book`, we want to be able to see information on the book's author as well, and we're able to do this by using the materializer to query author and return the name. It will use the authorID field to find each author. 

Next, to connect the backend, we implement the interface as a concrete type with `type BookBackend implements Book...`. 

Lastly, we implement our query. 

### What's in the query?

The `@supplies` directive tells StepZen what query we're concerned with fulfilling.

The `@dbquery` directive holds a lot here. It tells StepZen that we're using a database query and that we're using MySQL.

In the first `@dbquery` directive, the table fields are the same as the fields in our type, so there's no need for a custom query. 

In the second directive, you'll notice the line ` query: "SELECT * FROM books WHERE ? >= DATE '1900-00-00'",` which is a custom query that returns only books written in the 1900's and beyond. This is how you can make custom queries to your database. 


#### Author Interface and Types
In author.graphql, we have a similar pattern, without a materializer. 

```
interface Author {
  id: ID!
  name: String!
  isPseudonym: Boolean!
}
type AuthorBackend implements Author {}
type Query {
  author(id: ID!): Author
  authors(isPseudonym: Boolean!): [Author]
  authorFromBackend(id: ID!): AuthorBackend
        @supplies(query: "author")
        @dbquery(type: "mysql", table: "authors", configuration: "MySQL_config")
  authorsFromBackend(isPseudonym: Boolean!): [AuthorBackend]
        @supplies(query: "authors")
        @dbquery(
            type: "mysql",
            query: "SELECT * FROM authors WHERE isPseudonym = ?",
            configuration: "MySQL_config"
        )
}
```

But how do we tell StepZen that we want to tie these types together into one query? 

#### Creating the Schema

That's where `index.graphql` comes in:

```
schema @sdl(files: ["author.graphql", "book.graphql"]) {
  query: Query
}
```

This is how StepZen knows to assemble the listed files to create the GraphQL API. 

And that's the last piece in the puzzle! :fireworks:

Please enjoy playing around with this project and adding your own twist on our MySQL integration. 

If you have questions about MySQL and StepZen, feel free to [consult the docs](https://my.stepzen.com/docs/build-a-graphql-endpoint/dbquery). 
