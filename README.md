# Updating Records in an ORM

## Objectives

1. Write a method that will update existing database records once changes have been made to that record's equivalent Ruby object. 
2. Identify whether a Ruby object has already been persisted to the database. 
3. Build a method that can *either* find and update *or* create a database record. 

## Updating Records
It's hard to imagine a database that would stay totally static and never change. For example, a customer who uses your online market place updates their billing information or makes a new purchase. A user of your social networking site "friends" another user, creating a new association between them. A hospital updates the medical history of one of its patients. In any of these example of an app that uses a database, we need to be able to update, or change, the records that are stored in that database. 

What do we need to do in order to successfully update a record? We need to first find the appropriate record. Then, we make some changes to it and finally, save it once again. 

In our Ruby ORM, in which attributes of a given Ruby objects are stored as an individual row in a database table, we will need to retrieve these attributes, reconstitute them into a Ruby object, make changes to that object using Ruby methods and *then* save those (newly updated) attributes back into the database. 

Let's walk through this process together. 

## Updating Record in a Ruby ORM

For the purposes of this example, we'll be working with a fictitious music management app that allows the user to store their songs. Our app has a `Song` class that maps to a songs database table. Our `Song` class has all the methods it needs to create the songs table, insert records into that table and retrieve records from that table. 

### The `Song` Class

For this example, we'll assume that our database connection is stored in the `DB[:conn]` constant. 

```ruby
class Song

attr_accessor :name, :album
attr_reader :id
  
  def initialize(id=nil, name, album)
    @id = id
    @name = name
    @album = album
  end

  def save
    sql = <<-SQL
      INSERT INTO songs (name, album) 
      VALUES (?, ?)
    SQL

    DB[:conn].execute(sql, self.name, self.album)
    
  end

  def self.create(name:, album:)
    student = Student.new(name, album)
    student.save
    student
  end
  
  def self.find_by_name(name)
    sql = "SELECT * FROM songs WHERE name = ?"
    result = DB[:conn].execute(sql, name)[0]
    Song.new(result[0], result[1], result[2])
  end
end
```

With the `Song` class as defined above, we can create new `Song` instances, save them to the database and retrieve them from the database:

```ruby
ninety_nine_problems = Song.create(name: "99 Problems", album: "The Blueprint")

Song.find_by_name("99 Problems")
# => #<Song:0x007f94f2c28ee8 @id=1, @name="99 Problems", @album="The Blueprint">
```

Now that we see how to create a `Song` instance, save its attributes to the database, retrieve those attributes and use them to re-create a `Song` instance, let's move on to updating records and objects. 

### Updating Songs

In order to update a record, we must first find it:

```ruby
ninety_nine_problems = Song.find_by_name("99 Problems")

ninety_nine_problems.album
# => "The Blueprint"
```

Uh-oh, 99 Problems is off The Black Album, as we all know. Let's fix this. 

```ruby
ninety_nine_problems.album = "The Black Album"

ninety_nine_problems.album
# => "The Black Album"
```

Much better. Now we need to save this record back into the database. To do so, we'll need to use an `UPDATE` SQL statement. That statement would look something like this:

```sql
UPDATE songs 
SET album="The Black Album"
WHERE name="99 Problems";
```

Let's put it all together using our SQLite3-Ruby gem magic. Remember, in this example, we assume our database connection is stored in `DB[:conn]`. 

```ruby
sql = "UPDATE songs SET album=#{ninety_nine_problems.album" WHERE name = ?"

DB[:conn].execute(sql, ninety_nine_problems.name)
```

Here we updated the album of a given song. What happens when we want to update some other attribute of a song?

Let's take a look:

```ruby
Song.create(name: "Hella", album: "25")
```

Let's correct the name of the above song from `"Hella"` to `"Hello"`. 

```ruby
hello = Song.find_by_name("Hella")

sql = "UPDATE songs SET name='Hello' WHERE name = ?"

DB[:conn].execute(sql, hello.name)
```

This code is almost exactly the same as the code we used to update the album of the first song. The only differences are in the particular attribute we wanted to update. In the first case, we were updating the album. In this case, we updated the name. Repetitious code has a smell. Let's extract this functionality of updating a record into an method, `#update`. 

### The `#update` Method

How will we write a method that will allow us to update any attributes of any song? How will we know *which* attributes have been recently updated and which will remain the same? 

The best way for us to do this is to simply update *all* the attributes whenever we update a record. That way, we will catch any changed attributes, while the un-changed ones will simply remain the same. 

For example:

```ruby
hello = Song.find_by_name("Hella")

sql = "UPDATE songs SET name = 'Hello', album = ? WHERE name = ?"

DB[:conn].execute(sql, hello.album, hello.name)

```

Here we update *both* the name and album attribute of the song, even though only the album attribute is actually different. 

Okay, now that we've solved this problem, let's build our method:

```ruby
class Song
  ...
  
  def udpate
    sql = "UPDATE songs SET name = ?, album = ? WHERE name = ?"
    DB[:conn].execute(sql, self.name, self.album, self.name)
  end
end
```

Now we can update a song with the following:

```ruby
hello = Song.create(name: "Hella", album: "25")
hello.name = "Hello"
hello.update
```

## Duplication and Object ID

What happens when we create another song, also called `"Hello"`? After all, this is not *such* an uncommon song title. 

```ruby
another_hello = Song.create(name: "Hello", album: "Hello!!")

another_hello.album = "Hello"

another_hello.update
```
Remember that the SQL statement inside our `#update` method looks like this:

```sql
UPDATE 
songs SET name = ?, album = ? 
WHERE name = ?
```

We are identifying the record we want to update by matching the *name* attribute of the object to the name value of the table row. But, song names are not unique! The above call to `#update` would update the album name of *both songs with the name of `"Hello"`. Oh no!

Looks like we need to find some unique way to identify song objects and match them with their equally unique equivalent song records...

Luckily for us, each song record in the songs table already has a unique value in the `id` column. That is the great thing about the primary key column––it is always unique. 

Song records in the database table have a unique `id`, and our `Song` instances have an `id` attribute, but that attribute gets set to `nil` when an individual song is initialized. We want our `Song` instances to get assigned a unique `id` number, but when? and how?

The unique `id` number of a `Song` instance should *come from the database*. When a song record gets inserted into the database, it automatically gets assigned a unique ID number. We need to grab that ID number *from the database record* and assign it to the `Song` instance's `id` attribute. 

If that sounds confusing, check out this diagram:

![](http://readme-pics.s3.amazonaws.com/Untitled%20drawing.png)

Let's break it down:

* We create a new instance of the `Song` class. That instance has a `name` and `album` attribute. But its `id` attribute is `nil`. 
* The name and album of this song instance are used to create a new database record––a new row in the songs table. That record has an ID of `1` (this would appear to be the first song we've ever saved in our database). 
* The ID of the newly created database record is then taken and assigned to the `id` attribute of the original song object. 

What's so great about this? Well, with this pattern, every instance of the `Song` class that is ever saved into the database will be assigned a unique `id` attribute that we can use to differentiate it from the other `Song` objects we created and that we can use to find, retrieve and update unique songs. 

Now that we are all convinced that this is the behavior we want to implement, let's build it. 

### Assigning Unique IDs on `#save`

At what point in time should a `Song` instance get assigned a unique `id`? Right after it gets `INSERT`ed into the database. At that point, the database record that it is equivalent to will have a unique ID in the ID column. We want to simply grab that ID and use it to assign the `Song` object its `id` value. 

When do we `INSERT` a new record into our database? In the `#save` method:

```ruby
def save
  sql = <<-SQL
    INSERT INTO songs (name, album) 
    VALUES (?, ?)
  SQL
  DB[:conn].execute(sql, self.name, self.album)
end
```
Right after we `execute` the SQL `INSERT` statement  is an appropriate place to assign our `Song` object its unique `id` from the database. 

How do we get the unique ID of the record we just created? We query the database table for the ID of the last inserted row:

```sql
SELECT last_insert_rowid() FROM students
```

**Important:** When we execute the above SQL statement using our SQLite3-Ruby gem, we get back something that may feel unexpected:

```ruby
DB[:conn].execute("SELECT last_insert_rowid() FROM students")
# => [[1]]
```

Recall that whenever we execute SQL statements against our database using the SQLite3-Ruby gem's `#execute` method, we will get back an array of arrays. Here, we used the `last_insert_rowid()` SQL query to request one thing: the last inserted row's ID. Our SQLite3-Ruby gem obliged and gave us an array that contains one array that contains one element––the last inserted row ID. Phew!

So, let's put it all together with our new-and-improved `#save` method:

```ruby
def save
  sql = <<-SQL
    INSERT INTO songs (name, album) 
    VALUES (?, ?)
  SQL
  DB[:conn].execute(sql, self.name, self.album)
  @id = DB[:conn].execute("SELECT last_insert_rowid() FROM students")[0][0]
end
```

Now let's see what happens when we create a new song:

```ruby
hello = Song.create(name: "Hello", album: "25")

hello.name
# => "Hello"

hello.album 
# => "25"

hello.id
# => 1
```

We did it! Now our individual `Song` objects will get assigned a unique `id` attribute, as soon as they are saved to the database. That means that we can refactor our `#update` method such that it will only update the correct, unique record. 

## Using `id` to Update Records

Our `#update` method should identify the correct record to update based on the unique ID that both the song Ruby object and the songs table row share:

```ruby
class Song
  ...
  
  def udpate
    sql = "UPDATE songs SET name = ?, album = ? WHERE id = ?"
    DB[:conn].execute(sql, self.name, self.album, self.id)
  end
end
```

Now we will never have to worry about accidentally updating the wrong record. 

## Refactoring our `#save` Method to Avoid Duplication

Our `#save` method currently looks like this:

```ruby
def save
  sql = <<-SQL
    INSERT INTO songs (name, album) 
    VALUES (?, ?)
  SQL

  DB[:conn].execute(sql, self.name, self.album)
  @id = DB[:conn].execute("SELECT last_insert_rowid() FROM students")[0][0]
end
```

This method will *always `INSERT` a new row into the database table*. But, what happens if we accidentally call `#save` on an object has already been persisted? That already has an analogous database row?

It would have the effect of creating a new database row with the same attributes as an existing row. The only difference would be the `id` number:

```ruby
hello = Song.new("Hello", "25")
hello.save

DB[:conn].execute("SELECT * FROM songs WHERE name = "Hello" AND album = "25")
# => [[1, "Hello", "25"]]

# what happens when we save the same song all over again?

hello.save

DB[:conn].execute("SELECT * FROM songs WHERE name = "Hello" AND album = "25")
# => [[1, "Hello", "25"], [2, "Hello", "25"]]
``` 

Oh no! We have two records in our songs table that contain the same information. It is clear that our `#save` method needs some failsafes to protect against this kind of thing. 

We need our `#save` method to check to see if the object it is being called on has already been persisted. If so, *don't `INSERT` a new row into the database*, simply *update* an existing one. Now that we have our handy `#update` method ready to go, this should be easy. 

How to we know if an object has been persisted? If it has an `id` that is not `nil`. Remember that an object's `id` attribute gets set only once it has been `INSERT`ed into the database. 

Let's take a look at our new `#save` method:

```ruby
def save
  if self.id
    self.update
  else
    sql = <<-SQL
      INSERT INTO songs (name, album) 
      VALUES (?, ?)
    SQL
    DB[:conn].execute(sql, self.name, self.album)
    @id = DB[:conn].execute("SELECT last_insert_rowid() FROM students")[0][0]
  end 
end
```

Great, now our `#save` method will never create duplicate records!

<a href='https://learn.co/lessons/orm-updating-records' data-visibility='hidden'>View this lesson on Learn.co</a>
