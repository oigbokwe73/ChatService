# ChatService


### **Detailed Expansion of Steps 1-5 in the Chat Messaging System**

This section provides an in-depth look at the first five steps of the chat messaging system, covering how a message is **ingested, processed, and stored** before being delivered to its intended recipient.

---

## **1. User Sends a Chat Message via the Client Application**
- The user initiates a message from a **mobile app** or **web application**.
- The front-end **constructs a JSON payload** containing:
  - `SenderID` (user who sent the message)
  - `ReceiverID` (intended recipient of the message)
  - `Timestamp` (when the message was sent)
  - `MessageText` (actual message content)
  - `AttachmentURL` (if applicable)
- This payload is then sent via an HTTP POST request to an **Azure Function (HTTP Trigger)**.
- The HTTP API endpoint might look like this:

```
POST https://chat-functions.azurewebsites.net/api/sendMessage
Content-Type: application/json
```

**Example Payload:**
```json
{
  "SenderID": "user123",
  "ReceiverID": "user456",
  "Timestamp": "2025-02-11T12:30:45Z",
  "MessageText": "Hey, how's it going?",
  "AttachmentURL": null
}
```

---

## **2. Azure Function (HTTP Trigger) Receives the Request**
- The **Azure Function (HTTP Trigger)** acts as the **entry point** for the chat system.
- It receives the JSON payload and **validates the message**:
  - Checks for empty or malformed requests.
  - Scans the message for spam or offensive content.
  - If an attachment is included, generates a **SAS URL** for secure upload to **Azure Blob Storage**.
- After validation, the function **logs the request** for observability in **Azure Monitor**.
- If the request passes validation, the function **forwards it to Azure Service Bus**.


---

## **3. Message is Enqueued in Azure Service Bus**
- The **Azure Function enqueues the chat message** into an **Azure Service Bus Queue or Topic**.
- Service Bus ensures **asynchronous and durable message delivery**.
- **Why Service Bus?**
  - **Guaranteed delivery** even if the receiver is offline.
  - **Scalability**: Service Bus can handle millions of messages per second.
  - **Pub/Sub Support**: Can fan out messages to multiple recipients for group chats.

**Azure Service Bus Configuration:**
- **Queue Name:** `chat-queue`
- **Topic Name:** `chat-topic` (for group messaging)
- **Subscription:** `chat-subscription`

**Service Bus Message Structure:**
```json
{
  "SenderID": "user123",
  "ReceiverID": "user456",
  "MessageText": "Hey, how's it going?",
  "Timestamp": "2025-02-11T12:30:45Z"
}
```

---

## **4. Azure Function (Service Bus Trigger) Processes Messages**
- A **Service Bus-triggered Azure Function** continuously listens to the queue.
- When a message arrives, the function **processes and determines the recipient’s status**:
  - If **recipient is online**, the message is pushed to the client in real-time.
  - If **recipient is offline**, the message is persisted in **Azure SQL Database** or **Azure Table Storage**.


---

## **5. Message is Stored in Azure SQL or Azure Table Storage**
- If the recipient is offline, the message is stored for later retrieval.
- **Azure Table Storage** is used for fast retrieval of chat messages.
- **Azure SQL Database** is used if structured querying and relationships are needed.

### **Table Storage Schema**
| **PartitionKey (ReceiverID)** | **RowKey (Timestamp)** | **SenderID** | **MessageText** |
|---------------------|---------------------|----------|--------------|
| `user456` | `2025-02-11T12-30-45Z` | `user123` | `"Hey, how's it going?"` |

---

## **Mermaid Diagram for Steps 1-5**
```mermaid
sequenceDiagram
    participant User as User (Mobile/Web App)
    participant FunctionHTTP as REST Function (HTTP Trigger)
    participant ServiceBus as Service Bus Queue
    participant FunctionBus as Function (Service Bus Trigger)
    participant TableStorage as Azure Table Storage
    participant SQL as Azure SQL Database

    User->>FunctionHTTP: Send chat message (POST /sendMessage)
    FunctionHTTP->>FunctionHTTP: Validate message
    alt Message Valid
        FunctionHTTP->>ServiceBus: Enqueue message in Service Bus
    else Message Invalid
        FunctionHTTP-->>User: Return error response
    end

    ServiceBus->>FunctionBus: Deliver message from queue
    FunctionBus->>FunctionBus: Process message
    alt Recipient Online
        FunctionBus-->>User: Deliver via WebSocket
    else Recipient Offline
        FunctionBus->>TableStorage: Store message in Table Storage
        FunctionBus->>SQL: Store message in Azure SQL
    end
```

## **Key Takeaways**
1. **Event-Driven Approach**: Azure Functions and Service Bus ensure high scalability.
2. **Asynchronous Processing**: Messages are enqueued and processed reliably.
3. **Storage Options**: Table Storage for NoSQL access, Azure SQL for structured queries.
4. **Real-Time Delivery**: WebSockets notify users instantly if they’re online.
5. **Offline Persistence**: Messages are stored for offline users and retrieved later.


## **Database Tables **

### **Azure SQL Database Schema for a Twitter-Like Chat Messaging Service**  

The schema for a **Twitter-like messaging system** should support:
- **Users** (profile, authentication, metadata)
- **Messages** (tweets, direct messages, mentions)
- **Followers/Following** (social connections)
- **Likes, Retweets, Hashtags** (engagement tracking)
- **Media Storage** (attachments)
- **Notifications** (real-time updates)

---

## **1. Users Table**
Stores user profile details, authentication, and metadata.

```sql
CREATE TABLE Users (
    UserID UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Username NVARCHAR(50) UNIQUE NOT NULL,
    Email NVARCHAR(255) UNIQUE NOT NULL,
    PasswordHash NVARCHAR(512) NOT NULL,
    DisplayName NVARCHAR(100) NOT NULL,
    Bio NVARCHAR(255),
    ProfileImageURL NVARCHAR(512),
    CreatedAt DATETIME DEFAULT GETUTCDATE(),
    LastLogin DATETIME NULL
);
```

---

## **2. Tweets Table**
Stores individual tweets and their metadata.

```sql
CREATE TABLE Tweets (
    TweetID UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserID UNIQUEIDENTIFIER FOREIGN KEY REFERENCES Users(UserID) ON DELETE CASCADE,
    TweetText NVARCHAR(280) NOT NULL,
    MediaURL NVARCHAR(512) NULL,
    CreatedAt DATETIME DEFAULT GETUTCDATE(),
    RetweetCount INT DEFAULT 0,
    LikeCount INT DEFAULT 0
);
```

---

## **3. Direct Messages Table**
Stores private chat messages between users.

```sql
CREATE TABLE DirectMessages (
    MessageID UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    SenderID UNIQUEIDENTIFIER FOREIGN KEY REFERENCES Users(UserID) ON DELETE CASCADE,
    ReceiverID UNIQUEIDENTIFIER FOREIGN KEY REFERENCES Users(UserID) ON DELETE CASCADE,
    MessageText NVARCHAR(1000) NOT NULL,
    MediaURL NVARCHAR(512) NULL,
    SentAt DATETIME DEFAULT GETUTCDATE(),
    IsRead BIT DEFAULT 0
);
```

---

## **4. Followers Table**
Tracks user relationships (who follows whom).

```sql
CREATE TABLE Followers (
    FollowerID UNIQUEIDENTIFIER FOREIGN KEY REFERENCES Users(UserID) ON DELETE CASCADE,
    FollowingID UNIQUEIDENTIFIER FOREIGN KEY REFERENCES Users(UserID) ON DELETE CASCADE,
    FollowedAt DATETIME DEFAULT GETUTCDATE(),
    PRIMARY KEY (FollowerID, FollowingID)
);
```

---

## **5. Likes Table**
Tracks tweets liked by users.

```sql
CREATE TABLE Likes (
    LikeID UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserID UNIQUEIDENTIFIER FOREIGN KEY REFERENCES Users(UserID) ON DELETE CASCADE,
    TweetID UNIQUEIDENTIFIER FOREIGN KEY REFERENCES Tweets(TweetID) ON DELETE CASCADE,
    LikedAt DATETIME DEFAULT GETUTCDATE()
);
```

---

## **6. Retweets Table**
Tracks retweets and their references.

```sql
CREATE TABLE Retweets (
    RetweetID UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserID UNIQUEIDENTIFIER FOREIGN KEY REFERENCES Users(UserID) ON DELETE CASCADE,
    OriginalTweetID UNIQUEIDENTIFIER FOREIGN KEY REFERENCES Tweets(TweetID) ON DELETE CASCADE,
    RetweetText NVARCHAR(280) NULL,
    RetweetedAt DATETIME DEFAULT GETUTCDATE()
);
```

---

## **7. Hashtags Table**
Tracks hashtags used in tweets.

```sql
CREATE TABLE Hashtags (
    HashtagID UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Hashtag NVARCHAR(100) UNIQUE NOT NULL
);
```

---

## **8. TweetHashtags Table**
Associates tweets with hashtags.

```sql
CREATE TABLE TweetHashtags (
    TweetID UNIQUEIDENTIFIER FOREIGN KEY REFERENCES Tweets(TweetID) ON DELETE CASCADE,
    HashtagID UNIQUEIDENTIFIER FOREIGN KEY REFERENCES Hashtags(HashtagID) ON DELETE CASCADE,
    PRIMARY KEY (TweetID, HashtagID)
);
```

---

## **9. Media Storage Table**
Stores metadata for images, videos, and GIFs.

```sql
CREATE TABLE Media (
    MediaID UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserID UNIQUEIDENTIFIER FOREIGN KEY REFERENCES Users(UserID) ON DELETE CASCADE,
    TweetID UNIQUEIDENTIFIER NULL FOREIGN KEY REFERENCES Tweets(TweetID) ON DELETE CASCADE,
    DirectMessageID UNIQUEIDENTIFIER NULL FOREIGN KEY REFERENCES DirectMessages(MessageID) ON DELETE CASCADE,
    MediaURL NVARCHAR(512) NOT NULL,
    MediaType NVARCHAR(50) NOT NULL, -- (image, video, gif)
    UploadedAt DATETIME DEFAULT GETUTCDATE()
);
```

---

## **10. Notifications Table**
Stores user notifications (mentions, likes, new followers).

```sql
CREATE TABLE Notifications (
    NotificationID UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserID UNIQUEIDENTIFIER FOREIGN KEY REFERENCES Users(UserID) ON DELETE CASCADE,
    NotificationType NVARCHAR(50) NOT NULL, -- (mention, like, retweet, follow, message)
    ReferenceID UNIQUEIDENTIFIER NOT NULL, -- (TweetID, UserID, MessageID)
    IsRead BIT DEFAULT 0,
    CreatedAt DATETIME DEFAULT GETUTCDATE()
);
```

---

## **Indexes and Performance Optimization**
1. **Index for fast lookups on tweets by user:**
   ```sql
   CREATE INDEX idx_user_tweets ON Tweets(UserID, CreatedAt DESC);
   ```
2. **Index for fetching recent messages:**
   ```sql
   CREATE INDEX idx_messages_receiver ON DirectMessages(ReceiverID, SentAt DESC);
   ```
3. **Index for fast hashtag searches:**
   ```sql
   CREATE INDEX idx_hashtag_lookup ON Hashtags(Hashtag);
   ```
4. **Index for checking user follows:**
   ```sql
   CREATE INDEX idx_followers ON Followers(FollowerID, FollowingID);
   ```

---

## **Final Thoughts**
This schema provides a **highly scalable Twitter-like chat and messaging system** on **Azure SQL**. It supports:
- **User authentication and profiles**
- **Tweet storage and engagement tracking**
- **Direct messaging between users**
- **Follower/following relationships**
- **Hashtag searchability**
- **Media uploads (images, videos, GIFs)**
- **Notifications for user interactions**



