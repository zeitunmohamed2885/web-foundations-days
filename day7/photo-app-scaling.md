# Day 7 Assignment: Scaling SnapShare

## 1. Assumptions

SnapShare is a photo-sharing application where users upload photos and view photos from people they follow.

The following assumptions are used for the scaling estimates:

- Registered users: 10 million.
- Daily active users (DAU): 10% of registered users.
- Each daily active user uploads one photo per day.
- Each daily active user views 50 feed pages per day.
- Each original photo has an average size of 2 MB.
- Each photo produces one thumbnail of 50 KB.
- A day has 86,400 seconds, and a year has 365 days.
- Peak traffic is five times the average traffic.
- Storage calculations use decimal units: 1 TB = 1,000,000 MB.
- All uploads and thumbnails are retained for one year, with no deletion or compression beyond thumbnail generation.
- Feed views represent page requests, not individual image requests.
- Storage estimates exclude database metadata, backups, replication, and operational overhead.

### Daily active users

Daily active users are 10% of registered users.

DAU = 10,000,000 × 0.10

**Daily active users = 1,000,000**

## 2. Traffic and storage estimates

### Uploads per second

Each active user uploads one photo per day.

Daily uploads = 1,000,000 × 1 = 1,000,000 photos.

Average uploads per second:

1,000,000 ÷ 86,400 = 11.57

**Average upload rate: approximately 12 uploads per second.**

Assuming the same 5× peak factor, peak uploads are approximately 58 per second.

### Feed views per second

Each active user views 50 feed pages per day.

Daily feed views = 1,000,000 × 50 = 50,000,000 views.

Average feed views per second:

50,000,000 ÷ 86,400 = 578.70

**Average feed view rate: approximately 579 views per second.**

Peak feed views per second:

578.70 × 5 = 2,893.52

**Peak feed view rate: approximately 2,894 views per second.**

### Photo storage per year

Original photos:

1,000,000 photos/day × 2 MB × 365 days = 730,000,000 MB.

Original photo storage = **730 TB per year**.

Thumbnails:

1,000,000 thumbnails/day × 0.05 MB × 365 days = 18,250,000 MB.

Thumbnail storage = **18.25 TB per year**.

Total photo storage:

730 TB + 18.25 TB = **748.25 TB per year**.

This is the raw storage requirement for originals and thumbnails before replication, backups, and other overhead. Actual provisioned capacity must be higher.

## 3. Read-heavy or write-heavy?

SnapShare is a **read-heavy system**.

The application handles approximately 50 million feed page views per day compared with one million photo uploads per day. This represents about 50 feed views for every new photo uploaded.

The design should prioritize fast feed delivery through a CDN and caching, distribute database reads across a primary database and a read replica, and scale application servers horizontally. Upload processing should be asynchronous so thumbnail generation does not slow down the upload experience.

## 4. Why photos should not be stored inside the database

Photo files are large binary objects. Storing them directly in the database would increase database size, backup times, storage costs, and the workload involved in serving images.

Instead, original photos and generated thumbnails should be stored in **object storage**, such as Amazon S3 or an equivalent service. The database should store photo metadata and object keys or URLs, including the uploader, upload time, caption, and relationships to users.

A CDN can cache and deliver the stored images from locations close to users, reducing latency and repeated requests to the origin storage.

## 5. Architecture diagram

```text
                         +------------------+
                         |      Users       |
                         +--------+---------+
                                  |
                     +------------+------------+
                     |                         |
                     v                         v
              +-------------+           +-------------+
              |     CDN     |           |  Load       |
              | Image Cache |           |  Balancer   |
              +------+------+           +------+------+
                     |                         |
                     |                         v
                     |                 +---------------+
                     |                 | App Servers   |
                     |                 | (Horizontal   |
                     |                 | Scaling)      |
                     |                 +---+-------+---+
                     |                     |       |
                     |                     v       v
                     |              +----------+  +----------------+
                     |              |  Cache   |  | Message Queue  |
                     |              |          |  | Thumbnail Jobs |
                     |              +----+-----+  +--------+-------+
                     |                   |                 |
                     |                   v                 v
                     |           +---------------+  +----------------+
                     |           | Primary       |  | Thumbnail      |
                     |           | Database      |  | Worker         |
                     |           +-------+-------+  +--------+-------+
                     |                   |                   |
                     |                   v                   v
                     |           +---------------+   +---------------+
                     |           | Read Replica  |   | Object Storage|
                     |           | (Feed Reads)  |   | Originals and |
                     |           +---------------+   | Thumbnails    |
                     |                               +-------+-------+
                     |                                       |
                     +---------------------------------------+
                                Image Delivery
```

The diagram shows the main logical components. In a production implementation, the CDN should normally fetch uncached images from object storage or an origin endpoint, rather than routing every image request through the application servers. The load balancer handles application API requests.

## 6. What each component solves

1. **CDN:** Caches and delivers photos and thumbnails close to users, reducing latency and origin traffic.
2. **Load balancer:** Distributes incoming application requests across healthy app servers to avoid overloading a single server.
3. **App servers:** Handle authentication, uploads, feed generation, authorization, and other application business logic.
4. **Cache:** Stores frequently accessed feed data and metadata in memory to reduce repeated database queries.
5. **Primary database:** Stores authoritative structured data, such as users, follow relationships, photo metadata, and upload records.
6. **Read replica:** Serves eligible read queries so feed traffic does not place all its load on the primary database.
7. **Object storage:** Stores large original photos and thumbnails independently of the database.
8. **Message queue:** Holds thumbnail-generation jobs so uploads can complete without waiting for image processing.
9. **Thumbnail worker:** Consumes queued jobs, creates smaller thumbnail images, and stores them in object storage.

## 7. Photo upload flow

1. A user selects a photo and submits an upload request to the application.
2. The app server authenticates the user, checks authorization and upload limits, and validates the file type and size.
3. The application uploads the original photo to object storage, preferably through a short-lived, authorized pre-signed upload URL for large files.
4. After the original upload succeeds, the application records the photo metadata and object key in the primary database.
5. The application publishes a thumbnail-generation job to the message queue.
6. The application returns an upload confirmation to the user. The thumbnail may still be processing.
7. A thumbnail worker consumes the queued job and retrieves the original image from object storage.
8. The worker resizes the image, generates a thumbnail of approximately 50 KB, and saves it to object storage.
9. The worker updates the photo's processing status in the database or emits a completion event, making the thumbnail available to the feed.
10. When users view their feeds, app servers retrieve feed metadata from the cache or a database read replica, and image requests are served from the CDN when cached or from object storage through the CDN when not cached.

The queue and worker should support retries and idempotent processing so that temporary failures do not permanently lose thumbnail jobs or create inconsistent results.

## 8. Trade-offs

### Trade-off 1: Caching versus data freshness

Caching reduces database load and improves feed performance, but cached information can become stale. For example, a newly uploaded photo or a changed follow relationship might not appear immediately.

**Decision:** Use short cache expiration times or targeted cache invalidation for frequently changing feed data, balancing freshness against performance.

### Trade-off 2: Asynchronous thumbnail generation versus immediate availability

A queue allows uploads to complete quickly and absorbs bursts of thumbnail jobs. However, users may briefly see a placeholder while a thumbnail is being generated, and a failed worker can delay processing.

**Decision:** Accept a small delay in thumbnail availability, while monitoring queue depth and job failures and implementing retries and dead-letter handling.

### Trade-off 3: Read replicas versus consistency

Read replicas distribute read traffic and improve scalability, but replication lag can mean a recently uploaded photo is not immediately visible when a feed is read from a replica.

**Decision:** Use replicas for most feed reads, but route critical read-after-write requests to the primary database or use another consistency strategy.

### Trade-off 4: Object storage and CDN versus operational complexity

Object storage and a CDN are suitable for large photo collections and high-volume image delivery. However, they introduce extra configuration, access-control requirements, cache invalidation concerns, and separate storage and delivery costs.

**Decision:** Use private object storage with controlled access and CDN caching policies, and monitor bandwidth, storage growth, and cache-hit rates.

## 9. Conclusion

SnapShare has one million daily active users, approximately 12 average uploads per second, 579 average feed views per second, and a peak feed-view rate of approximately 2,894 per second under the 5× assumption. Original photos and thumbnails require about 748.25 TB of raw storage annually.

A CDN, horizontally scalable app servers, caching, a primary database with a read replica, object storage, and asynchronous thumbnail processing provide a scalable foundation for the application's read-heavy workload.
