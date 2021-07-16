# Chapter 4: Resiliency

## Lecture: Introduction to Chapter 4
- Welcome to Chapter 4.
- We are almost there. Before we start uncorking the champagne and celebrating your course completion certificate, we need to go through the last chapter of this course.
- This chapter is all about application resilience and robustness.
- We will be looking to how to make your applications resilient to a variety of different situations that may impact the performance and availability of your system.
- Don't forget, we are living in a distributive database world, and therefore, we need to prepare our application to be robust and performant, as well as scalable.
- Hang tight. This is going to be fun.

## Lecture: Connection Pooling
- In this lesson, we're going to cover connection pooling in MongoDB.
- So connection pooling is all about reusing database connections.
- The connection pool itself is just the cache of database connections maintains that they can be reused when future requests to the database are required.
- When issuing several different requests of the database, we could take the lazy approach and just create a new connection whenever we needed to make a request. And then when the request is done, we just destroy the connection.
- The issue with this approach is that establishing a database connection requires time and computing resources to complete the handshake with the server and everything else.
- We're essentially paying the costs of waiting for this connection to be established for every request.
- Connection pooling helps reduce the overhead of creating database connections by creating a whole bunch right off the bat.
- The next requests come in different connections in the pool get allocated to fulfill these requests.
- By default, the driver will create a connection pool of 100 connections to share. The default of 100 connections is adequate for most applications.
- Additionally, if we didn't use a connection pool and suddenly got a whole lot of requests, we might easily reach the limit that our hardware and software could handle, leading to a lot of errors and unhappy developers.
- So just to recap:
    - Connection pools allow connections to be recycled for new requests to the database.
    - To the developer, this will make database operations look faster because the cost to create a new connection has already been paid.
    - In the Mongo drivers, the default connection pools 100 connections large, which should be fine for most average applications.
- Following are the benefits of connection pooling:
    - New operations can be serviced with pre-existing connections, so a new connection doesn't have to be created each time.
    - A large influx of operations can be handled more quickly with a pool of existing connections.
- Following are wrong:
    - Multiple database clients can share a connection pool.
    - The connection pool will persist after the client is terminated.

## Lecture: Robust Client Configuration
- In this lesson, we're going to discuss ways in which you can make your application more robust with respect to how it communicates with the database.
- So you've learned about connection pooling all ready, but it's very important so we're going to briefly cover it again.
- Creating a new Mongo client for every request of the database might service your application in the short term, but it will eventually result in the application consuming and depleting available resources that become more and more scarce over time.
- Connection pooling reduces the total overhead associated with creating these new connections by allowing the application to recycle and reuse database connections for new requests.
- The M220 API you've been given correctly reuses the same class or object for all client communication, if you'd like to look at an example of how we did it.
- Another way to make a more robust database client is with the **write timeout** or **wtimeout**.
- No matter how well we engineer a system, we should always expect application external resources like queues, networks, and databases to take more time than expected.
- For an application or consumer critical operations, a developer may choose to write with `{ w: majority }` to ensure that acknowledged rights are written to a majority of nodes in the set.
- But if there's a problem on one of the secondary nodes, we might not get an acknowledgment back from the server for a while.
- If more writes than reads are coming into the system, and operations aren't being acknowledged, this will eventually lead to system gridlock.
- But we can avoid this gridlock by using a `wtimeout`.
- For any write operation written with w majority, always specify a write timeout.
- The specific length of a timeout will need to be determined based on your network and hardware, but you should always be setting timeouts on these sorts of writes.
- This `wtimeout` value is determined in milliseconds.
-   `{ w: "majority", wtimeout: 5000 }`
- So this would wait for 5 seconds before timing out on a `{ w: majority}` operation.
- And lastly, always handle the `serverSelectionTimeout` error.
- No ifs, ands, or buts about it.
- By handling this error, you also passively monitor the health of your application stack, and also become quickly aware of any hardware or software problems that haven't recovered in an adequate amount of time.
- If one of these servers goes down, the response we get back might let us know what happened.
- By default, the driver's going to wait 30 seconds before raising a server selection timeout error, but you could change this to suit your application's needs.
- By handling ths error, we also passively monitor the health of the application stack, and become quickly aware of any hardware and software problems that haven't been recovered in an adequate amount of time.
- Each driver and programming language has a specific way of dealing with errors, and we do handle this error in particular in the mflix application.
- So just to recap here:
    - Always use connection pooling, which, by default, will allow a connection pool of 100 connections.
    - Always specify a `wtimeout` for majority writes to make sure that the server isn't waiting for too long.
    - And always handles `serverSelectionTimeout` errors. This will make sure that the application becomes quickly aware of any hardware and software problems that haven't recovered in time.
- When should you set a `wtimeout`?
    - When our application is using a Write Concern more durable than `{ w: 1 }`.

## Lecture: Writes with Error Handling
- So in this lesson, we're going to encounter some of the basic errors in the PyMongo driver and how to handle these errors in a way that makes their application more consistent and reliable.
- So here I'm connecting to Atlas like we have before.
-   ```
    from pymongo import MongoClient, errors
    uri = "mongodb+srv://<username>:<password>@<host>/<database>"
    mc = MongoClient(uri)
    lessons = mc.lessons
    shipments = lessons.shipments
    ```
- We're using a new collection called shipments.
- So the scenario for this lesson is that our application is a clothing manufacturer that also handles the shipping of their new clothing items.
- This shipments collection will hold a document for each shipment. And we'll take a look at those documents in a minute.
- So this is a short script that's going to create some test data for the clothing manufacturer.
-   ```
    import time
    import random
    from pprint import pprint

    shipments.drop()

    cities = [ "Atlanta", "New York", "Miami", "Chicago", "Los Angeles", "Seattle", "Dallas" ]
    products = [ "shoes", "pants", "shirts", "hats", "socks" ]
    quantities = [ 10, 20, 40, 80, 160, 320, 640, 1280, 2560 ]
    docs = []

    for truck_id in range(30):
        source = random.choice(cities)
        destination = random.choice([c for c in cities if c != source])
        product = random.choice(products)
        quantity = random.choice(quantities)
        
        doc = {
            "truck_id": truck_id,
            "source": source,
            "destination": destination,
            "product": product,
            "quantity": quantity
        }
        
        docs.append(doc)
    ```
- This bit's included in the notebook itself, so you can test it out.
- So you can see the documents that we're producing have five fields each with this truck ID determined by the iteration of our loop.
- So this source and destination fields are strings that are derived from this cities array.
- But here, we're just making sure that the destination city is, in fact, different from the source city.
- Each shipment also has a product and a quantity, but the part we're going to focus on is this truck ID field.
- This is going to record the truck currently allocated for this shipment, so that the truck can be considered unavailable for any other shipments.
- This way when a new shipment comes in, we can make sure that truck that gets assigned to that shipment isn't already doing one.
- Just going to run this script and just verify what our data looks like.
-   ```
    insert_response = shipments.insert_many(docs)
    shipments.count_documents({})
    ```
- And it looks like that we successfully inserted 30 documents into our collection.
- So if we take a look at one of these documents, we can see that they do, in fact, have the five fields that we were expecting, plus an `_id` that Mongo added for us.
-   ```
    shipments.find_one()

    {
        '_id': ObjectId('548asd8wdas5d8asd')
        'truck_id': 0,
        'source': 'Los Angeles',
        'destination': 'Atlanta',
        'product': 'pants',
        'quantity': 10
    }
    ```
- The assumption I'm making for this data is that while this document exists in the collection, the shipment is still ongoing. So when this shipment is completed, we would delete this document from the collection.
- So that means that before, when we verified there were 30 documents in the shipments collection, that means that we also verified that 30 shipments are ongoing right now.
- So if we try to insert a new shipment, it would have to have a unique truck ID. This way each truck has only assigned to one shipment at a time.
- So this is the way that we're going to enforce uniqueness among the truck IDs in this collection.
-   ```
    shipments.create_index("truck_id", unique=True)
    ```
- So this is called a `unique` index, which will create an index on the truck_id field, and also make sure that there are no duplicate truck_ids.
- And it created this index called truck_id_1, the one meeting that the index is sorted in ascending order.
- So here's an example of a shipment being added to the shipments collection.
-   ```
    doc = {
        "source": "New York",
        "destination": "Atlanta",
        "truck_id": 4,
        "product": "socks",
        "quantity": 40
    }

    try:
        res = shipments.insert_one(doc)
        print(res.inserted_id)
    except errors.DuplicateKeyError:
        truck_id = doc["truck_id"]
        print(f"Truck #{truck_id} is currently performing a shipment. Please select another truck.")
    ```
- We want to ship 40 socks from New York to Atlanta. And we've assigned truck_id 4 to this shipment.
- This truck number could have been user input or something determined by our application. But either way, this is going to cause a duplicate key error, because we already have a shipment assigned to truck 4.
- So using the try accept block, our program prints out a message when the duplicate key error is thrown.
- This message tells us that the truck we wanted to send out, truck 4, has already been sent out for another shipment.
- So the application allows the insert to fail and then sends an error message up to the user to choose another truck.
- But we can actually be a little more proactive about handling this error if we know about the other trucks that are available for the job.
-   ```
    import string

    trucks = lessons.trucks
    trucks.drop()

    trucks.insert_many([
        { "_id": i, "license": "".join(random.choice(string.ascii_uppercase + string.digits) for _ in range(7)) } for i in range(50)
    ])
    trucks.count_documents({})
    ```
- So here's a new collection called trucks, which we're going to use to find another available truck.
- This should insert about 50 documents into the collection. It's a little hard to see, but I am actually iterating 50 times, and it looks like the count documents method gave us 50 back.
- So the documents in the trucks collection only have two fields.
-   ```
    trucks.find_one()
    {'_id': 0, 'license': '76I7H0O' }
    ```
- And underscore ID from 0 to 49, which will relate to the truck ID from the shipments collection. And I've assigned a random string of seven uppercase letters and numbers to be the license plate number. Although, some US states actually only allow six characters.
- So here's a similar block to the one we saw before, except we're handling the error in a little more of a proactive way.
-   ```
    doc = {
        "source": "New York",
        "destination": "Atlanta",
        "truck_id": 4,
        "product": "socks",
        "quantity": 40
    }

    try:
        res = shipments.insert_one(doc)
        print(res.inserted_id)
    except errors.DuplicateKeyError:
        busy_trucks = set(shipments.distinct("truck_id"))
        all_trucks = set(trucks.distinct("_id"))
        available_trucks = all_trucks.difference(busy_trucks)
        old_truck_id = doc["truck_id"]
        if available_trucks:
            chosen_truck = random.choice(list(available_trucks))
            new_truck_id = doc["truck_id"] = chosen_truck
            res = shipments.insert_one(doc)
            print(f"Truck #{old_truck_id} is currently performing a shipment. Truck #{new_truck_id} has been sent out instead.")
        else:
            print(f"Truck #{old_truck_id} is currently performing a shipment. Could not find another truck.")
    ```
- Instead of just servicing an error to the user, the application actually chooses a new truck, sends out that truck, and then alerts the user that the action was performed just by a different vehicle.
- So in this case, we tried to send truck number 4 out, but it was not available. So we chose another truck and then sent that truck out.
- But the way we chose this truck was actually very careful.
- We pulled all the distinct truck IDs from the shipments collection into this busy truck set, which basically just tells us all the trucks that are currently performing a shipment.
- We then pulled all the distinct `_id`s out from the truck's collection, which actually just gives us all the trucks that we have in circulation.
- The difference of these two sets will give us the available trucks.
- So this check to see which trucks are available is actually somewhat expensive, as these two distinct queries require two database round trips.
- Because of this, the application takes a pretty lazy approach here, assigning trucks to shipments.
- So it won't actually do any round trips until the truck that it tries to send out results in a duplicate key error.
- This might be suitable if the collisions won't occur very often, which is to say the trucks are usually available when we try to request them.
- But on the rare occasions they are not available, we do a little extra work and then send out a truck that we know, for a fact, will be available.
- So in this lesson:
    - We demonstrated how to handle a very specific duplicate key error. And it's important to remember that this error typically occurs on the `_id`, which is unique by default.
    - But it also pertains to fields that are contained in a unique index, like the index we had on truck ID.
    - Really, when handling these errors, we want to think about how much we can do after receiving the error.
    - If there's nothing we can do in response, if this error is truly fatal, then we should just return it to the user.
    - But if we can do something, as was the case with the shipments collection, we should try to handle the error in a more flexible way.
    - In this example, the error was that the truck we tried to reserve was already in use. But at that time, the program actually had the resources to determine which trucks were still available. So it sent down one of those trucks instead.

## Lecture: Principle of Least Privilege
- So in this lesson we're going to talk about the principle of least privilege and how we can apply it in our own security practices.
- So here's a short relevant quote from Jerry Seltzer, who is a domain expert within security and distributed networks.
- And essentially what the quote says is that all programs and users on a system should only have the privileges that are necessary to complete their intended purposes.
- We'll see what he means by that in a second.
- So to a certain extent at the application layer, we are already kind of doing this for our mFlix users.
- For example, we make sure that only certain resources and privileges are available to users who have been logged in. And even those users have different permissions from each other.
- For example, a user only has the permission once they're logged in to delete their own comments and no one else's.
- So MongoDB actually offers the same sort of robust user management at the database level.
- So by creating a database user specifically for the application, we can in a more granular way select the privileges and resources that mFlix should have access to.
- So this kind of forces us to ask the question, should the application be able to create indexes or create new collections, or should the application be able to drop an entire database?
- Questions like these tend to require some foresight about what actions the applications should be able to perform so they won't always be easy to answer.
- But they are very important in order to prevent the application from accessing a resource that it should never need.
- If the application has permission to use an important collection that it's not programmed to ever use, than that permission exists only as a vulnerability in our application, and we should remove it.
- So that's all for now about security.
- We highly recommend that if you're interested, you should take our MongoDB security course to learn more about securing your MongoDB deployments in production.
- So just to recap:
    - Make sure to engineer your systems with the principle of least privilege in mind.
    - In order to do this, we have to first consider what kinds of users we'll have on our system and what kind of permissions they'll need.
    - This includes application users who will be using the application itself and database users who will connect to and apply operations against the database.

## Lecture: Change Streams
- In this lesson, we're going to use change streams to track real time data changes to the data that our application is using.
- So as of MongoDB 3.6, change streams report changes at the collection level. So we open a change stream against this specific collection.
- But by default, it will return any change to the data in the collection regardless of what that change is. So we can also pass a pipeline to transform the change events we get back from the stream.
- So to demonstrate how important change dreams are, we're going to use an example supermarket application that wants to track the quantity of each item that it has in stock.
- So here I'm just initializing my Mongo client with a uri string.
-   ```
    from pymongo import MongoClient, errors
    uri = "mongodb+srv://<username>:<password>@<host>/<database>"
    client = MongoClient(uri)
    ```
- So I'm going to use a new collection for this lesson called Inventory.
-   ```
    lessons = client.lessons
    inventory = lessons.inventory
    inventory.drop()

    fruits = [ "strawberries", "bananas", "apples" ]
    for fruit in fruits:
        inventory.insert_one( { "type": fruit, "quantity": 100 } )
        
    list(inventory.find())
    ```
- I'm just dropping the collection before we get started.
- If you imagine we have a store that sells fruits, this collection would store the total quantities of every fruit that we have in stock.
- In this case, we have a very small store that only shows three types of fruits. And we've just updated the inventory to reflect that we got a shipment of 100 quantity of each.
- Now I'm just going to verify that our collection looks the way we expect.
- So I just ran a quick find command against the inventory collection to see what's in there. And it looks like we have a hundred of each fruit right now.
- But people are going to start buying them, because, you know, people like fruit. And we want to make sure that we don't run out.
- So I'm going to open a change stream against the inventory collection and track the data changes in real time.
-   ```
    try:
        with inventory.watch(full_document='updateLookup') as change_stream_cursor:
            for data_change in change_stream_cursor:
                print(data_change)
    except pymongo.errors.PyMongoError:
        print('Change stream closed because of an error.')
    ```
- So here I'm opening a change stream against the inventory collection using this watch method.
- Watch is going to return a cursor back to us so I can iterate through it in Python.
- We've also wrapped this whole thing in a try-except block, so that if something actually happens to the connection or something else, we'll know about it quickly.
- Just going to run this to open our change stream. And you can see the star here (jupyter notebook cell notation to indicate that the cell is running), because it's actually just waiting for changes to the collection.
- And until something goes wrong, it's just going to keep iterating through the empty cursor.
- So this is actually a completely different notebook. It's called Updates Every One Second. And you can find it attached to the lecture if you want to try it out.
- It's mainly just meant to mimic customers at the supermarket so that our change stream has changes to react to.
- Here I'm just initialising my client database and collection objects.
-   ```
    from pymongo import MongoClient, errors
    uri = "mongodb+srv://<username>:<password>@<host>/<database>"
    client = MongoClient(uri)
    lessons = client.lessons
    inventory = lessons.inventory
    ```
- So this part of the script is actually mimicking customers at the supermarket.
-   ```
    import time
    import random
    fruits = ["strawberries", "bananas", "apples"]
    quantities = [-1, -2, -4, -8]

    while True:
        random_fruit = random.choices(fruits)
        random_quantity = random.choice(quantities)
        inventory.update_one({"type": random_fruit, "quantity": { "$gt": 10 }})
        time.sleep(1)
    ```
- They're going to buy random fruits and random quantities. And these numbers in the quantities array are negative, because buying an item causes the quantity of that item to decrease.
- I've also asked the program to sleep for a second after every purchase so we can actually read the output that we get back from the change stream.
- Let's get that kicked off.
- If we go back to our change stream (line no. 280), we can see that the change stream is actually capturing these change events and printing them out to us.
- We can see that the document _id that we updated, the field that we updated, and what it was after we updated it, any fields that we might have removed.
- But the change stream cursor is just spitting out anything that gets with no filter. Any change to the inventory collection will appear in this output.
- But really this is noise. We don't care when the quantity drops to 99. We only want to know when it's close to zero so we can actually do something about it.
- So let's say we want to know if any of our quantities dips below 20 units.
- Here I've defined a pipeline for the change event documents to go through before they're returned by the cursor.
-   ```
    low_quantity_pipeline = [ { "$match": { "fullDocument.quantity": { "$lt": 20 } } } ]

    try:
        with inventory.watch(pipeline=low_quantity_pipeline, full_document='updateLookup') as change_stream_cursor:
            for data_change in change_stream_cursor:
                current_quantity = data_change["fullDocument"].get("quantity")
                fruit = data_change["fullDocument"].get("type")
                msg = "There are only {0} units left of {1}!".format(current_quantity, fruit)
                print(msg)
    except pymongo.errors.PyMongoError:
        logging.error('Change stream closed because of an error.')
    ```
- And in this case, if the cursor returns a change event after it went through this pipeline, it's because that change event caused one of our quantities to fall below 20 units.
- I'm just going to open this change stream.
- And so now we're back to Updates Every One Second to mimic the customers at the supermarket.
-   ```
    inventory.drop()
    fruits = ["strawberries", "bananas", "apples"]
    for fruit in fruits:
        inventory.insert_one({"type": fruit, "quantity": 100 })
    while True:
        random_fruit = random.choice(fruits)
        random_quantity = random.choice(quantities)
        inventory.update_one({"type": random_fruit, "quantity": {"$gt": 10} })
        time.sleep(.1)
    ```
- I've dropped the inventory collection and then repopulated it with 100 units of each fruit.
- And this loop is the same one as before, except I'm only waiting a tenth of a second after each update - These customers are just a little faster than the other ones :p
- If I just start this loop, then we can see that it's figuring out with the pipeline (code snippet at line no. 325) which updates caused the quantities to fall below 20 units and then reporting those changes back to us.
- These print statements have just been formatted by this message up here.
- So just to recap:
    - Change streams are a great way to track changes to the data in a collection.
    - If you're using MongoDB 4.0, you can actually open a change stream against the whole database and even a whole cluster.
    - We also have the flexibility to pass an aggregation pipeline to the change stream to transform or maybe filter out some of the change event documents.
- Following are true about Change Streams in PyMongo:
    - They output cursors, which contain change event documents.
    - They accept pipelines, which can be used to filter output from the change stream.
    - They can be used to log changes to a MongoDB collection.