# Requirements and Goals of the System
## Functional requirement 
- Users should be able to upload videos.
- Users should be able to share and view videos.
- Our services should be able to record stats of videos, e.g., likes/dislikes, total number of views, etc.
- Users should be able to add and view comments on videos.

## Design constraints
- Number of users: 1.5 billion
- Daily active users: 800 million
- No of videos watched by user daily: 5
- Upload to view ratio: 1:200
## Non-Functional Requirements:
- The system should be highly reliable, any video uploaded should not be lost.
- The system should be highly available. Consistency can take a hit (in the interest of availability); if a user doesn’t see a video for a while, it should be fine.
- Users should have a real time experience while watching videos and should not feel any lag.
## Not in scope
- Video recommendations
- most popular videos
- channels
- subscriptions
- watch later
- favorites

# System API/Micro services
## User service
```
POST /users?apiKey=string
{
    username: string
    email: string
    dob: datetime
    lat: int
    long: int   
    age: int
    address: string 
}
```
### User storage schema
```
{
    userId: int 4 bytes
    username: string 240 bytes
    email: string 240 bytes
    dob: datetime 8 bytes
    createdAt: datetime 8 bytes
    updatedAt: datetime 8 bytes
    lat: int 4 bytes
    long: int   4 bytes
    age: int 4 bytes
    address: string  240 bytes
}
```
### Storage estimate
- Size of one document: 760 bytes
- Size of all keys: 200 bytes
- index on userId: 4 bytes
- Size of one document: 760 + 200 + 4 = 964 bytes ~1kb
- Number of users: 1.5 billion
- Total size of user data: 1.5 billion * 1kb ~ 1.5tb
- Storage per server : 1TB
- Number of shards: 2
- Shard key : hash(userId)
- Number of replicas per shard: 3
- Total storage: 6TB
## Upload video
### API
```
POST /upload?apiKey=string&authToken=string multipart
-- multipart delimiter--------
title: string
description: string
tags: string[],
categoryId: int,
language: int,
details: string
-- multipart delimiter--------
```
- A successful upload will return HTTP 202 (request accepted)
- Once the video encoding is completed the user is notified through email with a link to access the video.
- We can also expose a queryable API to let users know the current status of their uploaded video.
### Video Metadata schema
Following will be schema for video metadata.
```
{
    videoId: int 4 bytes
    title: string 240 bytes
    description: string 240 bytes
    tags: string[], 100 bytes
    categoryId: int, 4 bytes
    language: int, 4 bytes
    details: string 240 bytes
    userId: int 4 bytes
    size: int 4 bytes
    url: string 240 bytes
    thumbnail: string 240 bytes
    likes: int 4 bytes
    dislikes: int 4 bytes
    views: int 4 bytes
    createdAt: datetime 8 bytes
    updatedAt: datetime 8 bytes
}
```
### Video metadata storage estimation
- size of one document: 1346 bytes
- size of keys = 16 * 20 bytes ~ 320 bytes
- index on videoId = 4 bytes 
- Total size for one document: 1346 + 320 + 4 bytes = 1670 bytes ~ 2kb
- Number of users: 1.5 billion
- Daily active users: 800 million
- No of videos watched by user daily: 5
- Upload to view ratio: 1:200
- Total no of videos watched daily: 800 * 5 million ~ 4 billion
- Number of videos uploaded by users daily: 4000 million / 200 ~ 20 million
- Daily size of video metadata: 20 million * 2kb = 40gb
- Size of video metadata in 5 years: 5 * 365 * 40GB = 73TB
- Storage limit per server: 1TB
- Number of shards : 73
- shard key: has(videoId)
- Number of replicas per shard: 3
- Total storage: 73*3 = 219TB
### Video storage estimation
- Number of videos uploaded by users daily: 20 million
- Avg size of video: 100MB
- Size of uploaded video per day: 20 million * 100MB ~ 2PB
- Total size of videos in 5 years: 365 * 24 * 2PB ~ 17520 PB
## Search/List videos Micro Service
```
GET /videos/search?apiKey=string&authToken=string&q=string&location=int&pageSize=int&start=int

Response: 
[
    pageInfo: {
        hasNext: boolean
        pageSize: int
    },
    data: [
        {
            videoId: int
            title: string
            description: string
            tags: string[]
            categoryId: int
            language: int
            details: string  
            views: int
            thumbnail: string
            url: string        
        }
    ]
]
```
## Stream Video to view
```
GET /videos/<videoId>/view?apiKey=string&authToken=string&offset=int&codec=int&resolution=int
```
- offset
    - We should be able to stream video from any offset; this offset would be a time in seconds from the beginning of the video. 
    - If we support playing/pausing a video from multiple devices, we will need to store the offset on the server. 
    - This will enable the users to start watching a video on any device from the same point where they left off.
- codec/resolution
    - We should send the codec and resolution info in the API from the client to support play/pause from multiple devices. 
    - Imagine you are watching a video on your TV’s Netflix app, paused it, and started watching it on your phone’s Netflix app. 
    - In this case, you would need codec and resolution, as both these devices have a different resolution and use a different codec.
## Video Comment
```
POST /videos/<videoId>/comments/<commentId>?apiKey=string&authToken=string
{
    videoId: int
    userId: int
    comment: string
}
```
### Comment Storage Schema
```
{
    commentId: int 4 bytes
    videoId: int 4 bytes
    userId: int 4 bytes
    comment: string 240 bytes
    createdAt: datetime 8 bytes
}
```
### Comment Storage estimate
- Size of one document: 260 bytes
- Size of keys: 5 * 20 bytes ~ 100 bytes
- Index on commentId and videoId and userId ~ 3 *4 bytes ~ 12 bytes
- Total size for one document: 260 + 100 + 12 bytes ~ 372 bytes ~ 500bytes
- Number of users: 1.5 billion
- Daily active users: 800 million
- No of videos watched by user daily: 5
- Upload to view ratio: 1:200
- Total no of videos watched daily: 800 * 5 million ~ 4 billion
- % of users makes comment daily: 1%
- Total number of comments per day: 40 million
- Total size per day: 40 million * 500 bytes ~ 20 gb
- Total size in 5 years: 365 * 24 * 20 gb ~ 175.2 tb
- Storage limit per server: 1TB
- Number of shards : 176
- shard key: has(commentId)
- Number of replicas per shard: 3
- Total storage: 176*3 = 528TB
## Video likes
```
POST /videos/<videoId>/likes?apiKey=string&authToken=string
```
This will update video metadata table.
## Video dislikes
```
POST /videos/<videoId>/dislikes?apiKey=string&authToken=string
```
This will update video metadata table.
# High level system design
![](https://docs.google.com/drawings/d/e/2PACX-1vTquLWMbUYrEMWyKDePEuBA1qePmkJ53e8S1fRfajUL8ofCUHUgf5C1Zm6YRQ9PO7fH4X75lHrC1l0n/pub?w=960&h=720)

# Video storing for streaming
## Video storage estimation
- Number of users: 1.5 billion
- Daily active users: 800 million
- No of videos watched by user daily: 5
- Upload to view ratio: 1:200
- Total no of videos watched daily: 800 * 5 million ~ 4 billion
- Number of videos uploaded by users daily: 4000 million / 200 ~ 20 million
- Avg size of video: 100MB
- Size of uploaded video per day: 20 million * 100MB ~ 2PB
- Total size of videos in 5 years: 365 * 24 * 2PB ~ 17520 PB
## Why we need to scale streaming?
- Daily active users: 800 million
- No of videos watched by user daily: 5
- Total no of videos watched daily: 800 * 5 million ~ 4 billion
- No of users watching video concurrently: 10% ~ 400 million
- Video download speed per user: 3MB per second
- Total video download speed : 3 MB/s * 400 million ~ 1200 TB/s
- Limit per single server to deliver data: 1-2GB/s (SSD) and 300MB/s for spinning disk
## Can we solve it by adding CDN?
- Limit per server: 1GB/s
- Storage limit per CDN: 1TB
- Number of CDN servers: `1200 TB/s` / `1GB/s` ~ 1,200,000 
## How to shard movies?
### Shard horizontally
- shard as per videoId
- If video is too hot then all request will be served by same shard
### Shard vertically
- Size of video is large
- Divide into vertically
    - First 120MB of a particular movie on one shard
    - Stream api will pass both movieId and offset
## How client/video player handle stream?
- Maintain ring buffer
- Preload video chunks by making stream api call
- Act as producer/consumer
    - consumption rate should be lower than the production rate
    - otherwise you will see the circle
    - seed for a frame consumption rate and delivery rate
    - Delivery rate should be higher
    - Keep downloading data at 3MB/sec
    - Showing data at much lesser rate
    - Queue is always full
    - Moment if you are not able to deliver at the desired rate then you will see bufferring
## Use hybrid approach
- We were able to reduce the hotspot
- We were able to distribute the read workload (bcz throughput was bottleneck)
- dividing by chunks helps
- Google has GFS
- Hadoop is open source of GFS
- GFS stores chunks of 64MB
- It allow randomly distributes contents across servers
- Read API have to  gives chunk id so that content can be read
- If movie is so hot then?
- Replication will help    
