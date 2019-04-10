# GoogleAnalytics

# Traffic estimates: 
There are over 1 Billion requests per day or approximately 11,500 requests per second.  
Let’s assume we have 50Million total users. (This is the total number of Google Analytics Users)


# Requirements:

The system needs to:
(1) Handle large write volume: Billions write events per day.

(2) Handle large read/query volume: Millions of merchants want to get insight about their business. Read/Query patterns are time-series related metrics.

(3) Provide metrics to customers with at most one hour delay.

(4) Run with minimum downtime.

(5) Have the ability to reprocess historical data in case of bugs in the processing logic.


# System APIs

### (1) Create an Event
We can have SOAP or REST APIs to expose the functionality of our service. The following could be the definition of the API for creating a new event:  
API - createEvent(api_dev_key, event_type, event_data, user_id)

### Parameters:
api_dev_key (string): The API developer key of a registered account. This will be used to, among other things, throttle users based on their allocated quota.

event_type (string): The type of event

event_data (string): The data of the event

user_id (int): The ID of the user.

Returns: (JSON)
A successful post will return the Success Response. Otherwise, an appropriate HTTP error is returned.


### (2) Read the Metric Data

We can have SOAP or REST APIs to expose the functionality of our service. The following could be the definition of the API for reading metrics:  
getMetric(api_dev_key, metric_type, user_id, start_date, end_date)



### Parameters:
api_dev_key (string): The API developer key of a registered account. This will be used to, among other things, throttle users based on their allocated quota.

metric_type (string): The type of metric to be fetched
user_id (int): The ID of the user. (OPTIONAL)

start_date (string): The Starting Date of the event

end_date (string): The Ending Date of the event

Returns: (JSON)
A JSON containing information about the metric a list of businesses matching the search query. 


# High-Level System Design

We need a system that can efficiently store all the new events, 1Billion/86400s => 11500 events per second and read 50Million/86400s => 600 read per second. It is clear from the requirements that this will be a write-heavy system.

At a high level, we need multiple application servers to serve all these requests with load balancers in front of them for traffic distributions. On the backend, we need an efficient database that can store all the new events and can support a huge number of writes.

At a high level this problem can be divided into two parts:

## Event generation Service: 
This Service will periodically analyze the events and create the metrics. We will perform the following steps:

Retrieve all the event's data
Apply the processing logic and create the metrics
Rank these metrics based on the relevance to User (OPTIONAL)
Store these metrics in the cache
On the front-end, when User wants to get insights about their business, it will be served the metrics directly from the cache only
One thing to notice here is that we generated the metrics once and stored it in the cache. For one type of event, this will be done every hour.



## Metrics publishing: 
Whenever Customer wants to get insights about their business. He/She will use the read metric API.  The data will be served the metrics directly from the cache only

At a high level, we will need the following components in our Analytics System:

#### Web servers: 
To maintain a connection with the user. This connection will be used to transfer data between the user and the server.  
###### Technology Used - Nginx    
&nbsp;

#### Application server: 
To execute the workflows of storing new events in the database servers. We will also need some application servers to retrieve and to push the metric to the end user. 
###### Technology Used - Spring Boot, Apache Tomcat  
&nbsp;
#### Metrics cache: 
To store the data about Metrics. Technology Used - Redis cache
 Database: To store metadata about User and Events. 
###### Technology Used - MySQL Database  
&nbsp;
#### Video and photo storage, and cache: 
Blob storage, to store all the media(If there is any).
###### Technology Used - Amazon S3  
&nbsp;
### Metric generation service: 
To gather and process all the relevant events for a user to generate metrics and store in the cache. 

Although our expected daily write load is 1 Billion and read load is 50 million requests. This traffic will be distributed unevenly throughout the day, though, at peak time we should expect at least 11500 write requests and around 600 read requests per second. We should keep this in mind while designing the architecture of our system.

# Database Design

There are three primary objects: User, Event (e.g. Clicks, Visit, etc.). Here are some observations about the relationships between these entities:

A User can be associated with multiple Events.
Each Event will have a UserID which will point to the User who created it. For simplicity, let’s assume that only users can create Events.
User(userId, firstname, lastname, email, phonenumber, creationDate)

Events(eventid, eventData, eventType, userId, creationDate, updationDate) 

Metric(metricId, metricType, metricValue, eventType, eventIdList, creationDate)
If we are using a relational database, we would need to relation: User-Events relation.  The “eventType” column in “Events” identifies the type of event that is being created.




# Database Sharding

Since we have a huge number of new events every day and our write load is extremely high too, we need to distribute our data onto multiple machines such that we can read/write it efficiently. For sharding our databases that are storing Events and metric data, which is being stored in memory, we can partition it based on UserID. We can try storing all the data of a user on one server. When storing, we can pass the UserID to our hash function that will map the user to a cache server where we will store the user’s Events and Metrics objects. To get the feed of a user, we would always have to query only one server. For future growth and replication, we must use Consistent Hashing. 


# Replication and Fault Tolerance

We can have multiple secondary database servers for each DB partition. Secondary servers will be used for read traffic only. All writes will first go to the primary server and then will be replicated to secondary servers. This scheme will also give us fault tolerance since whenever the primary server goes down we can failover to a secondary server. 

This will give us the ability to reprocess historical data in case of bugs in the processing logic
