# Requirements and Goal of the system
We will be designing simple version of Twitter with following requirement details.
## Functional requirements
- User should be able to post new tweet
- A user should be able to follow other user
- A user should be able to mark tweet as favorite
- A service should be able to generate users' timeline consisting of top tweets from all the people user follows 
- Tweet can contain photos and videos
## Non functional requirements
- Service needs to be highly available
- Acceptable latency of the system is 200ms for timeline generation
- Consistency can take a hit in the interest of availability; if user doesn't see a tweet for while, it should be fine
## Not in scope
- Searching for tweets
- replying to tweets
- Trending topics - current hot topics/searches 
- Tagging other users 
- Tweet notifications 
- Who to follow? Suggestions?
- Moments 
# Capacity estimation and constraints 
- Total users: 1 billion
- Daily active users: 200 million
- On avg every users makes `1` tweet in two days
- Daily new tweets: 100 million
- On avg, each user follows `200` other users
# High level system design 
Following are proposed micro services
- User service to create users/authenticate users
- Create tweet service to create new tweets
- User follower service to follow other users
- tweet favorite service to mark favorite tweet
- user's timeline service
![](Twitter-System-Design.drawio)
# API Design
## Create user
```
POST /users?api_key=string
{
    name: string
    email: string
    dob: datetime    
}
``` 
Following will be schema to store user details
```
{
    userId: integer
    name: string
    email: string
    dob: datetime  
    createdAt: datetime
    updatedAt: datetime
    lastLogin: datetime  
}
```
## Create tweet
```
POST /tweets?api_key=string&authToken
{
    tweetData: string
    tweetLat: int
    tweetLong: int
    userLat: int
    userLong: int
}
```
## Following users service
```
POST /users/follow?api_key=string&authToken=string
{
    userId: string
}
```
Following will be schema to store user followers
```
{
    userId: int
    followUserId: int
    createdAt: int
    updatedAt: int
}
```
## Mark tweet as favorite
```
POST /tweets/favorite/<tweetId>?api_key=string&authToken=string
```
Following will be schema to store favorite tweet
```
{
    tweetId: int
    userId: int
    createdAt: datetime
    updatedAt: datetime
}
```
## User timeline 
```
GET /users/<userId>/timeline?api_key=string&since=datetime&count=int&excludeReplies=boolean
```
# Scale App Server
- Total users: 1 billion
- Daily active users: 200 million
    - Concurrent users: 20 % of active users ~ 400k
    - max concurrent connection per server: 500
    - Number of app servers: 800
- On avg every users makes `1` tweet in two days
- Daily new tweets: 100 million
    - Concurrent users: 20 % of active tweets ~ 200k
    - max concurrent connection per server: 500
    - Number of app servers: 400  
- On avg, each user follows `200` other users
    - Concurrent users: 80 % of active users ~ 1.6k
    - max concurrent connection per server: 500
    - Number of app servers: 32  
- On avg, each user like `10` other tweets
    - Concurrent users: 80 % of active tweets ~ 0.8k
    - max concurrent connection per server: 500
    - Number of app servers: 16
# Storage details
Following are proposed storage system
## NoSQL Storage for User
### Schema
```
{
    userId: integer (4 bytes)
    name: string (32 bytes)
    email: string (32 bytes)
    dob: datetime  (8 bytes)
    createdAt: datetime (8 bytes)
    updatedAt: datetime (8 bytes)
    lastLogin: datetime  (8 bytes)
}
```    
- Size of each document:  100 bytes 
    - NoSQL stores each key and data types as well
    - Index also use disk storage
    - Keep size as 1kb to estimate on higher side
- Total users: 1 billion
    - Total size: 1 billion * 1kb ~= 1Tb
    - In 5 years: 5TB
### Data Sharding
- Shard key: userId
- Limits per shard: 1TB
- Total shards: 5
- Keep 3 replicas in each shards to handle failover
- Do write on majority i.e. compromise on consistencies and use eventual consistencies. 
- I.e in CAP, use AP system 
## NoSQL Storage for Tweets
### Schema
```
{
    tweetData: string (140 chars ~ 280 bytes)
    tweetLat: int (4 bytes)
    tweetLong: int (4 bytes)
    userLat: int (4 bytes)
    userLong: int (4 bytes)
}
```
- Size of each document:  296 bytes 
    - NoSQL stores each key and data types as well
    - Index also use disk storage
    - Keep size as 1kb to estimate on higher side
- Total users: 1 billion
- Daily active users: 200 million
- On avg every users makes `1` tweet in two days
- Daily new tweets: 100 million
    - Daily tweets size: 100 million * 1kb ~ 100GB
    - Tweets size in 1 year: 36.50TB
    - Tweets size in 5 years: 182TB
- Media storage
    - 1 out of every 5 tweets will have image
    - 1 out of 10th tweets will have video
    - avg image size: 200kb
    - avg video size: 2mb
    - Total image size every day: 100m/5*200kb + 100m/10*2mb = 24TB
    - Total image size in 5 years: 120TB
### Data Sharding
- Shard key: tweetId
- Limits per shard: 1TB
- Total shards: 5
- Keep 3 replicas in each shards to handle failover
- Do write on majority i.e. compromise on consistencies and use eventual consistencies. 
- I.e in CAP, use AP system 
# User timeline service
## Feed generation
- retrieve  all userids given user follows
- Retrieve latest, most popular, and relevant tweets for followed userids. These are potential tweets can be shown to user
- Rank these tweets based on relevance to user. This represent user's current feed
- Store this feed in the cache and return top posts (say 20) to be rendered on user's timeline
- Frontend can make paginated api call to fetch next `20` tweets

