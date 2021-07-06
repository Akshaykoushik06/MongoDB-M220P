# Chapter 1: Driver Setup

## Lecture: Introduction to Chapter 1
- Hello and welcome to Chapter One of the course.
- We've already covered all of the bookkeeping information for the course and explored the application structure. So now it's time to dive in.
- In this chapter we're going to discuss and explore basic read operations, adding field projections, and how to handle different query predicates to allow searching our data set from the UI.
- Our goal is that by the end of this chapter you are comfortable with basic query semantics and basic shaping operations the application might typically require.
- Remember to read those user stories closely and run the unit tests. And I hope you enjoy this chapter of M220.

## Lecture: MFliz Application Architecture
- In this lesson, we'll go over the application architecture for the mflix application.
- An overview of the structure using the tree command in my terminal is as follows:
    - The README.rst file contains detailed instructions for setting up your environment as well as information about the layout of the project.
    - The data directory contains all of the source data we'll be using for the mflix application.
    - The .ini file here will be modified and renamed by you in this course, and it's going to contain information so that the application can connect to your Atlas cluster, as well as other information that the application will use internally. More on that later.
    - There are two directories under the mflix directory
        - api
        - build.
    - Api contains some flask root handlers for the application, and build contains the front end application.
    - Both of these are completed for you to allow you to focus solely on learning to use PyMongo with MongoDB.
    > **Note:** A few sections will have you edit the index style HTML file if necessary. You can find it under -> mflix -> build -> index.html.
    - The db.py file is where most of your effort will be focused. It contains all of the methods that interact with the database. Initially, most of these would just be stubs that you will have to fill out with the required functionality.
    - You do not need to have MongoDB installed, as you'll be using your own MongoDB Atlas cluster.
    - Factory.py contains functionality that assembles the FOS application for running. You don't have to modify this file.
    - Lastly, the tests directory contains all of the unit tests.
    - Throughout the course, you'll be presented with labs that we call tickets.
    - These will contain a user story or some other instructions, along with instructions for running the particular test suite for that ticket.
    - Once all of the tests are passing for that particular task, you'll go to the status page in the UI.
- And that sums up the application architecture.
- Remember to read the README.rst file, as it contains detailed setup instructions.
- The API layer that handles requests when the application is running is implemented in movies.py and user.py.
- Feel free to look at these files, but do not modify them, as doing so may prevent the UI from validating correctly.
- Db.py is where all of the methods that interact with Atlas are located and where you will be doing all of your implementation.
- The tests directory contains all of the unit tests.
- Make sure you read them to see what is being passed into the methods.
- We also highly recommend that you focus on making one test pass at a time.
- If there are four or five unit tests within the test file, write enough functionality to make one test pass.
- Trying to make all of the tests pass on your first try can lead to a lot of frustration.

## Lecture: Mongo Client
- So on this lesson we're going to discuss the MongoClient, and how we can use it to instantiate a connection to Mongo with the Python driver.
- So the MongoClient object is a part of the PyMongo library, and the URI string that we pass to the client is going to contain most of the information about the connection.
-   ```
    from pymongo import MongoClient
    uri = "mongodb+srv://<username>:<password>@<host>/<database>"
    ```
- It has our username and our password, it has the hostname of the server we want to connect to, and the name of the database that we want to connect to by default.
-   ```
    client = MongoClient(uri)
    ```
- So here, as you can see, we've only passed that URI string to the client object to instantiate it.
- But this is also where we would define other configuration options, like how the driver connects to MongoDB, and the nature of the operations that are performed using that connection. But for right now all we're going to use is that URI string.
- And if we take a look at the stats about this connection, you can see the SSL flag is set to true, because we used an Atlas SRV string for our URI.
- That way we got an SSL connection for free.
- We also authenticate against the admin database by default, so that username and password from the SRV string are used to authenticate against the admin database.
- So now that we have our client object for our connection, we can start exploring the databases that we have access to on Atlas.
-   ```
    client.list_database_names()
    ['ABC', 'XYZ', 'mflix']
    ```
- And it looks like we have a few databases right now on our cluster. We're just going to focus on the MFlix database for right now.
- So one useful thing about the client object is that we can use property accessors to actually define database handles.
- So here we've defined a variable for our MFlix database, and all it took was client.mflix.
-   ```
    mflix = client.mflix
    mflix.list_collection_names()
    ['theaters', 'comments', 'watching_pings', 'movies', 'sessions', 'users']
    ```
- We can use list collection names to get a list of the collections that are on this database. And here you can see all the collections we have on the MFlix database.
- So I just want to point out that we can also use a dictionary accessor to define this database handle.
-   ```
    mflix = client['mflix']
    mflix.list_collection_names()
    ['theaters', 'comments', 'watching_pings', 'movies', 'sessions', 'users']
    ```
- So this MFlix object is just as same as before, but we defined it with a dictionary accessor, instead of a property accessor.
- But all the collections are the same, because this refers to the same database.
- So now let's focus on the movies collection on the MFlix database.
- We can define a collection handle using a property or a dictionary accessor.
-   ```
    movies = mflix.movies
    ```
- As you can see here, I've used a property accessor.
- So now let's perform a query against the movies collection.
- We'll get a count of the documents with this command.
-   ```
    movies.count_documents({})
    ```
- And the empty Python dictionary just represents a pipeline that we would use to transform the documents that we get back.
- And it looks like we have about 46,000 movies in this collection.
- So here we've defined a MongoClient object with a few other parameters, in addition to the URI string.
-   ```
    client = MongoClient(uri, connectTimeoutMS=200, retryWrites=True)
    ```
- Here we've specified that we want the driver to wait 200 milliseconds before erroring out on a connection. And we've specified that we want the driver to retry writes in the event of a network error.
- And if we see the stats on this client reflect the parameters that we set when we instantiated it.
- So the connect time out milliseconds has been set to 200, and retry writes has been set to true.
- If we hadn't specified these in the MongoClient object, then these would be set to the defaults of 20,000 and false, respectively.
- So just to recap
    - The MongoClient object accepts a lot of optional keyword arguments to fine tune the connection. But the simplest instantiation would just be with the URI string.
    - After instantiating the client, we can create database handles with property or dictionary accessors, and create collection handles the same exact way.
    - The vast majority of CRUD operations, like querying or updating documents, have to be performed against this collection object.

## Lecture: Basic Reads
- In this lesson, we're going to perform our first read operation in MongoDB.
- So here I've imported PyMongo, set up my URI to connect to my own Atlas cluster, set the client to connect, set the DB object against M 220, and set a collection object called movies.
-   ```
    import pymongo
    uri = "mongodb+srv://<username>:<password>@<host>/<database>"
    client = pymongo.MongoClient(uri)
    m220 = client.m220
    movies = m220.movies
    ```
- This is the `find_one` command, which returns the first document in natural order. Natural order refers to the order in which the database refers to documents on disk.
-   ```
    movies.find_one()
    ```
- But the actual order of documents is determined internally. So we shouldn't rely on the natural order to have any particular structure.Let's see what document it finds.
-   ```
    {
        '_id': ObjectId('adasaf45454'),
        'cast': ['Carmencita'],
        ...
        ...
        ...
    }
    ```
- We also haven't passed anything in the query predicate, so MongoDB isn't going to filter out any documents from the movies collection. It's going to return the first document of any kind.
- This command, however, has a query predicate mandating that Salma Hayek was in the cast of the movie.
-   ```
    {
        movies.find_one({"cast": "Salma Hayek"})
    }
    ```
- The query predicate is just a Python dictionary. And PyMongo will turn this into BSON for us before sending the query to MongoDB.
-   ```
    {
        '_id': ObjectId('adasaf45454'),
        'awards': '1 nomination',
        'cast': ['Carmencita', 'Salma Hayek', 'David Arquette'],
        ...
        ...
        ...
    }
    ```
- And we can see that even though the value of the cast field is an array, we can query it as if it were just a string. We can do this because MongoDB treats arrays as first class objects.
- Now `find_one` is a pretty useful command when we just need one document. But that's often not the case.
- We typically don't want to limit our query to one document, and we also might want to sort on a field in the collection instead of just natural order.
- So this is a regular find query, which we can use to retrieve all documents in a collection.
-   ```
    movies.find({"cast": "Salma Hayek"})
    <pymongo.cursor.Cursor at 0x107c2dc18>
    ```
> `find_one` does not return a cursor. only `find` returns it.
- This query didn't return a document. Instead the driver sends back a cursor object, and we can treat this like any other Python iterable.
-   ```
    movies.find({"cast": "Salma Hayek"}).count()
    29
    ```
- And we see that the cursor had 29 elements.
-   ```
    cursor = movies.find({"cast": "Salma Hayek"})
    from bson.json_util import dumps
    print(dumps(cursor, indent=2))
    ```
- Here we're going to store our cursor in a variable and then print out documents contained in the cursor.
- And look at that. We have access to our documents in the cursor.
- Dumps is from the JSON Util Library. And we're using it to print the output in this nice format.
- An important thing to remember is that we can only iterate through this cursor once.
- Once we iterate through the entire cursor, we can no longer pull any documents from it.
- Now, the previous query returned every matching document in its entirety. But what if we don't need all of that information?
- Say we just one the names of the movies Salma Hayek has been in.
- We can specify that we want Salma Hayek to be in the cast of the movie and then specify that we only want the title back.
-   ```
    movies.find({"cast": "Salma Hayek"}, {"title": 1})
    print(dumps(cursor, indent=2))
    ```
- The second dictionary is referred to as the projection of the query.
- Let's see it in action. That's a lot easier to read. And we didn't even have to pretty print it.
-  But we still have the `'_id'` field in each of the documents.
- MongoDB will return this `'_id'` field unless we explicitly say that we don't want it.
-   ```
    movies.find({"cast": "Salma Hayek"}, {"title": 1, "_id": 0})
    print(dumps(cursor, indent=2))
    ```
- This is the way that we suppress the inclusion of the `'_id'` field in our output. And we can see that we no longer have the `'_id'` field and just have the title, as we expressed.
- Filters and projections are great because they allow more work to be done on the database side and reduce the amount of data sent over the wire.
- We could have just asked MongoDB for all of the documents in the collection. And then in Python code, we could have filtered for movies that have Salma Hayek, and then parse the dictionary for the title of each movie.
- But then MongoDB is sending all of those movies that don't have Salma Hayek in them, even though our application was going to filter those movies out anyway.
- It also would have meant more Python code in our application, which makes our application more complicated.
- So let's recap:
    - In this lesson, we've covered reading with find one, reading with find, and iterating through cursors, and field projections with queries.
    - And that's it for basic read operations.