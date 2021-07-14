# Chapter 2: User - Facing Backend

## Lecture: Introduction to Chapter 2
- Hello and welcome to Chapter 2 of the M220 Developer Course.
- I hope you've had success developing the mflix application into a MongoDB database client. 
- Setting up the driver to read data from Mongo into your application is fundamental. And I hope you've enjoyed building that crucial piece of infrastructure.
- Now that we do have a connection from mflix to Mongo, we can start really leveraging the driver to create a more functional and durable backend.
- In this chapter, we'll learn why some complex queries are not possible with the query language alone and how the aggregation framework can actually help us build these queries.
- We'll start using the driver to write data to MongoDB. But not all those writes are created equal.
- We'll explore the nature of these writes to determine which are critical to our application and then use the driver to increase the durability of those writes.
- As far as mflix is concerned, the application's functionality will grow immensely in this chapter.
- New users will be able to join the site, update their preferences, and leave reviews on the movies they feel strongly about.
- We'll even allow users to edit or remove their own reviews, but make sure they can't mess with anyone else's.
- All told, this is a dynamite chapter designed to make you comfortable writing mflix data to Mongo.
- At the end of the chapter, mflix users should be able to not only read about their favorite movies, but actually create a community on the site.

## Lecture: Cursor Methods and Aggregation Equivalents
- So in this lesson, we're going to discuss methods we can call against PyMongo cursors and the aggregation stages that would perform the same tasks in a pipeline.
- The first operation we're going to perform against the cursor is limiting the number of documents that can be returned by that cursor.
- So here's a collection object for the movies collection.
-   ```
    import pymongo
    from bson.json_util import dumps
    uri = "mongodb+srv://<username>:<password>@<host>/<database>"
    client = pymongo.MongoClient(uri)
    m220 = client.m220
    movies = m220.movies
    ```
- So this is a find query with a predicate and a projection.
-   ```
    limited_cursor = movies.find(
        { "directors": "Sam Raimi" },
        { "_id": 0, "title": 1, "cast": 1 }
    ).limit(2)
    print(dumps(limited_cursor, indent=2))
    ```
- The find method is always going to return a cursor to us. But before assigning that cursor to a variable, we can limit the number of documents that return in that cursor with `limit` method.
- And we can see, the cursor only returned two documents to us.
- Now, this is the equivalent operation with the aggregation framework.
-   ```
    pipeline = [
        { "$match": { "directors": "Sam Raimi" } },
        { "$project": { "_id": 0, "title": 1, "cast": 1 } },
        { "$limit": 2 }
    ]
    limited_aggregation = movies.aggregate(pipeline)
    print(dumps(limited_aggregation, indent=2))
    ```
- Instead of tacking a `limit` to the end of the cursor, we add `$limit` stage to our pipeline.
- And we can see, it's the same output.
- These match and project stages represent the query predicate and the field projection from when we were using the query language.
- So the next operation we're going to perform against the cursor is sorting.
- This is an example of the sort cursor method.
-   ```
    from pymongo import DESCENDING, ASCENDING
    sorted_cursor = movies.find(
        { "directors": "Sam Raimi" },
        { "_id": 0, "year": 1, "title": 1, "cast": 1 }
    ).sort("year", DESCENDING)
    print(dumps(sorted_cursor, indent=2))
    ```
- Sort takes two parameters:
    - The key that we're sorting on.
    - The sorting order.
- In this example, we're sorting on year in increasing order.
- Ascending and descending are values from the PyMongo library to specified sort direction.
- But really, they're just the integers -1 and 1.
- And we can see that these movies were returned to us in order of the year that they were made.
- So here's an aggregation pipeline that performs the same operation.
-   ```
    pipeline = [
        { "$match": { "directors": "Sam Raimi" } },
        { "$project": { "_id": 0, "year": 1, "title": 1, "cast": 1 } },
        { "$sort": { "year": ASCENDING } }
    ]
    sorted_aggregation = movies.aggregate( pipeline )
    print(dumps(sorted_aggregation, indent=2))
    ```
- We've replaced our sort` cursor method with the sort stage and just specified a dictionary with the field we want to sort on and the sorting order. And we have the same output as before.
- So just a special case to note here.
-   ```
    sorted_cursor = movies.find(
        { "cast": "Tom Hanks" },
        { "_id": 0, "year": 1, "title": 1, "cast": 1 }
    ).sort([("year", DESCENDING), ("title", ASCENDING)])
    print(dumps(sorted_cursor, indent=2))
    ```
- Sorting on multiple keys in the cursor method `sort` is going to look a little different than before.
- So when we sort on one key, the sort method takes two arguments:
    - the key.
    - the sort order.
- But when we sort on two or more keys, the sort method takes a single argument, which is an array of tuples.
- And each tuple has the field that we want to sort on and the sorting order.
- And we can see that after sorting on year, the cursor actually sorted each movie title alphabetically.
- So here's the same operation in the aggregation framework.
-   ```
    pipeline = [
        { "$match": {"cast": "Tom Hanks"} },
        { "$project": {"_id": 0, "year": 1, "title": 1, "cast": 1} },
        { "$sort": {"year": ASCENDING, "title": ASCENDING} }
    ]
    sorted_aggregation = movies.aggregate(pipeline)
    print(dumps(sorted_aggregation, indent=2))
    ```
- And you can see that the format of the sort stage doesn't change, depending on the number of fields that we sort on. That's just the cursor method.
- And it looks like this gave us the same results.
- So the last operation we're going to perform against the PyMongo cursor is skipping through the documents on the cursor.
- So this pipeline is going to count the number of documents in the movie's collection that were directed by Sam Raimi.
-   ```
    pipeline = [
        { "$match": {"directors": "Sam Raimi" } },
        { "$project": {"_id": 0, "title": 1, "cast": 1} },
        { "$count": "num_movies" }
    ]
    sorted_aggregation = movies.aggregate(pipeline)
    print(dumps(sorted_aggregation, indent=2))
    ```
- The project stage shouldn't actually affect the number of results we get back. And count is just going to return a single field that has a count of the number of documents returned by this pipeline.
- Looks like there are 15 movies that Sam Raimi directed in this collection.
- Note that the cursor method that counts documents in a cursor has actually been deprecated. So if you want to know how many documents are returned by a query, you should structure the query in the aggregation pipeline and then use this count stage.
- A `skip` method allows us to skip documents in a collection. So only documents we did not skip will appear in the cursor.
- Because we only have 15 documents, skipping 14 of them should only return one back to us.
- And it looks like we are right. It just returned one document because it skipped the first 14.
- The issue is, we don't really know which documents we skipped over because we haven't specified a sort key. And really, we have no idea the order in which documents are stored in the cursor.
- So let's sort and then skip.
-   ```
    skipped_sorted_cursor = movies.find(
        { "directors": "Sam Raimi" },
        { "_id": 0, "title": 1, "year": 1, "cast":: 1 }
    ).sort("year", ASCENDING).skip(10)
    print(dumps(skipped_sorted_cursor, indent=2))
    ```
- Here we're sorting on year in ascending order and then skipping the first 10 moves.
- So now that we know when we skip 10 documents, we're skipping the 10 oldest Sam Raimi movies in this collection.
- And it's a little hard to see here, but it did only return five movies back to us and they were the five most recent Sam Raimi movies in this collection.
- These cursor methods are nice because we can kind of tack them on to a cursor in the order that we want them applied.
- We can accomplish the same ordering in aggregation just by ordering the stages in the pipeline.
- So here we have a sort on year ascending before we skip 10 documents.
-   ```
    pipeline = [
        { "$match": { "$directors": "Sam Raimi" } },
        { "$project": { "_id":0, "year": 1, "title"1, "cast":1 } },
        { "$sort": { "year": ASCENDING } },
        { "$skip": 10 }
    ]
    sorted_skipped_aggregation = movies.aggregate(pipeline)
    print(dumps(sorted_skipped_aggregation, indent=2))
    ```
- It's a little hard to see the results right now because it's cut off by the screen.
- But if you want, you can download the notebook and the handout and then run it to check the output yourself.
- So just to recap in this lesson we covered:
    - Some cursor methods and their aggregation equivalents.
    - Remember that there won't always be a one-to-one mapping just because the aggregation framework can do a lot more than cursors can.
    - But these three methods exist as both aggregation stages and cursor methods.
    - `$sort`, `$limit`, `$skip` - these are the aggregation stages that have equivalen cursor methods.

## Lecture: Basic Aggregation
- So in this lesson, we're going to briefly cover the Aggregation framework in MongoDB. We're going to use the Aggregation builder in Compass to export this pipeline to the language of our choice.
- So the first thing to know about Aggregation in MongoDB is that it's a pipeline composed of one or more stages. And within each stage, we define expressions that evaluate and transform the data at that particular stage.
- To use an analogy, documents flow through the pipeline, sort of like the widgets do on a factory conveyor belt.
- Each stage is like an assembly station.
- Documents enter, some work is performed, and then some output is produced that gets piped in as the input to the following stage.
- And at their core, expressions are functions.
- So here we're going to take a look at the `add` function in three different programming languages, as well as the Aggregation framework.
- So here it is in Python. Simple function, has two inputs, and returns the sum of those two inputs.
-   ```python
    def add(a, b):
        return a + b
    ```
- So here in Java, we have a little more strict typing.
- So we've just specified that the type of each of our input has to extend this number interface, which is to say that we can add the two inputs.
- But it's more or less the same function.
-   ```java
    static<T extends Number> double add(T a, T b) {
        return a.doubleValue() + b.doubleValue();
    }
    ```
- And here it is in JavaScript. It's about as short as it is in Python, but the syntax looks a little bit like Java.
-   ```javascript
    function add(a, b) {
        return a + b
    }
    ```
- So now, here is the add function in aggregation.
-   ```
    { "$add": ["$a", "$b"] }
    ```
- Essentially it's this add expression, to which we pass an array of the values that we want to sum up.
- So all expressions and stages in the Aggregation framework will have a dollar sign `$` before them.
- And the dollar sign is just how we refer to variables within the expression.
- We have a course that dives much more deeply into Aggregation, covering syntax and semantics at almost every stage. You can find more by looking at the lesson handout. Also included is a link to the Aggregation quick reference.
- So now, let's jump into Compass and start building some Aggregations.
- So here I've just opened up Compass and connected to my Atlas cluster.
- Currently I'm connected to the movie's collection on the MFlix database.
- And I'm using the Aggregations tab so we can write a pipeline against the movies collection.
- The first stage we're going to add is a `match` stage to only pick up movies that were directed by Sam Raimi. The way that we're going to do that is by specifying the field and the value that we want.
- And we can see this is only returning movies to us that have Sam Raimi as a director in them.
- So the way the match stage works is actually very interesting and somewhat subtle.
- So here the director's field actually has an array as its value.
- And that array has all the directors for that specific movie. But we only need to specify the one that we want to match against, and it will actually parse the array out for us.
- So now let's add another stage to this pipeline.
- We're going to try to figure out the average IMDb rating for all the movies that Sam Raimi has directed.
- Just going to add a new stage here.
- So here's a `project` stage in our pipeline.
- This is going to essentially choose which fields we want to display and suppress from the previous stage in the pipeline.
- I don't really care about the underscore ID in this case. So I'm gonna say I don't want it with `_id: 0`.
- I do want the title of the movie. And I need the IMDb rating in order to find the average.
- Now because rating is contained within an IMDb object, we have to use dot notation in order to project that field. And when we use dot notation, we have to just surround it in quotes like this `"imdb.rating":1`
- And it looks like this did what we wanted it to do. We only got back the title and the IMDb rating for each movie.
- So now that we have all the IMDb ratings, we can find the average. And to do so, we're going to use a `group` stage.
- So the group stage can be used for a lot of different purposes, but the way we're using it right now is to find the average of all of the movies Sam Raimi directed.
- We've already matched all the movies that Sam Raimi directed, so we don't need to transform the input.
- But we really do want to find the average of all the movies. So I've specified a 0 to `_id` here.
- And as the output of this stage, which is the last stage in our pipeline, we see the grouping criteria, which was none, and the average rating value, which was 6.86. Way to go, Sam Raimi.
- So just to recap:
    - Aggregations in MongoDB are pipelines that are composed of one or more stages.
    - And each stage uses functions that we write in the Aggregation syntax to transform and evaluate the data.

## Lecture: Basic Writes
- In this lesson, we're going to cover basic write operations in MongoDB.
- Here, we've imported our dependency PyMongo, set up our URI to connect to our Atlas cluster, set up a client and database object.
-   ```
    import pymongo
    uri = "mongodb+srv://<username>:<password>@<host>/<database>"
    client = pymongo.MongoClient(uri)
    db = client.electronicsDB
    ```
- We can view the collections contained in this database with this command: `db.collection_names()`
- And right now, the result is `['video_games']`.
- So here, we're creating a collection object to point to our video_games collection like this: `vg = db.video_games`
- And this command is inserting a single document into the video_games collection. This statement will return a result object which we're assigning to a variable called `insert_result`.
-   `insert_result = vg.insert_one({"title": "Fortnite", "year": 2018})`
- Let's go ahead and look at the acknowledged property on insert_result.
-   ```
    insert_result.acknowledged
    True
    ```
- And we see that it's `True`. This tells us that MongoDB acknowledged our right.
- And here, we can see the inserted ID property. This will tell us the _ID of the inserted document.
-   ```
    insert_result.acknowledged
    ObjectId('5aec71834578632f4fea6548')
    ```
- And we can see an object ID above.
- Now, if we wanted, we could query directly on this object ID to retrieve the document we just inserted.
-   ```
    vg.find_one({"_id": insert_result.insert_id})
    {
        '_id': ObjectId('5aec71834578632f4fea6548'),
        'title': 'Fortnite',
        'year': 2018
    }
    ```
- The `insert` statement is a useful type of write operation, and it's very simple. It inserts new data into the database without regard for what the new data is or was already in the database.
- So what happens if we want to insert our document but we're not sure if it's already in the database?
-   ```
    fortnite_doc = {"title": "Fortnite", "year": 2018}
    upsert_result = vg.update_one({"title": "Fortnite"}, {"$set": fortnite_doc}, upsert=True)
    ```
- Above statement is called an `upsert`, and it's going to check for a document matching `{"title": "Fortnite"}`. If it doesn't find it, then it will insert this document here. If it does find it, then it will set this `fortnite_doc` on the existing document.
- In this case, the operator is `set`, which means any fields that we specify here will be set in the document we find with the passed value - here it is `fortnite_doc`.
- This is a way for insert operations to be idempotent. Because we can run upserts multiple times with the same result.
- The first time it doesn't find the document, it will perform an insert, but all the other times it will find the document and nothing will change.
- And it looks like this upsert did find the matching document, because we just inserted this exact document into the database with our insert command above.
-   ```
    upsert_result.raw_result
    ```
- `updatedExisting` in the above output, tells us that MongoDB found the document we were looking for and attempted to update it. But even though it found the document, it doesn't mean it wrote any new data to the database.
- The new document was the same as the old document, so after performing our upsert, we didn't have to actually update anything.
- That's why `nModified` here is 0.
-   ```
    rocketleague_doc = {"title": "Rocket League", "year": 2015}
    upsert_result = vg.update_one({"title": "Rocket League"}, {"$set": rocketleague_doc}, upsert=True)
    ```
- Now this here is a brand new document that doesn't exist in the database. As we can see, the update result is a little different than the one we had previously.
- `updatedExisting` is false, `nModified` is still 0, and we can see this new field called `upserted` with an object ID for the object that was inserted into the database.
- All right, let's go ahead and recap:
    - The things that we covered in this lesson are basic inserts, basic upserts, and consuming result objects.
    - You can find the full list of methods available and their associated result objects by looking in the PyMongo documentation, which will be linked in the reference documentation to this lesson.
    - These methods will vary depending on the type of operation used to produce the result object.
    - And that's it for basic inserts.
    - The following are true about the object returned by `insert_one`:
        - It can tell us whether the operation was acknowledged by the server.
        - It contains the `_id` of an inserted document.

## Lecture: Write Concerns
- All right, so in this lesson, we're going to discuss write concerns and how they can provide a different level of write durability in our application.
- If you're not sure right now what write durability is, no worries. We'll cover it in this lesson.
- So for right now, let's just consider a small supermarket application using a replica set as its data source.
- Whenever a customer puts an item into their cart, the application will send a write statement over to MongoDB, and that write will be received for that primary node of the replica set.
- So the first thing the primary node's going to do when it receives this write is it's going to apply the write and its copy of the data.
- And by default, as soon as it's done performing the write in its database, it's going to send an acknowledgment back to the client.
- So at this point, the client has received a write acknowledgment back from the database, and it considers the write to be complete.
- It assumes that the secondary nodes will replicate the data sometime soon, but it doesn't actually have any immediate proof of it from this acknowledgment alone.
- So that was an example of a write with writeConcern `{w: 1}`.
- The number 1 here refers to the number of nodes in this set that must apply the write before a client gets an acknowledgment back from the driver. In this case, it was just **one** node.
- This is the default behavior in MongoDB, so if you send a write to MongoDB without a writeConcern specified, it will use {w 1} by default.
- So now let's consider a different level of write concern.
- Our shopping cart application sends a write statement to the primary node, and the primary applies that write just like it did before.
- But this time, the primary waits before sending an acknowledgment back to the client.
- And what is it waiting for?
- Well, before sending an acknowledgment of the write back to the client, the primary will actually wait for one of the secondary nodes to replicate the data.
- When the secondary applies the write, it will send an acknowledgment back to the primary saying, hey, I applied this write to my copy of the data.
- Once the primary knows that in addition to it having applied the write itself, one of the secondaries has also applied the write, only then will it send an acknowledgment back to the client.
- This write was sent with `w: majority`, which means that the client isn't going to get an acknowledgment back from the driver until a majority of nodes in the set have applied the write.
- In this case, this is a three-node set, so we only needed two of the nodes to apply the write.
- You can think of `w: majority` as a contract with the client that this write will not be lost, even in the event of hosts going down.
- If an application sends a write with `w: majority` and gets an acknowledgment back for that write, it knows that even if the current primary were to go down, one of the secondaries in the set has also captured the write.
- So with `w: majority`, the connection is going to wait for a majority of nodes to apply the write before sending an acknowledgment back to the client.
- For that reason, it takes a little longer and is subject to replication lag.
- But there's no additional load on the server, so the primary can still perform the same number of writes per second.
- However, `w: majority` essentially guarantees to the client that a write will not be rolled back during fail over, because the write was committed to a majority of nodes.
- This is useful when some of our writes are vital to the success of the application.
- A common example of this is a new user on a website.
- These types of operations must succeed, because without an account, the user can't really do anything else on the site.
- So I just want to discuss one more write concern, `{w: 0}`.
- By now, you must have realized that when the value of w is a number, it's the number of nodes that must apply a write before the client receives an acknowledgment.
- We can pass any number here to the w field, but it will throw us an error if this number is higher than the total number of nodes in the set.
- Following that rule, when w is 0, none of the nodes in the set actually need to apply a write before the client gets acknowledgment.
- This means that when we're `{w: 0}`, there's a chance that we get acknowledgment before any data has actually been written.
- So if the server crashes, we might lose a few writes.
- This type of write is referred to as a **fire and forget** operation, because it sends the write and doesn't really worry about the response.
- But this isn't entirely true, because the acknowledgment from a `{w: 0}` operation can also alert us to network errors and socket exceptions.
- So the client can implement some logic to figure out if the write was actually received by the database.
- In any case, writing with `{w: 0}` is very fast and can be useful for less important writes that occur frequently.
- For example, if an internet of things device is sending a ping to Mango every two minutes to report its status, it might be OK to speed up every write operation at the risk of losing a few writes.
- So to recap:
    - `{w: 1}` is the default write concern in Mongo, and it commits a write to one node before sending an acknowledgement back to the client.
    - `{w: majority}` will make sure the write was applied by a majority of the set before sending an acknowledgment back to the client. This means the application will have to wait a little longer for a response, but it should not have a performance impact so long as you have enough connections to the primary to handle your requests.
    - `{w: 0}` does not commit the write at all, but sends an acknowledgement back to the client immediately. So there's a slightly higher chance that we lose data in the event of a primary going down.

## Lecture: Basic Updates
- In this lesson, we'll discuss update operations with PyMongo.
- Update operations in PyMongo are identical to what you've already learned.
- In PyMongo, the two idiomatic update methods are `update_one` and `update_many`.
- Let's dive in to a quick example.
- First, we'll bring in our dependencies. For this lesson, we'll also bring in Faker, which was included as a dependency of the mflix application.
-   ```python
    import pymongo
    from bson.json_util import dumps
    from faker import Faker
    import rndom
    fake = Faker()
    fake.seed(42)
    random.seed(42)
    uri = "mongodb+srv://<username>:<password>@<host>/<database>"
    client = pymongo.MongoClient(uri)
    mflix = client.mflix
    ```
- Faker is a library that allows us to easily generate fake data, especially useful if we want to create documents to start testing against.
- We've seeded Faker and ran them here to ensure consistent results.
- One thing of note-- make sure to change the URI to your own Atlas cluster if you'd like to follow along.
- Next, we'll create a collection to work with called `fake_users`. If it exists already, we'll drop it to ensure we're working with fresh data.
-   ```python
    fake_users = mflix.fake_users
    fake_users.drop()
    ```
- Here, I've made a small utility function to return back a dictionary representing a fake user.
-   ```python
    def make_user(iter_count):
        account_type = "premium" if iter_count % 2 == 0 else "standard"
        return {
            "name": fake.name(),
            "address": fake.address(),
            "email": fake.email(),
            "age": random.randrange(18, 65),
            "favourite_colors": [fake.color_name(), fake.color_name(), fake.color_name()],
            "account_type": account_type
        }
    ```
- It takes this `iter_count` argument so that it can determine whether a user will be given a standard or premium account.
- Here, I'm using Faker to generate a name, address, email, and favorite colors, and using rand to give an age.
- OK, let's create a list of 10 users to insert and insert them into our fake_users collection.
-   ```python
    to_insert = [make_user(i) for i in range(10)]
    fake_users.insert_many(to_insert)
    ```
- OK, let's find a user to perform some update operations on with a `find_one` method.
-   ```python
    print(dumps(fake_users.find_one(), indent=2)
    ```
- And it looks like we'll work with Allison Hill - the name was Allison Hill from the output of the above command
- Now, let's suppose Allison just had a birthday and we want to increment her age. Just like you've already learned, we can use the `$increment` operator.
-   ```
    allison = {"name": "Allison Hill"}
    fake_users.update_one(allison, {"$inc": {"age": 1}})
    ```
- It'll increment her age by one, from 19 to hopefully 20. And we can see that Allison is now 20 years old. Happy birthday, Allison.
- Next, let's add black to Alison's favorite colors.
-   ```
    fake_users.update_one(allison, {"$push": {"favourite_colors": "Black"}})
    ```
- Timeless. And let's view the results of that push operation.
- Great. We can see that black was pushed into Alison's favorite colors.
- Let's get a count of users with an account type of standard. It should be five.
-   ```
    print(fake_users.count({"account_type": "standard"}))
    ```
- Now, let's list their names and account types out so that we can reference the names in a little bit.
-   ```
    print(dumps(fake_users.find({"account_type": "standard"}, { "_id": 0, "name": 1, "account_type": 1}), indent=2))
    ```
- Great. Now, let's run a promotion where we give everyone a premium account for a week.
- Let's set all of their account types to premium, but also add a key called `free_trial` which will be set to true so we can track them.
- I've assigned this operation to the variable `u_r`, short for update result, as we'll cover that next.
-   ```
    u_r = fake_users.update_many({"account_type": "standard"}, {"$set": { "account_type": "premium", "free_trial": True}})
    ```
- Now that we've updated, let's go ahead and get a count of all users with an account type of standard. It should be 0.
-   ```
    print(fake_users.count({"account_type": "standard"}))
    ```
- Great. Now, let's find all users where the free_trial key is set to true. We'll also print their name and account type out for comparison to our previous output.
-   ```
    print(dumps(fake_users.find({"free_trial": True}, { "_id": 0, "name": 1, "account_type": 1}), indent=2))
    ```
- And we can see the names are the same, and their account type is now premium.
- Let's take a closer look at that update result object that we've been getting back from our update operations.
- First, we'll look at all of the attributes on it using `print(dir(u_r))`
- And we can see that there are quite a few. However, the important ones are `acknowledged`, `matched_count`, `modified_count`, and `upserted_id`.
- Let's go ahead and inspect just those ones.
-   ```
    print(u_r.acknowledged, u_r.matched_count, u_r.modified_count, u_r.upserted_id)
    ```
- And we can see, for our last update operation, we had acknowledged true five matched, five modified, and none for upserted_id.
- So let's imagine that we've combined our user signup flow with updating their information. We'll create another fake user.
- And this time, we'll specify upsert equals true after the update operation.
- We know this user's email won't match anything we currently have thanks to the library figure.
-   ```
    new_or_updated_user = make_user(0)
    u_r = fake_users.update_one({"email": new_or_updated_user["email"]}, {"$set": new_or_updated_user}, upsert=True)
    ```
- All right. And now let's search for the new user.
-   ```
    print(dumps(fake_users.find_one({"email": new_or_updated_user["email"]}), indent=2))
    ```
- And we can see that their record was successfully written.
- Let's look at the update result that was returned by that operation.
-   ```
    print(u_r.acknowledged, u_r.matched_count, u_r.modified_count, u_r.upserted_id)
    ```
- All right. And we can see true for `acknowledged`, the 0 for `matched_count`, 0 for `modified_count`, and we now have an `_id` for upserted_id.
- Remember, in the case of an `upsert`, `match_count` and `modified_count` will be 0.
- Lastly, we'll go ahead and drop the collection now that we're done with it. If you'd like to continue experimenting, you can skip this.
-   ```
    fake_users.drop()
    ```
- Let's go ahead and recap what we've learned:
    - The two idiomatic update operations in PyMongo our `update_one` an `update_many`.
    - Update operations will return an update result with important properties `acknowledged`, `matched_count`, `modified_count`, and `upserted_id`.
    - In the case of an upsert, `modified_count` and `matched_count` will be 0.
    - And that covers basic update operations in PyMongo.
    - The following are the valid update operators in pymongo: `$inc`, `$push`, `$set`.

## Lecture: Basic Joins
- So in this lesson, we're going to cover joins in MongoDB.
- So joins are used to combine data from two or more collections, which is true for all database systems, but the implementation is going to be a little different in MongoDB.
- The join we're going to do here is between the movies and comments collection from the mflix database.
- Each comment in mflix is posted by a user, and associated with one specific movie. And we want to count how many comments are associated with each movie.
- Users use comments as a way to discuss movies, so we can think of this sort of like a popularity contest. You know, which movies are being talked about the most on our site.
- We're going to use the new expressive `$lookup` in MongoDB 3.6, so we can express a pipeline for the data that we're joining.
- This might not make sense right now, so we'll explore what it means in a minute.
- We're going to build this pipeline in Compass, and then use Compass' export to language feature to produce code that we can copy directly into our application's native language.
- So here I'm just connected to the mflix database in Compass.
- And we're going to start our aggregation from the movies collection, and then join from the comments collection. Although it would probably work the other way around, as well.
- Just going to move to Aggregations tab.
- And I can select a new stage.
- I'm going to add a `$match` stage to select only the movies that came out in the 1980s.
-   ```
    {
        year: { '$gte': 1980, '$lt': 1990 }
    }
    ```
- And now that our `$match` stage is fully written out, it looks like Compass has already loaded the documents that would be returned by this query. And it looks like all of these movies came out in the 1980s.
- So now here's the stage where the join actually happens.This is a `$lookup` stage in the expressive version. And there are four fields: `from`, `let`, `pipeline`, and `as`.
-   ```
    {
        from: 'comments',
        let: { 'id': '$_id' },
        pipeline: [
            {
                '$match': {
                    '$expr': { '$eq': ['$movie_id, '$$id'] }
                }
            }
        ],
        as: 'movie_comments'
    }
    ```
- `from` is going to be the collection that we're joining from.
- So we're running this aggregation from the movies collection, and we want to join from the comments collection. So that's the one I've specified here.
- `let` is when this starts to get a little complicated, so try to follow closely.
- The `pipeline` we write inside the join has access to fields of documents inside the comments collection, because that's the collection that we're joining from.
- But it doesn't have access to the field inside the movies collection, unless we specify them in `let`.
- So if we want to use the `_id` from the movies documents inside the pipeline, we have to declare this variable `id` and assign it to the `$_id` from the movies collection.
- So if we look inside the pipeline, we can see that we refer to this variable with two dollar signs, because the variables with one dollar sign refer to fields inside the comment documents.
- This obviously can get a little bit complicated with all the dollar signs, but just remember that double dollar sign means that the variable was defined in the `let` statement.
- The pipeline itself only has one match stage right now, and it's matching the movie ID of the comment to the underscore ID from the movie.
- We've set `as` to `movie_comments`, so that the movie document will now have an array field called `movie_comments` that contains a list of all the comments associated with that movie.
- And we can check that that field exists down here.
- It looks like it did create this movie comments field, which is a type array.
- And each element of the array is its own document, which look like the exact comment documents from the comments collection.
- Now I embedded all the comment documents inside each movie, but all I really wanted to figure out was how many comments were associated with each movie.
- I don't really care what each comment says, or who wrote it, or when it was written. I just care how many there are.
- So here I've changed up our look up stage a little bit by adding this `count` stage to the pipeline.
-   ```
    {
        from: 'comments',
        let: { 'id': '$_id' },
        pipeline: [
            {
                '$match': {
                    '$expr': { '$eq': ['$movie_id, '$$id'] }
                }
            },
            {
                '$count': 'count'
            }
        ],
        as: 'movie_comments'
    }
    ```
- `$count` is just going to count all the documents that pass through this pipeline.
- And since we already used a `$match` stage to make sure that each comment was associated only with that movie, this meets our needs perfectly.
- And we can see we've ended up with a single array field with one value that just has a count of the number of comments associated with this movie.
- So this pipeline in the expressive lookup is actually very powerful, because it allows us to transform the comments documents returned by a join on the server before that data even gets embedded inside this movies document.
- And now that we've written out our pipeline, we can verify that our output documents look the way we expect.
- We can export the pipeline to a language that suits our application's needs. We have Python 3, C#, Node JS, and Java available to us.
- So just to recap:
    - Expressive lookup up allows us to pass an aggregation pipeline to the command that can transform the data before that data is actually joined.
    - And let allows us to declare variables in that pipeline that refer to document fields in our source collection.
    - Once we're done writing the pipeline out in Compass, we can use the export to language feature to produce the aggregation in the language that's native to our application.

## Lecture: Basic Deletes
- So in this lesson, we're going to discuss deleting documents in MongoDB.
- So first, I'm just going to import a Mongo client and set up my URI string, then create my client object from the URI.
- And since we're learning about deletes in this lesson, we don't want to work with any of our production data. We're going to create a new collection called deletes.
-   ```python
    from pymongo import MongoClient
    uri = "mongodb+srv://<username>:<password>@<host>/<database>"
    lessons = client.lessons
    deletes = lessons.deletes
    ```
- So now that we have a collection object called deletes with no data in it, we can insert some data.
- We scroll over this Insert Many statement. We can see we're inserting 100 documents with _id values from 0 to 99, and this random_bool field, which will randomly be true or false.
-   ```
    import random
    random.seed(42)
    deletes.drop()
    imr = deletes.insert_many([{'_id': val, 'random_bool': random.choice([True, False])} for val in range(100)])
    assert len(imr.inserted_ids) == 100
    ```
- We'll also run an insertion at the end here to make sure that a 100 object IDs have been inserted into the collection. And if this is not true, we'll see an error.
- We've also added the drop method in the beginning of the code cell to ensure repeatability, in case we want to run through this lesson again.
- And it looks like that worked.
- OK. Let's grab the first three documents to get a sense for what they look like.
-   ```
    list(deletes.find().limit(3))
    ```
- And this is more or less what we were expecting. We're convinced that we have a somewhat random random_bool field, an _id value from 0 to 99.
- So at this point, we already know how to create, read, and update documents. So now we're going to delete some.
- So PyMongo offers two idiomatic delete methods: `delete_one` and `delete_many`.
- We're going to take a look at both to get a sense for how they work.
- So `delete_one` is a lot like find_one in the sense that it looks for the first document that matches the query predicate. The only difference is instead of returning that document, it deletes it.
- If multiple documents match the predicate, delete_one will only delete the first document that it found.
- So next, we're going to use delete_one to delete the first document where that random_bool field was true.
- And if delete_one behaves the way we expect, we should be left with 99 documents afterward.
- So here, we're just assigning the output of this `delete_one` statement to a delete result or `dr` variable. And that variable can actually tell us how many documents were deleted.
-   ```
    dr = deletes.delete_one({'random_bool': True})
    dr.deleted_count
    ```
- And it looks like we only deleted one document, which is what we were expecting.
- `delete_one` is a very precise operation. So if we know a value or values that uniquely identify a specific document, we can run delete_one and guarantee that we're only going to delete that one document.
- We know `_id` is unique. So let's, for example, delete the document with `_id` 99.
- Just verifying that that document exists in the collection, and now we're going to actually run a `delete_one` on this one query predicate.
-   ```
    deletes.find_one({'_id': 99})
    deletes.delete_one({'_id': 99})
    deletes.find_one({'_id': 99})
    ```
- And it looks like that works. Let's just check with `find_one` . And it looks like we couldn't find that document anymore, so it was in fact deleted.
- So next we're going to cover `delete_many`.
- Unlike `delete_one`, `delete_many` will delete all documents that match the supplied predicate. And because of this, you could consider it a little more dangerous.
- So to get a sense of how `delete_many` works, we're going to count how many documents have false and true for their random_bool value, and then we're going to use `delete_many` to delete all the documents that have that value false.
- So first, let's get a count of how many documents in this collection have the random_bool value set to false.
-   ```
    len(list(deletes.find({'random_bool': False})))
    ```
- And it looks like 44, so about half of them - a little less than half. And let's check how many have that value set to true.
-   ```
    len(list(deletes.find({'random_bool': True})))
    ```
- And it looks like we have a few more documents (54) with that value set to true.
- So now we know that 44 of the documents in our collection have random_bool set to false, which means the other 54 have the random_bool you set to true.
- If we run a `delete_many` on the documents that have random_bool set to false, as we have here, the deleted count should yield 44, which it does.
-   ```
    dr = deletes.delete_many({'random_bool': False})
    dr.deleted_count
    ```
- And the number of documents left in the collection that have random_bool set to true should be 54.
-   ```
    len(list(deletes.find({'random_bool': True})))
    ```
- So `delete_many` is behaving the way we're expecting. It's only deleting the documents that match the query predicate, but it will delete all of the documents that match the predicate.
- So that about covers the basics of deleting documents in PyMongo.
    - Remember that `delete_one` will only delete the first document that matches the supplied predicate, but `delete_many` will delete all of the documents that match the predicate.
    - The number of documents that have been deleted can be accessed using the `deleted_count` property on the resulting object of the delete operation. Both `delete_one` and `delete_many` will return this property.