# Chapter 0: Introduction and Setup

## Lecture: Welcome to M220P

-   Hello, and welcome to M220P, the MongoDB developer course for Python developers.
-   My name is Matt, and I'll be one of your instructors.
-   In this course, you'll learn how to use MongoDB Python driver to develop applications that meet the modern standards for performance and resilience.
-   We'll be jumping right into using MongoDB with Python, so it's recommended that you take our introductory level MongoDB course, or at least have two months experience using Mongo.
-   Basic familiarity with Python will also be required, just so you can get the most out of this course.
-   By joining the course, you'll actually become the newest member of an application development team, building a movie-browsing application called mflix, and your job will be implementing the database layer of an API for that application.
-   Each lab in this course is presented as a ticket, and for each ticket, you'll have to read the user story, make all the unit tests pass, and then use the UI to run integration tests that actually call the database access methods that you wrote.
-   In building the backend for this application, you'll learn about creating and sharing database connections, writing data with different levels of durability, and handling and recovering from errors.
-   But most importantly, you'll learn how to develop an application with MongoDB the right way.
-   The technology stack is composed of Python 3, MongoDB, Flask, and React.
-   The React component is pre-built and bundled for you, and the API layer is also fully implemented.
-   You'll be focusing on implementing the part of Python backend that talks to MongoDB.
-   So thanks so much for joining us and good luck on your path to becoming a MongoDB developer.

## Lecture: MongoDB URI

-   In this lesson, we are going to discuss the mongo db URI.
-   URI stands for **Uniform Resource Identifier**. It's going to look very similar to URL which you are probably very familiar by browsing the internet.
-   URIs are used to define the connection between the applications and mongo db instances. Applications can use URI strings to tell the driver the hostname and port where mongo db is running.
-   The username and password used to login and any other options. Basically everything about a connection to mongo db can be defined in the URI string used to start the connection to database.
-   This is the standard URI string:
    -   `mongodb://username:password@cluster0-shard-00-00-m220-lessons-mcxlm.mongodb.net:27017,cluster0-shard-00-01-m220-lessons-mcxlm.mongodb.net:27017,cluster0-shard-00-02-m220-lessons-mcxlm.mongodb.net:27017/mflix`
-   In order to use this URI string, we have to be explicit about the hostnames of each of the servers in our cluster and if the hostname of one of the servers changes, we have to update the URI string to account for it.
-   In many courses, we'll be using a new kind of URI string what we call an **SRV string**.
-   The SRV string is like below:
    -   `mongodb+srv://<username>:<password>@<host>/<database>`
-   The SRT string begins with the prefix `mongodb+srv://` that tells mongo db that it's the address of the SRV record. We'll talk about that in a minute.
-   The next portion is composed of authentication credentials. We separate the username & password with thr colon `:`.
-   And then follow the password with the `@` sign. After the `@` sign, we specify the host which is the hostname of the SRV record for the mongo db cluster that we want to connect to.
-   Note that this hostname does not actually point to a database server. It only hosts the SRV records.
-   The host name is the address of the file called SRV records or service records. This service record will define its own DNS with a list of hostnames that we want to resolve to.
-   So, even though we are connecting to a cluster of servers, we don't need to know where each server in the cluster is because the SRV record keeps track of it for us.
-   This is specially useful when the servers in our cluster change or rotate out because the SRV record will be updated to reflect this and we don't need to chaneg anything in our client-side.
-   The last part of the connection string is the authentication database. This is the database in our cluster that contains the credentials of this user.
-   Essentially what we are saying is I want to connect as this user and the credentials to authenticate this user are stored in this database.
-   Here is an example of a URI string with SRV format:
    -   `mongodb+srv://USERNAME:PASSWORD@Mxxx-zzzz-mcxlm.mongodb.net/admin`
-   We've specified our USERNAME and PASSWORD. Then we have the host name (Mxxx-zzzz-mcxlm.mongodb.net) of the SRV record. Thne the databse that we want to authenticate against (admin)/
-   If we want to specify any other option for the connection, we make with the same SRV string and specify them at the end after a question mark `?`
-   Here's an example: `mongodb+srv://USERNAME:PASSWORD@Mxxx-zzzz-mcxlm.mongodb.net/admin?retryWrites=true`
-   This option will retry by default if the connection error arises. So we set it to `retryWrites=true`
-   To summarize:
    -   We covered the structure of the URI string
    -   We briefly doscussed the reason to use SRV records.
    -   The URI string is required to connect your application to mongodb. So, make sure your string is correctly formatted.

## Lecture: Setting up Atlas

-   In this lesson, we'll walk through setting up a free Atlas account, connecting to a new free-tier mongo db cluster, and importing the sample data that we'll use throughout this course.
-   We're using Atlas because it is the easiest path setting up a mongo db data store.
-   While you can complete this course using a local installation of mongo db, we assume Atlas is being used throughout.
-   Lets jump in.
-   The first step is to create a ne wAtlas account. If you already have an Atlas account, you don't need to create new one. Just login and a new cluster will be added to your existing account.
-   Otherwise, open your browser and navigate to mongodb.com/atlas. On the atlas homepage, click the **Start free** button.
-   On the next screen, enter your details, create the password and click on **Get started free**. A new account is now created and you are redirected to another page where you have to select the type of cluster you want to create. Select the free starters clusters option by **Create a cluster** button in that section
-   On the next page, you are asked to choose how you want to setup the cluster. Since we're deploying a free tier cluster, we can go with the default options.
-   The only change we should make here is to change the cluster name to **mflix**. Scroll down and type in **mflix** in cluster name section.
-   I'm confirming that I'm still setting up a free tier and then I'm going to click **create cluster**.
-   As you can see on the next screen, our new atlas cluster is being built. When it is ready, we'll import the sample data.
-   The setup is complete. Let's click on the 3-dots button under the cluster name and choose the option to load the smple dataset.
-   We'll click **load sample dataset** to confirm. This loads a number of sample datasets into your cluster including the sample_mflix dataset that we'll use throughout the course.
-   And that's it! We now have a fully functional database cluster with all of our dample data. Now it's time to connecto to our new cluster.
-   We'll click the **Connect** button and are presented with the pop-up which tells us that we can't connect until we configure security.
-   Atlas requires a baseline level of security for all clusters. To start, we can restrict access to one or more IP addresses.
-   I want to take a moment to discuss this: In a production environment we should limit which computers can access the data. For example, your backend servers maybe within a fixed range of IP addresses and you can specify that here.
-   Likewise for this course if you are planning on using your computer in only one location, you can just use your current IP address.
-   Now I tend to work from many locations so, I don't want to limit the IP address range and then wonder later why my code is not working. So I'm gonna click **add a different IP address** and then I'm going to enter the CIDR address that covers all IP addresses which is 0.0.0.0/0.
-   A **CIDR** address just defines a range of IP addresses and a subnet mask and I'll click **Add IP address**.
-   Remember that this is not a best practice for production data.
-   Now we'll create our first mongo db user. The username and password for this course will be `m220student` and `m220password` respectively.
-   Click on **Create mongo db user** and now we have our credentials we have ready to connect to our cluster.
-   While we are here, let's install compass for this course so that we have a nice GUI for exploring the movie dataset. Click **Choose a connection method** and then click **Connect with mongo db compass**.
-   Choose **I do nit have compass**, select your OS and click **Download compass**.
-   The installer downloads and we can run it. Depending on which OS you are using, the steps will be different but just follow the instructions.
-   I'm installing on Windows 10. While compass installs, let's switch back to Atlas and click the copy button next to connection string.
-   Now when you open compass, you can just paste in the connection string and change the username and password which are `m220student` and `m220password` respectively.
-   Ok, compass is installed. So let's run it and paste the connection string with the password.
-   Before I click connect, we'll click the **favourite** button since we'll be using this connection again later, this favourite button will allow us to give it a name that makes sense to us and then we can save it.
-   Now, click **Connect** and a new page will open showing all of the sample data that Atlas imported for us. We can find the `sample_mflix` database and feel free to explore here and look at the data we'll be working with through out this course.
-   With this, our database is all setup and ready to move on to next lesson.

## Lecture: Overview

### Overview

-   We are going to have series of README instructions to be able to setup our MFLIX application successfully. With MFLIX application, you will learn to create and share a database connection, perform the basic Create, Read, Update, and Delete operations through the driver, handle errors, utilize the MongoDB best practices and more.
-   Mflix is composed of two main components:
    -   Frontend: All the UI functionality is already implemented for you, which includes the built-in React application that you do not need to worry about.
    -   Backend: The project that provides the necessary service to the application. The code flow is already implemented except some functions.
-   You'll only be implementing the functions which directly call to MongoDB.

### Database Layer

-   We will be using MongoDB Atlas, MongoDB's official Database as a Service (DBaaS), so you will not need to manage the database component yourself. However, you will still need to install MongoDB locally to access the command line tools that interact with Atlas, to load data into MongoDB and potentially do some exploration of your database with the shell.
-   The following README sections are here to get you setup for this course.

## Setting up the code

-   Just followed the steps in the lecture and I was good to go - covered 2 lessons.
