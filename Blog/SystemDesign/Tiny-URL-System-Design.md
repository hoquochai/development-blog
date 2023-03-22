# Tiny URL System Design

The TinyUrl system design are applications which generates a short url corresponding to a big url. The short url generated corresponding to a big url will take a user to the same webpage as the original url would. The tiny urls makes it easier for users to share or save the urls.

This article will focus on how we can design a system which can generate tinyUrl on a massive scale.

## Requirements

### Functional

* User should be able to generate a tiny url for a given url.
* User should be able to fetch the original url by giving the tiny url.
* User should be able to give a custom name for their tiny url. If not given a unique code should be generated which will be 6–8 characters long.
* User can provide a tiny url expiry time as an optional parameter while generating tiny url. If not provided a default expiration time should be used.

### Non Functional

* The System should be highly available.
* The Latency to fetch the original URL corresponding to a tiny url should be low(< 100ms).
* We are okay with eventual consistency since the tiny url generated is not used immediately after generation.
* Tiny urls generated should not be predictable.

## Capacity Estimation

* Let’s assume there are 10 million new tiny url generation requests per day.
* Let’s assume on an average for every 1 tiny url generated there are 10 original request per day calls.
* So, there will be a total of 10 * 10 = 100 million requests per day to fetch original url corresponding to a given tiny url.
* If we intend to store 1 tinyUrl for 5 years then total number of Urls we need to store are -
10*365*5 ~ 18 billion

## API’s required
Based on the requirements we will define our API’s that will be required. We will go with REST API’s as they are lightweight, flexible and easy to implement.

1. Create a Tiny URL
```
POST: /tiny
Request {
    originalUrl: String
    expiryTimeInMs: Long (Optional)
    customName: String (Optional)
}
Response {
    tinyUrl: String
}
```
Note: userId is not a part of request since userId’s are a part of session context.

2. Get the original url corresponding to a given tinyUrl.
```
GET: /url/{tinyUrl}
Response {
    originalUrl: String
}
```

## Database Entities
Now we have defined our API’s we should be good now to define our database schema. Couple of points to note down before we define our schema -

* We want our database to be highly available.
* We are okay with eventual consistency.
* We do not require any strict relations between Entities and neither we want transactional capabilities. So, we can go with NoSQL databases.
* Our database is read heavy.
* We need to store billions of data. So, we should look for horizontal scaling and partitioning of data with even distributed load on different partitions.

_Considering all the above points we can go with a Key Value based NoSQL database like DynamoDB._

**We need two tables**

1. User table to store the user details.
```
userId: String [HashKey]
name: String
emailId: String
creationDateInUtc: long
```
2. Url table to store the tinyUrl to originalUrl mapping details.
```
tinyUrl: String [HashKey]
originalUrl: String
creationDateInUtc: long
expirationDateInUtc: long
```

## Component Details
Let’s talk about each component in detail here —

### TinyUrl format 
As per our capacity estimation calculation we will require to store 18 billion tinyUrl records. If we choose to create tinyUrl by using [a-z, A-Z, 0–9] with 6 character long string then we can have 6²⁶~ 56 billion records. So, a 6 character long string will be sufficient for generating all tinyUrls. Once a tinyUrl expires we can reuse the tinyUrl for some other original url. Hence, limiting the number of storage.

### KeyGenerationService 
This service is responsible for providing a new Key(aka tinyUrl). We need to create a key which should be 6 character long. There can be multiple approaches which we can discuss below.
* **Random key generation on every request.**

One approach that we can think of for generating a tiny url is that on every request we use randomisation to generate a key and return it to tinyUrlService. The issues with this approach are- 1) There can be a duplicate key generation using randomisation and KeyGenerationService has to retry the request to generate a different key, this will increase the latency of the write request. 2) We cannot reuse the key’s once they are expired, so we need to update our tiny Code length more frequently to meet the traffic requirements.

* **Pre generate the keys and use the keys on every request.**

The other approach that we can think is to pre generate the keys and store them in a database. The database will contain the key and a flag corresponding to that key which will be used to determine if that key is already in use or not. For a new tinyUrl request the KeyGenerationService will fetch an unused key from keyDDB and return it to TinyUrlService. Here we need to maintain a high consistency so that same key is not returned for 2 different request. What we can do to solve this is, we can store some keys in KeyGenerationService as a cache on individual host. Each host can query the database and fetch few(let’s say 1000) keys and store them in cache and mark those keys as used in DDB. Keeping the keys in cache can be done via separate task scheduled at a KeyGenerationService level. Once a tinyUrl expires the entry in the database will marked as available by a separate executor(like an AWS lambda).

### KeyDDB 
Database to store the pre generated keys, it will contain the key and a parameter corresponding to a key that tells wether this key is already used or not.

### Lambda 
AWS lambda that will update the KeyDDB and mark the key as available whenever there is an existing tinyUrl expires. Lambda can listen to UrlDDB update events and update the KeyDDB.

### TinyUrlService 
This service is responsible for exposing API’s that we discussed above. The clients will directly call these API’s and this service will interact with KeyGenerationService and TinyUrl DDB and provide the functionality to end clients.

### RedisCache 
Since our system is read heavy which means that we will be fetching originalUrl by providing a tinyUrl. To improve the latency for our getOriginalUrl we can add a caching layer, this layer will store the tinyUrl to originalUrl data in the cache. Our TinyUrlService will first try to fetch the data from cache, if the data will be present in cache then it will return the response, if data is not present in cache then it will fetch the data from DDB.

### UrlDDB 
This is the database that will store the url data as per the schema we discussed above.

### LoadBalancer
Since, we want to make our system highly available we need to add a load balancer between our client and tinyUrlService. There will be multiple instances running tinyUrlService and the load balancer’s job is to distribute the traffic among different servers. This will ensure that the request does not go to any faulty server and load is distributed equally among different servers.

### Client 
The client here refers to the front end application or a 3rd party system which will be using our exposed API’s to provide a tinyUrl service to end users.

## High Level Design
After discussing all the main requirements and approaches here is the final High level design for our TinyUrl System.

![alt text](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*rgYQPzhDwiuLLnaAoV-sEw.png)

## More Detailed Discussions
We can further extend the design to add more features to it, like —

* Adding privilege based url access. The owner of a tinyUrl can specify which users can access the origin url.
* Analytics dashboard creation of the above system.
* Client based rate limiting of requests.

## Resources
https://nikhilgupta1.medium.com/tiny-url-system-design-846a66c7f9d3