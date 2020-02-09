# Step 1:
## Collect functional requirement 
- Use registration + login + profile
- Search
- Recommendation
- Payment
- Content ingestion and delivery
    - Rights and license
    - Categorizations
- Notifications
- Trending movies
- Watch history

## Collect design constraints
# Step 2:
## Bucketize functional requirements into micro services
Building blocks for each
- Use registration + login + profile
- Search
- Recommendation
    - User activity service
    - Recommendation generator
        - Take snapshot of data from expresso
        - Put data into HDFS
        - Run Map Reducer
    - Recommendation dashboard
- Payment
- Content ingestion and delivery
    - Rights and license
    - Categorizations
- Notifications
- Trending movies
- Watch history
## Get clarity weather problem is breadth-oriented or depth-oriented
breadth-oriented

# Step 3:
## Draw a logical diagram
## Draw and explain data/logic flow between them

# Step 4:
## Deep dive on each micro services at a time
## Content Ingestion and delivery service
- App tier
- Cache for content metadata
- Storage for both data and metadata

Data structure 
- Metadata
- K: content id, v: json object for metadata
- Data
- k: content id v: byte stream
How to store metdata?
- Hashmap in memory
-  Row oriented in storage
How to store data?
- File system
API?
- create(K,V) for ingestion
- read(K, offset)

## Content micro service
- Need to scale for storage? Yes
    - A*B A=number of K-Vs B= size of K-V
    - Metadata: 20,000 * 10KB = 200MB storage is trivial 
    - Data: 20,000 * 10GB = 200TB
- Need to scale for throughput? Yes
    - Number of create(K,V) = negligible
    - read(K, offset) : 3MB/s * 10 millions users = 30 TB/sec
        - A single server delivers around 1-2GB/s (SSD) and 300MB/s for spanning disk
    - Metadata read is negligible
- Need to scale for API parallelism?
Not relevant
- Availability?
Yes
- Geo location based distributions?
Yes
- Shard?
    - yes, vertically shard
    - Keep movie offset
    
