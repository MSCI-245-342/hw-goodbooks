# hw-schema-and-models

Author: Mark Smucker  
Date: July 2020

## Introduction

In MSCI 245, we started off with learning about relational databases.  We learned about the ideas of relations and how to interact with RDBMs using SQL.  We then progressed to learning about database design with entity-relationship diagrams and normalization.

Rails has extensive support for working with relational databases.  Rails and other web frameworks tend to be object oriented, but our databases are not.  The databases are relational.  Thus, there is a mismatch between the way Rails wants to work with data and the way the database wants to work with data.

Traditional database applications would use an API such as ODBC or JDBC to connect to the database, run queries, and get back the results of queries.  Wanting to make the programmers' lives better, Rails and other web frameworks tend to use an [Object Relational Mapping (ORM)](https://en.wikipedia.org/wiki/Object-relational_mapping).  These mappings are a compromise to connect two different ways of managing data.  

To build web apps, you need to learn the ORM of Rails, which largely means learning about:

1. How to create and manage your database schema using migrations.  

1. How to create models that map to the database.

### Reference Materials

This homework is not a replacement for written documentation and tutorials.  Besides the course textbook, you will find the following helpful:

1. Guide: [Active Record Basics](https://edgeguides.rubyonrails.org/active_record_basics.html)

1. Guide: [Active Record Migrations](https://edgeguides.rubyonrails.org/active_record_migrations.html), References: [Schema Definitions](https://api.rubyonrails.org/v6.0.3.2/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html) and  [table-definition (inside the do block of create_table)](https://api.rubyonrails.org/v6.0.3.2/classes/ActiveRecord/ConnectionAdapters/TableDefinition.html)

1. Guide: [Active Record Validations](https://edgeguides.rubyonrails.org/active_record_validations.html), Reference: [Validations, Helper Methods](https://api.rubyonrails.org/classes/ActiveModel/Validations/HelperMethods.html)

1. Guide: [Active Record Associations](https://edgeguides.rubyonrails.org/association_basics.html), Reference: [ActiveRecord::Associations::ClassMethods](https://api.rubyonrails.org/classes/ActiveRecord/Associations/ClassMethods.html)

1. Guide: [Active Record Query Interface](https://edgeguides.rubyonrails.org/active_record_querying.html)

You are also likely to find helpful the full books about Rails that you can read in the O'Reilly site that all UW students have access to (follow instructions in Learn that we gave for Head First SQL):

1. [Learning Rails 5](https://learning.oreilly.com/library/view/learning-rails-5/9781491926185/)

1. [Ruby on Rails Tutorial, 6th Ed](https://learning.oreilly.com/library/view/ruby-on-rails/9780136702726/)

1. [The Rails 5 Way, Fourth Ed.](https://learning.oreilly.com/library/view/the-rails-5/9780134657691/)

1. [Agile Web Development with Rails 6](https://learning.oreilly.com/library/view/agile-web-development/9781680507522/)


## Models and Migrations

**First things first: Your models are the most important thing in your web app.**  All of your "business logic" or what we call the brains of our apps should live in the objects that we call models.  The controller should only be a traffic cop, i.e. directing requests from the user to the models and collecting the output to hand to the views for display.  The views should have as little logic in them as possible and should largely just be able ouputting the data handed to them from the controller, which is coming from the models.  

Let's use a restaurant analogy: If the database holds our ingredients (data), the models are the chefs in the kitchen making all the food.  The executive chef acts like the controller.  The executive chef takes the order from the waitperson and directs the chefs to produce a table's dishes such that they are all ready at the same time.  The waitstaff are sort of like the views: they take orders (HTML forms and links) and deliver the food to the table.  The waitstaff is responsible for delivering the order to the kitchen, but the waitstaff shouldn't be touching the food on delivery except to put the right plates on the table.  Likewise, the executive chef (controller) delegates all the cooking to the kitchen staff.

So, the next time you are writing a view, remember: you shouldn't be cooking any food; your job is to deliver the meals nice and neatly.  Your models should have done the work for you.  Likewise, when you are writing the controller, your job is just to get the models to make the right dishes for you to give to the views.

**Summary:** Push implementation down into your models.  You can always write more model code.  Just like the WordGame model, not all models are connected to a database table.  You might have models that compute the shortest path for a delivery truck, but this computation is not tied to a database table itself (it will likely read data from another model that does represent data in the database).  If you stuff business logic into views and controllers, you defeat the advantages of the MVC pattern.

### Migrations

When we were working with SQL, we would write SQL DDL to create tables, add foreign key constraints, etc.  We could do that with Rails, too, but it is far better to adopt Rails' tools for creating and maintain our database so that we create it in a way that meshes well with ActiveRecord models.  

The concept of migrations is simple: write a script to change your database schema such that we can rollback the schema changes if needed.  By putting all of our schema changes into these scripts, we can create exactly the same database on each developer's machine, and we can make sure our production server is the same, too.  (Every developer uses their own database.  We do this to avoid messing up each other's debugging.  We absolutely never develop using the production database, which is running the web app live for customers.  In 245/342, our production server is running in the cloud at Heroku.)

We use rails to generate a migration for us that we can then edit as needed.  Each migration is given a timestamp as part of its file name.  When we run migrations via `rails db:migrate`, the migrations are run in order of their timestamps.  Rails also know which migrations have been run and which haven't.  Rails will only run new migrations that have not yet been run.

Rails can generate migrations as part of generating a model, as well as generate migrations on their own, i.e. no model creation.

When we are creating the database, we need to think about the order in which to create the tables.  For example, if a table has a foreign key, we cannot make that table until we've the foreign table.  

#### Example - GoodBooks

To illustrate models and migrations, we'll work with the following example schema:

books ( id (PK), title, year, author_id (FK) )  
authors ( id (PK), name )  
users ( id (PK), name, email )  
readinglists ( id (PK), name, user_id (FK) )  
ratings ( user_id (PK, FK), book_id (PK, FK), rating )  
books_readinglists ( book_id (PK, FK), readinglist_id (PK, FK) )

This is a schema for a book site name GoodBooks.  The schema is deficient because each book has one and only one author, but we're keeping things simple here.

The relationships between the entities are:

+ authors have a one to many relationship with books, and books have a one to one relationship with authors.  In other words:  
  + each book has one author (total participation, all books must always have an author)  
  + each author has zero or more books  (we can add and keep authors without a book entry)
+ users have a one to many relationship with readinglists, and readinglists have a one to one relationship with users.  In other words:  
  + each user has zero or more readinglists
  + each readinglist has one user (total participation, all readinglists must always have a user)
+ users and books have a many to many relationship through the ratings table.  This table stores user ratings for books.  Both users and books can have zero ratings.

Some notes about tables in Rails:
1. We name our entities using the plural rather than singular.  The model that is associate with an entity set, is the singular form of the plural used for the table.  Thus, we will have a books entity set (table) in the database, and a Book model.  Each Book object will represent one row in the books table.

1. Rails wants the primary key to be named `id` for entity sets.  The norm is for this to be an autoincrementing column.  There is a gem `composite_primary_keys` to support using composite primary keys, but in general when it makes sense for entities, we should stick with the `id`.  

1. When we have foreign keys, Rails will name them with the singular of the foreign table name with an `_id` suffix.

1. Relationship sets, also known as join tables in Rails, are typically named by joining the two table names with an underscore and ordering them alphabetically.  For example, books_readinglists.  We don't have to do that, for example, ratings.

#### Getting Rails up and running

You are reading these directions having cloned the repository to a directory named `goodbooks`.  First:

```
cd goodbooks
```

and then do:

```
rails new . --database=postgresql --skip-test --skip-action-mailer --skip-action-mailbox --skip-action-text --skip-action-cable --skip-active-storage --skip-turbolinks 
```

to create your rails install.  Then:

```
 bundle install
 ```

 Then do:

```
rails db:create db:migrate
```

This will create your databases and put in them the tables that Rails needs.  

 Then run `rails server -b 0.0.0.0` and go to your "Box URL".  You will see a message to add your codio box to the development environment.  In `config/environments/development.rb` add your host after the `do` and before the `end`.  This will look something like:

```ruby
Rails.application.configure do
  config.hosts << "YOUR-CODIOBOX-3000.codio.io"    
...
end
```

Then `CTRL-C` your web server, and start it up again and make sure you do get "Yay! You're on Rails!".  This is a good time to commit and push to GitHub.

#### Our first table and model

Since `authors` does not have any foreign keys, we can make it before the others.   

We will want to create an `Author` model, and its corresponding table `authors` with a migration.  When creating the model/table, we should decide at least on the data types for each attribute.  We're effectively going to be calling the [add_column](https://api.rubyonrails.org/v6.0.3.2/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_column) method, and the documentation for it show all of the available data type we can specify.  

For `authors`, we only have the `name` attribute, and the `:string` data type is appropriate.  The various books referenced above go into detail about how each datatype is mapped to specific datatypes in each different database. 

We will generate the model `Author` and a migration for the table `authors` with a `name` attribute of type `string` using the following command:

```
rails generate model Author name:string
```

when it runs, we get:

```
      invoke  active_record
      create    db/migrate/20200715231141_create_authors.rb
      create    app/models/author.rb
```

*rough drafts* of our model and migration.  You should always expect that you need to edit these auto-generated files before they are ready for use.

We should work with the migration first:

```ruby
class CreateAuthors < ActiveRecord::Migration[6.0]
  def change
    create_table :authors do |t|
      t.string :name

      t.timestamps
    end
  end
end
```

Let's run the migration and see what happens:

```
rails db:migrate

== 20200715231141 CreateAuthors: migrating ====================================
-- create_table(:authors)
   -> 0.0838s
== 20200715231141 CreateAuthors: migrated (0.0839s) ===========================
```

The `create_table(:authors)` method was run.  We can view the table in the database using psql:

```
psql -d goodbooks_development

goodbooks_development=# \d authors
                                          Table "public.authors"
   Column   |              Type              | Collation | Nullable |               Default
------------+--------------------------------+-----------+----------+-------------------------------------
 id         | bigint                         |           | not null | nextval('authors_id_seq'::regclass)
 name       | character varying              |           |          |
 created_at | timestamp(6) without time zone |           | not null |
 updated_at | timestamp(6) without time zone |           | not null |
Indexes:
    "authors_pkey" PRIMARY KEY, btree (id)
```

We can see that Rails automatically gave us our primary key `id`.

We can see that the `name` attribute is allowed to be null, and we don't want to allow that.  If you look up the [manual page in Postgresql](https://www.postgresql.org/docs/9.1/datatype-character.html#:~:text=The%20notations%20varchar(n)%20and,latter%20is%20a%20PostgreSQL%20extension.), character varying is a special datatype in Postgresql that allows strings of any length, which is non-standard for SQL.  We know that we don't want author names of unlimited size, and we should probably place a limit on their size.

We know from our study of SQL, that we can specify size limits and whether or not an attribute can be null.  These options are [available](https://api.rubyonrails.org/v6.0.3.2/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_column) for us to specify.

Use `\q` to quit psql.

We'd like to change the migration to put a size limit on name of 70 characters and make sure it is never null, but we've already made the table, what should we do?  We can simply rollback the migration:

```
rails db:rollback
```

which does:

```
== 20200715231141 CreateAuthors: reverting ====================================
-- drop_table(:authors)
   -> 0.0129s
== 20200715231141 CreateAuthors: reverted (0.0183s) ===========================
```

and we see that Rails knows to drop the table to undo the create table.

Okay, let's change our migration to put a limit on the size of name and prevent nulls:

```ruby
class CreateAuthors < ActiveRecord::Migration[6.0]
  def change
    create_table :authors do |t|
      t.string :name, limit: 70, null: false

      t.timestamps
    end
  end
end
```

we write `limit:` and not `:limit` because we are [specifying named (keyword) arguments](https://docs.ruby-lang.org/en/2.6.0/syntax/calling_methods_rdoc.html) to a method.  The symbol `:limit` is just a read only string.  The `t.string :name, limit: 70, null: false` is really short hand for: `t.column( :name, :string, limit: 70, null: false )`.  

Now, we can run the migration again:

```
rails db:migrate
```

And checking our work in psql, we can see we succeeded:

```
psql -d goodbooks_development

goodbooks_development=# \d authors
                                          Table "public.authors"
   Column   |              Type              | Collation | Nullable |               Default
------------+--------------------------------+-----------+----------+-------------------------------------
 id         | bigint                         |           | not null | nextval('authors_id_seq'::regclass)
 name       | character varying(70)          |           | not null |
 created_at | timestamp(6) without time zone |           | not null |
 updated_at | timestamp(6) without time zone |           | not null |
Indexes:
    "authors_pkey" PRIMARY KEY, btree (id)
```

### User model, users table

Another table without any foreign keys is the users table:  

users ( id (PK), name, email ) 

What will be the rails command to generate the User model and users table?

```
rails generate model User name:string email:string
```

**Task:** Edit the generated migration to limit the size of `name` to 70 characters, and `email` to 255 characters.  Also edit the migration so that neither `name` nor `email` can be null.  Run your migration and verify the correct construction of your users table by using psql.

Your table should like like this in psql:

```
goodbooks_development=# \d users
                                          Table "public.users"
   Column   |              Type              | Collation | Nullable |              Default
------------+--------------------------------+-----------+----------+-----------------------------------
 id         | bigint                         |           | not null | nextval('users_id_seq'::regclass)
 name       | character varying(70)          |           | not null |
 email      | character varying(255)         |           | not null |
 created_at | timestamp(6) without time zone |           | not null |
 updated_at | timestamp(6) without time zone |           | not null |
Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
Referenced by:
    TABLE "readinglists" CONSTRAINT "fk_rails_7c626f1dc9" FOREIGN KEY (user_id) REFERENCES users(id)
```

### Associations: books table

All of the remaining tables have foreign keys, and so now we can start working on creating them.  Let's first work on the `books` table:

books ( id (PK), title, year, author_id (FK) )  

We can actually specify more about the table as part of the rails generate command.  To see how to use it, do:

```
rails generate model
```

For our `string` attributes, we can specify the length limit, for example: `title:string{255}`.  

A nice option is that we can say that the books table "references" an author in the authors table.  What Rails will do is create an attribute, author_id, and set it as a foreign key.  Thus, our generate syntax for the Book model is:

```
rails generate model Book title:string{255} year:integer author:references
```

The migration it produces:

```ruby
class CreateBooks < ActiveRecord::Migration[6.0]
  def change
    create_table :books do |t|
      t.string :title, limit: 255
      t.integer :year
      t.references :author, null: false, foreign_key: true

      t.timestamps
    end
  end
end
```

is close to what we want.  We also want the year and title attributes to not be allowed to be null, either.  (Should `year` be an integer or a text string?  An integer seems pretty obvious, but what if we have some strange item that we want to describe its year as 'circa 1970'? For now, we'll use integers, but these are important design questions you need to ask when designing your database.)

Once updated, the migration looks like:

```ruby
class CreateBooks < ActiveRecord::Migration[6.0]
  def change
    create_table :books do |t|
      t.string :title, limit: 255, null: false
      t.integer :year, null: false
      t.references :author, null: false, foreign_key: true

      t.timestamps
    end
  end
end
```

Let's run the migrations with `rails db:migrate` and look at the table in psql:

```
                                          Table "public.books"
   Column   |              Type              | Collation | Nullable |              Default
------------+--------------------------------+-----------+----------+-----------------------------------
 id         | bigint                         |           | not null | nextval('books_id_seq'::regclass)
 title      | character varying(255)         |           | not null |
 year       | integer                        |           | not null |
 author_id  | bigint                         |           | not null |
 created_at | timestamp(6) without time zone |           | not null |
 updated_at | timestamp(6) without time zone |           | not null |
Indexes:
    "books_pkey" PRIMARY KEY, btree (id)
    "index_books_on_author_id" btree (author_id)
Foreign-key constraints:
    "fk_rails_53d51ce16a" FOREIGN KEY (author_id) REFERENCES authors(id)
```

We nicely see the `author_id` is a FK referencing authors(id).  If we also do `\d authors`, we see that `authors` knows it is referenced by `books`.

Sometimes we want to delete all records that depend on a foreign key when we delete an item from the database.  For example, if you delete an author, you could have all books with that author automatically deleted for you if the foreign key was marked to `ON DELETE CASCADE`.  At the moment, the foreign key will prevent us from deleting an author until we also delete all the books that depend on it.  Given that for this database, it is likely a mistake to get rid of books, we will not set up a cascading delete.  Most likely someone would mistakenly delete an author when what they really wanted to do was update it with a new spelling.  See the [manual](https://api.rubyonrails.org/v6.0.3.2/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_foreign_key) for options.

**Task:** Generate the model and migration for readinglists.  The relation is: readinglists ( id (PK), name, user_id (FK) ) .  The `name` attribute should be a string with a limit of 100 characters and cannot be null.  Run the migration and verify that you created it correctly with psql.

You table in psql should look like:

```
goodbooks_development=# \d readinglists
                                          Table "public.readinglists"
   Column   |              Type              | Collation | Nullable |                 Default
------------+--------------------------------+-----------+----------+------------------------------------------
 id         | bigint                         |           | not null | nextval('readinglists_id_seq'::regclass)
 name       | character varying(100)         |           | not null |
 user_id    | bigint                         |           | not null |
 created_at | timestamp(6) without time zone |           | not null |
 updated_at | timestamp(6) without time zone |           | not null |
Indexes:
    "readinglists_pkey" PRIMARY KEY, btree (id)
    "index_readinglists_on_user_id" btree (user_id)
Foreign-key constraints:
    "fk_rails_7c626f1dc9" FOREIGN KEY (user_id) REFERENCES users(id)
```

### Join table: books_readinglists

The relationship set:

books_readinglists ( book_id (PK, FK), readinglist_id (PK, FK) )

is in Rails terms, a "join table" where we have a many to many relationship between books and readinglists.  

This relationship set has no extra attributes beside the foreign keys, and thus there is no Rails model to go along with it.  We will only be using it to connect two models: books and readinglists.  

Thus, we only want to generate a migration and not a model, too.

To do this, we can run `rails generate migration`, which to see how to use do:

```
rails generate migration
```

In that usage information, and in this [guide](https://edgeguides.rubyonrails.org/active_record_migrations.html), we can see that if we are very careful in naming our migration and how we specify the tables, we can get it to be a join table with book_id and readinglist_id with a unique index on them, which makes them act as a composite primary key.  To do this we do:

```
rails generate migration CreateJoinTableBooksReadinglists books readinglists:uniq 
``` 

This is our migration:

```
class CreateJoinTableBooksReadinglists < ActiveRecord::Migration[6.0]
  def change
    create_join_table :books, :readinglists do |t|
      # t.index [:book_id, :readinglist_id]
      t.index [:readinglist_id, :book_id], unique: true
    end
  end
end
```

Run the migration with `rails db:migrate` and view the table in psql:

```
goodbooks_development=# \d books_readinglists
            Table "public.books_readinglists"
     Column     |  Type  | Collation | Nullable | Default
----------------+--------+-----------+----------+---------
 book_id        | bigint |           | not null |
 readinglist_id | bigint |           | not null |
Indexes:
    "index_books_readinglists_on_readinglist_id_and_book_id" UNIQUE, btree (readinglist_id, book_id)
```

The index looks right. If the index was not `UNIQUE`, then we could have multiple identical rows in the table.  

Unfortunately, the table lacks foreign key constraints. Fortunately, we can add them separately.  Let's put it in a new migration:

```
rails generate migration AddReferencesToBooksReadinglists
```

This produces an empty migration:

```ruby
class AddReferencesToBooksReadinglists < ActiveRecord::Migration[6.0]
  def change
  end
end
```

The [manual](https://api.rubyonrails.org/v5.1.7/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_foreign_key) tells us how to add foreign keys:

```ruby
class AddReferencesToBooksReadinglists < ActiveRecord::Migration[6.0]
  def change
      add_foreign_key :books_readinglists, :readinglists, column: :readinglist_id, primary_key: "id"      
      add_foreign_key :books_readinglists, :books, column: :book_id, primary_key: "id"                  
  end
end
```

Make the changes above to the migration, and then run `rails db:migrate`.

When we examine the `books_readinglists` table, we find that we've added the foreign keys:

```
goodbooks_development=# \d books_readinglists
            Table "public.books_readinglists"
     Column     |  Type  | Collation | Nullable | Default
----------------+--------+-----------+----------+---------
 book_id        | bigint |           | not null |
 readinglist_id | bigint |           | not null |
Indexes:
    "index_books_readinglists_on_readinglist_id_and_book_id" UNIQUE, btree (readinglist_id, book_id)
Foreign-key constraints:
    "fk_rails_3079928121" FOREIGN KEY (book_id) REFERENCES books(id)
    "fk_rails_36bee1a32e" FOREIGN KEY (readinglist_id) REFERENCES readinglists(id)
```

### Ratings relationship set

The ratings relationship set:

ratings ( user_id (PK, FK), book_id (PK, FK), rating )

is something that we also want a model for.  We want to be able to access the rating values.  So, let's generate a Rating model:

```
rails generate model Rating rating:integer book:references user:references
```

This produces the following migration for us:

```ruby
class CreateRatings < ActiveRecord::Migration[6.0]
  def change
    create_table :ratings do |t|
      t.integer :rating
      t.references :book, null: false, foreign_key: true
      t.references :user, null: false, foreign_key: true

      t.timestamps
    end
  end
end
```

But there are some issues with this table.  By default, Rails is going to give it a primary key of `id`, which we don't want.  To avoid the `id` primary key, we add `id: false` as an argument to the `create_table`.  Plus, we want rating to not be null.  Also, we want the pair book_id and user_id to be a unique key in the table, so we'll ask for an index to be added:

```ruby
class CreateRatings < ActiveRecord::Migration[6.0]
  def change
    create_table :ratings, id: false do |t|
      t.integer :rating, null: false
      t.references :book, null: false, foreign_key: true
      t.references :user, null: false, foreign_key: true
      t.index [:book_id, :user_id], unique: true
      t.timestamps
    end
  end
end
```

We can run this migration:

```
rails db:migrate
```

and with that, we should have built our schema.

You can see the complete schema by looking in `db/schema.rb`.

## Getting rid of a model or migration

If you generate a model or a migration that you don't want, you can get rid of it.

First, if you have run a migration that you don't want, do `rails db:rollback` to undo the migration.

Next, repeat your `rails generate` command except replace `generate` with `destroy`.  NOTE: this will delete the files that had been generated.  If you want to save any changes in these files, you need to copy it first.

## Models

After all of that, we now have five models in app/models/ :

+ author.rb
+ book.rb
+ rating.rb
+ readinglist.rb
+ user.rb

The first thing we need to do, is make sure each model has the proper assocations noted in it.

The [possible associations](https://api.rubyonrails.org/classes/ActiveRecord/Associations/ClassMethods.html) are:

+ belongs_to
+ has_one
+ has_many
+ has_and_belongs_to_many

We use these to handle the cardinalities such as one-to-one, one-to-many, and many-to-many.

Recall our schema:

+ authors have a one to many relationship with books, and books have a one to one relationship with authors.  In other words:
  * each book has one author (total participation, all books must always have an author)  
  + each author has zero or more books  (we can add and keep authors without a book entry)
+ users have a one to many relationship with readinglists, and readinglists have a one to one relationship with users.  In other words:  
  * each user has zero or more readinglists
  * each readinglist has one user (total participation, all readinglists must always have a user)
+ users and books have a many to many relationship through the ratings table.  This table stores user ratings for books.  Both users and books can have zero ratings.

So, we want the Author model to say that it `has_many :books` and the Book model to say that it `belongs_to :author`.  If you want, you can "in your mind" replace "belongs_to" with "references", which is what we used to define the schema, but you cannot actually write "references" in the Book model.  

We cannot say a Book model `has_one :author`, for we already express that with the `belongs_to`, which is the correct association for having a foreign key.

We want a User model to say it `has_many :readinglists` and each Readinglist to say it `belongs_to :user`.  

The Readinglist model is connected to the Book model via the join table books_readinglists, and we do not access the join table directly, and thus the Readinglist model should say it `has_and_belongs_to_many :books` and likewise the Book model says it `has_and_belongs_to_many :readinglists`.  

Our Rating model `belongs_to :book` and `belongs_to :user` because of the foreign keys.  The Rating model does **not** have any associations.  The associations are on the entities.  While we have a model to store the rating, we will access it via User and Book and their associations.

Each Book model should say it `has_many :ratings` and each User model should say it `has_many :ratings`, too.

The models are thus:

```ruby
class Author < ApplicationRecord
    has_many :books
end

class Book < ApplicationRecord
  belongs_to :author
  has_many :ratings
  has_and_belongs_to_many :readinglists  
end

class Rating < ApplicationRecord
  belongs_to :book
  belongs_to :user
end

class Readinglist < ApplicationRecord
  belongs_to :user
  has_and_belongs_to_many :books
end

class User < ApplicationRecord
    has_many :readinglists
    has_many :ratings
end
```

**Task:** Edit your model files to match the above.

## Seed Data

We need to make some seed data.

As you can read in the [guide about associations](https://guides.rubyonrails.org/association_basics.html), when we add these associations to the models, new methods are added to the model to help us work with models and their relationships to other models.

For example, the Book model `belongs_to :author` which means the following methods were added to the Book model:

+ author
+ author=
+ build_author
+ create_author
+ create_author!
+ reload_author

So, for example, I could create a new author:

```ruby
# adds the author to the database
a = Author.create(name: "Mark Twain" ) 
```

then I could make a book object:

```ruby
b = Book.new( title: "The Adventures of Tom Sawyer", year: 1876 )
b.author = a
b.save # commit to database
```

Likewise, if I have a Book `b`, I can do `b.ratings` and get all of the ratings for that book.  

**Task:** Edit `db/seeds.rb` to populate the database with the following data:

Books  
1. Ender's Game, by Orson Scott Card, 1985
2. The Hunger Games, by Suzanne Collins, 2008
3. 1984, George Orwell, 1949
4. Catching Fire, by Suzanne Collins, 2009

Authors  
1. Orson Scott Card
2. Suzanne Collins
3. George Orwell
4. Mark Twain

Users  
1. Bob, bob@bob.com
1. Mary, mary@mary.com
1. Sue, sue@sue.com
1. Fred, fred@fred.com

Ratings  
1. Bob rated Ender's Game a 5
1. Bob rated 1984 a -3
1. Mary rated Catching Fire a 3
1. Mary rated 1984 a 5
1. Sue rated The Hunger Games a 5

Readinglists  
1. On Bob's list named "May the odds be in my favor": The Hunger Games and Catching Fire
2. On Sue's list named "Gotta Read": 1984 and Ender's Game
3. On Fred's list name "Sue's Favorite": Hunger Games

Load your seed data:

```
rails db:seed
```

Have you committed and pushed your code to GitHub recently?  Commit early and often.

## Model Improvements

**Tasks:**

1. Add a method named `num_ratings` to Book model that returns the number of times a book has been rated by users.

1. Add a method named `average_rating` to the Book model that returns a book's average rating.  If a book has no ratings, return nil.

1. Add a method named `num_readinglists` to the Book model that returns the number of reading lists the book is found on.

1. Add a method named `favorite_book` to the User model that returns a user's favorite book based on their ratings.  The item returned should be a Book object and not an ActiveRecord::Relation.  If a user has no ratings, return nil.

1. Add a method named `books_in_common` to the User model that takes as input the id of another user and returns an array of Book objects that both users have given positive ratings to.  If the two users have no such books in common, return an empty array.

1. Add a method named `most_liked_book` to the Author model that returns a Book object for the author's highest rated book.  If an author has no books rated, return nil.  

## Schema Addition

**Task:** Update the schema via migration(s) to allow users to write reviews of books.  A review may be up to 1024 characters long.  Each review must be associated with the user that wrote the review and the book that the review is about.  Reviews may not be null.  A user may write at most one review for a book.  A book may be reviewed by zero or more users.  Update the models for User and Book to include the appropriate associations.  Include the appropriate foreign key constraints in the database. Reviews are separate from ratings.  You should not modify the ratings table.

# Submit Your Work

1) Put your name into the `Readme.md` file.  

1) Commit your repo and push to GitHub.

1) Verify that when viewing the Readme in GitHub, that it shows your full name.

Please note that you will not be able to mark your work as completed in Codio. You submit your work by committing it and pushing it to GitHub. **The time of your last commit in GitHub will be used as the time of submission.**

