# Notification System Design

# Stage 1 – REST API Design

## Core Notification Features

The notification system supports:

* Create notification
* Fetch notifications
* Fetch unread notifications
* Mark notification as read
* Mark all notifications as read
* Delete notification
* Bulk notifications
* Notification preferences
* Real-time notifications

## Common Request Headers

```http
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
Accept: application/json
```

## Common Response Headers

```http
Content-Type: application/json
X-Request-ID: <uuid>
```
## Notification Object

```json
{
  "id": "uuid",
  "userId": 1042,
  "type": "Placement",
  "title": "Placement Update",
  "message": "XYZ Corporation hiring",
  "isRead": false,
  "priority": 5,
  "createdAt": "2026-04-22T17:51:18Z"
}
```
## 1. Get Notifications

### Endpoint

```http
GET /api/v1/notifications
```

### Query Parameters

```http
?page=1
&limit=20
&type=Placement
&isRead=false
```

### Response

```json
{
  "data": [
    {
      "id": "123",
      "type": "Placement",
      "message": "XYZ Corporation hiring",
      "isRead": false,
      "createdAt": "2026-04-22T17:51:18Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100
  }
}
```
## 2. Get Unread Notifications

### Endpoint

```http
GET /api/v1/notifications/unread
```

### Response

```json
{
  "count": 12,
  "notifications": []
}
```
## 3. Create Notification

### Endpoint

```http
POST /api/v1/notifications
```

### Request

```json
{
  "userId": 1042,
  "type": "Result",
  "title": "Exam Result",
  "message": "Mid-sem result published"
}
```

### Response

```json
{
  "notificationId": "uuid"
}
```
## 4. Mark Notification Read

### Endpoint

```http
PATCH /api/v1/notifications/{notificationId}/read
```

### Response

```json
{
  "success": true
}
```
## 5. Mark All Notifications Read

### Endpoint

```http
PATCH /api/v1/notifications/read-all
```

### Response

```json
{
  "updated": 34
}
```
## 6. Delete Notification

### Endpoint

```http
DELETE /api/v1/notifications/{notificationId}
```

### Response

```json
{
  "success": true
}
```
## 7. Bulk Notifications

### Endpoint

```http
POST /api/v1/notifications/bulk
```

### Request

```json
{
  "userIds": [101,102,103],
  "type": "Event",
  "message": "Farewell Event"
}
```

### Response

```json
{
  "accepted": true,
  "jobId": "bulk-job-001"
}
```
## Notification Preferences

### Endpoints

```http
GET /api/v1/preferences
PUT /api/v1/preferences
```

### Example

```json
{
  "email": true,
  "push": true,
  "inApp": true
}
```

---

## Real-Time Notification Design

### Technology

WebSocket

### Endpoint

```http
wss://api.company.com/ws/notifications
```

### Authentication

```http
Authorization: Bearer <JWT_TOKEN>
```

### Server Event Example

```json
{
  "event": "notification.created",
  "data": {
    "id": "123",
    "type": "Placement",
    "message": "XYZ Corporation hiring"
  }
}
```

### Why WebSocket?

* Persistent connection
* Low latency
* Bidirectional communication
* Better than polling for real-time systems

---

# Stage 2 – Database Design

## Database Choice

### PostgreSQL

Reasons:

* ACID compliance
* Strong indexing support
* Partitioning support
* High reliability
* Mature ecosystem

---

## Database Schema

### Users Table

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255)
);
```

### Notifications Table

```sql
CREATE TABLE notifications (
    id UUID PRIMARY KEY,
    user_id BIGINT NOT NULL,
    notification_type VARCHAR(30) NOT NULL,
    title VARCHAR(255),
    message TEXT,
    priority SMALLINT DEFAULT 5,
    is_read BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,

    CONSTRAINT fk_user
        FOREIGN KEY(user_id)
        REFERENCES users(id)
);
```
## Indexes

```sql
CREATE INDEX idx_notifications_user
ON notifications(user_id);

CREATE INDEX idx_notifications_user_read
ON notifications(user_id,is_read);

CREATE INDEX idx_notifications_created
ON notifications(created_at DESC);

CREATE INDEX idx_notifications_type
ON notifications(notification_type);
```
## Scaling Challenges

### Challenge 1: Millions of Notifications

Problem:

* Large tables
* Slower queries

Solution:

* Table partitioning by date

```sql
PARTITION BY RANGE(created_at);
```
### Challenge 2: High Read Traffic

Problem:

* Repeated reads

Solution:

* Redis cache
### Challenge 3: Bulk Notifications

Problem:

* Heavy write load

Solution:

* Kafka or RabbitMQ
* Background workers
## Sample Queries

### Fetch Notifications

```sql
SELECT *
FROM notifications
WHERE user_id = 1042
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;
```

### Fetch Unread Notifications

```sql
SELECT *
FROM notifications
WHERE user_id = 1042
AND is_read = FALSE
ORDER BY created_at DESC;
```

### Mark Notification Read

```sql
UPDATE notifications
SET is_read = TRUE,
    updated_at = NOW()
WHERE id='uuid';
```

### Mark All Notifications Read

```sql
UPDATE notifications
SET is_read = TRUE,
    updated_at = NOW()
WHERE user_id = 1042
AND is_read = FALSE;
```

### Delete Notification

```sql
DELETE FROM notifications
WHERE id='uuid';
```

### Create Notification

```sql
INSERT INTO notifications
(
id,
user_id,
notification_type,
title,
message,
priority,
created_at,
updated_at
)
VALUES
(
gen_random_uuid(),
1042,
'Placement',
'Placement Update',
'XYZ Corporation hiring',
5,
NOW(),
NOW()
);
```
# Stage 3 – Query Optimization

## Existing Query

```sql
SELECT *
FROM notifications
WHERE studentID = 1042
AND isRead = false
ORDER BY createdAt ASC;
```

### Is the Query Slow?

Yes.

Reasons:

* Table contains 5 million rows
* Full table scan possible
* Sorting operation is expensive

## Should Every Column Be Indexed?

No.

Reasons:

* Increased storage usage
* Slower inserts
* Slower updates
* Many indexes may never be used

## Better Index

```sql
CREATE INDEX idx_notifications_student_read_created
ON notifications(studentID,isRead,createdAt DESC);
```

This index supports:

```sql
WHERE studentID=?
AND isRead=?
ORDER BY createdAt DESC
```
## Optimized Query

Requirement:

Unread placement notifications from the last 7 days.

```sql
SELECT id,
       notificationType,
       message,
       createdAt
FROM notifications
WHERE studentID = 1042
AND isRead = FALSE
AND notificationType = 'Placement'
AND createdAt >= NOW() - INTERVAL '7 days'
ORDER BY createdAt DESC
LIMIT 100;
```

Recommended Index:

```sql
CREATE INDEX idx_notifications_placement
ON notifications(
studentID,
isRead,
notificationType,
createdAt DESC
);
```
# Stage 4 – Caching Strategy

## Option 1: Cache Entire Notification List

Example:

```text
notifications:user:1042
```

Advantages:

* Fast reads
* Minimal database hits

Disadvantages:

* Large memory usage
* Hard cache invalidation

## Option 2: Cache Only Unread Count

Example:

```text
unread_count:user:1042
```

Advantages:

* Small memory footprint
* Easy invalidation

Disadvantages:

* Notification list still requires database access

## Recommended Approach: Hybrid Strategy

Cache:

* Latest notifications
* Unread count

Redis Keys:

```text
notifications:user:1042:latest
unread_count:user:1042
```

Flow:

```text
Client
  |
Redis
  |
Cache Hit -> Return

Cache Miss
  |
Database
  |
Update Redis
  |
Return
```

### Benefits

* Reduced database load
* Fast response time
* Lower memory usage than full caching
# Stage 5 – Bulk Notification Design

## Existing Implementation

```python
for student_id in student_ids:
    send_email(student_id, message)
    save_to_db(student_id, message)
    push_to_app(student_id, message)
```

### Problems

* Sequential processing
* Slow for 50,000 students
* No retry mechanism
* Failures may cause data inconsistency
## Improved Architecture

```text
HR/Admin
   |
API Gateway
   |
Notification Service
   |
Kafka / RabbitMQ
   |
--------------------------------
|              |              |
Email Worker   Push Worker   DB Worker
```
## New Flow

1. API receives bulk notification request.
2. Request is validated.
3. Campaign record is stored.
4. Message is published to Kafka.
5. Worker services consume messages.
6. Notifications are delivered in parallel.

## Reliability Improvements

### Retry Mechanism

Use exponential backoff.

### Dead Letter Queue

Failed messages move to DLQ.

### Idempotency Key

```text
campaignId-userId
```

Prevents duplicate notifications.

## Outbox Pattern

### Transaction

```sql
BEGIN;

INSERT INTO notifications(...);

INSERT INTO outbox_events(...);

COMMIT;
```

### Worker

```text
Read Outbox
Send Email
Mark Processed
```

### Benefits

* Reliable delivery
* Atomic database operations
* Failure recovery support

# Conclusion

The proposed system uses:

* REST APIs for notification management
* WebSockets for real-time delivery
* PostgreSQL for persistence
* Redis for caching
* Kafka/RabbitMQ for scalable bulk processing
* Optimized indexing for fast queries
* Outbox pattern and retries for reliability

This architecture can efficiently support millions of notifications while maintaining low latency and high reliability.
