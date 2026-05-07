# Social Network - System Design

Social Network is a VK-like social network for users from the CIS region. The service allows users to manage profiles, friends, posts, media, feeds, private messages, group chats, and notifications.

### Functional requirements:

* view user profile
* add and remove friends
* view friend list
* create and view posts
* upload media files for posts
* view home feed
* view posts of a specific user
* view dialogs and group chats
* send and read text messages
* track message read status
* notify users about new messages
* track user online / offline status

### Non-functional requirements:

* 70 000 000 DAU
* 100 000 000 MAU
* availability 99.95%
* maximum message size is 1000 characters
* maximum chat size is 1000 users
* system is focused on users from the CIS region
* on average, each user sends 20 messages per day
* on average, each user creates 0.1 posts per day
* on average, each user views feed 20 times per day
* messages are stored for 5 years

## Design overview

For system design I have used [C4 model](https://c4model.com/). The design does not go below the second C4 level:

* Level 1. System context diagram
* Level 2. Container diagram

Level 1 diagram is described with PlantUML and can be opened in [PlantUML Online Editor](https://www.plantuml.com/plantuml/uml/) without installing anything locally.

Level 1. System context diagram: [`architecture/context.puml`](architecture/context.puml)

```plantuml
@startuml SocialNetworkSystemContext
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Context.puml

title Social Network - System Context

Person(user, "User", "A social network user from the CIS region")
Person(admin, "Admin", "Operates support, moderation, and incident handling")

System(social_network, "Social Network", "Allows users to manage profiles, friends, posts, feeds, media, private messages, group chats, and notifications")

System_Ext(push_provider, "Push Notification Provider", "Delivers notifications about new messages and user activity")
System_Ext(media_cdn, "Media CDN", "Distributes uploaded photos, audio, and video to users")

Rel(user, social_network, "Uses profiles, friends, posts, feeds, media, and messages", "HTTPS")
Rel(admin, social_network, "Manages support and moderation workflows", "HTTPS")
Rel(social_network, push_provider, "Sends push notifications", "HTTPS")
Rel(social_network, media_cdn, "Publishes and serves media files", "HTTPS")

SHOW_LEGEND()
@enduml
```

Level 2. Container diagram:

[`architecture/container.puml`](architecture/container.puml)

```plantuml
@startuml SocialNetworkContainer
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

title Social Network - Container Diagram

Person(user, "User", "A social network user from the CIS region")

System_Boundary(social_network, "Social Network") {
    Container(api_gateway, "API Gateway + LB", "Gateway", "Routes requests, validates auth tokens, applies rate limits")

    Container(social_backend, "Social Backend", "Backend service", "Profiles, friends, posts, comments, likes, counters")
    Container(feed_service, "Feed Service", "Backend service", "Builds and returns home feeds")
    Container(messaging_service, "Messaging Service", "Backend service", "Private messages, group chats, read status")
    Container(media_service, "Media Service", "Backend service", "Uploads and validates media files")
    Container(notification_worker, "Notification Worker", "Worker", "Consumes events and sends push notifications")

    ContainerDb(main_db, "Main DB", "PostgreSQL", "users, relationships, posts, comments, likes, media metadata")
    ContainerDb(redis, "Redis Cluster", "Redis", "hot profiles, hot posts, friendship checks, counters, rate limits")
    ContainerDb(feed_store, "Feed Store", "Redis", "user_feed and home_feed post id lists")
    ContainerDb(message_db, "Message DB", "NoSQL", "messages, chats, chat_members, read_status; sharded by chat_id")
    ContainerDb(blob_storage, "Blob Storage", "S3-compatible storage", "media files")
    ContainerQueue(kafka, "Kafka", "Event bus", "post_created, comment_created, message_sent and notification events")
}

System_Ext(cdn, "CDN", "Delivers media files to users")
System_Ext(push_provider, "Push Provider", "Delivers push notifications")

Rel(user, api_gateway, "Uses social network API", "HTTPS")
Rel(user, cdn, "Downloads media files", "HTTPS")

Rel(api_gateway, social_backend, "Reads/writes profiles, friends, posts, comments, likes", "HTTPS")
Rel(api_gateway, feed_service, "Gets home feed", "HTTPS")
Rel(api_gateway, messaging_service, "Sends and reads messages", "HTTPS")
Rel(api_gateway, media_service, "Uploads media files", "HTTPS")

Rel(social_backend, main_db, "Reads/writes source of truth")
Rel(social_backend, redis, "Reads/writes hot data")
Rel(social_backend, kafka, "Publishes domain events")

Rel(feed_service, feed_store, "Reads/writes user feeds")
Rel(kafka, feed_service, "Delivers feed events")

Rel(messaging_service, message_db, "Reads/writes messages and chats")
Rel(messaging_service, kafka, "Publishes message events")
Rel(kafka, messaging_service, "Delivers async message events")

Rel(media_service, blob_storage, "Stores media files")
Rel(blob_storage, cdn, "Serves media through")

Rel(kafka, notification_worker, "Delivers notification events")
Rel(notification_worker, push_provider, "Sends push notifications")

SHOW_LEGEND()
@enduml
```

## API

REST API is described with OpenAPI 3.0.3: [`api/rest_api.yml`](api/rest_api.yml).

## Database

Database schema is described with DBML: [`database/schema.dbml`](database/schema.dbml).

## Basic calculations

RPS (create post):

    DAU = 70 000 000
    Each user creates 0.1 posts per day
    Posts per day = 70 000 000 * 0.1 = 7 000 000
    RPS = 7 000 000 / 86 400 ~= 82

RPS (read feed):

    DAU = 70 000 000
    Each user views feed 20 times per day
    Feed reads per day = 70 000 000 * 20 = 1 400 000 000
    RPS = 1 400 000 000 / 86 400 ~= 16 204

Incoming traffic (create post without media):

    Posts per day = 7 000 000
    Average create post request size = 2 KB
    Incoming traffic per day = 7 000 000 * 2 KB = 14 GB/day
    Incoming traffic per second = 14 000 MB / 86 400 ~= 0.162 MB/s

Messages storage for 5 years:

    DAU = 70 000 000
    Each user sends 20 messages per day
    Message size = 512 B
    Retention = 5 years
    Messages for 5 years = 70 000 000 * 20 * 365 * 5 = 2 555 000 000 000
    Raw storage = 2 555 000 000 000 * 512 B = 1308.16 TB
    Storage with indexes and overhead = 1308.16 TB * 2 = 2616.32 TB
    Physical storage with replication factor 2 = 2616.32 TB * 2 = 5232.64 TB

Required HDD count:

    Disk capacity = 16 TB
    Primary shards = ceil(2616.32 TB / 16 TB) = 164
    Replication factor = 2
    Total shard copies = 164 * 2 = 328
    Required HDD count = 328
