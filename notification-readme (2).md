# Notification Center Documentation

## System Flow Diagram
The notification system consists of two main flows: the Publishing Flow and the Retrieval Flow. Each component in the diagram corresponds to specific functionality documented in later sections.

<details>
<summary>Click to expand the Notification Center Flow Diagram</summary>

```mermaid
flowchart TD
    subgraph "Publishing Flow"
        A[User Action] -->|Triggers| B[publishMessage]
        B -->|Publishes to| C[notificationCenter Topic]
        C -->|Triggers| D[Cloud Function]
        D -->|Process Event| E{Event Type}
        E -->|likesOnPost| F1[Handle Post Likes]
        E -->|likesOnComment| F2[Handle Comment Likes]
        E -->|commentOnPost| F3[Handle Post Comments]
        E -->|follow| F4[Handle Follows]
        E -->|commentOnMandiPost| F5[Handle Mandi Comments]
        
        F1 & F2 & F3 & F4 & F5 -->|Update| G[Firebase DB]
        F1 & F2 & F3 & F4 & F5 -->|Send| H[External API]
    end

    subgraph "Retrieval Flow"
        I[Client Request] -->|Triggers| J[getNotifications Function]
        J -->|Fetch| K1[Chat Notifications]
        J -->|Fetch| K2[Global Notifications]
        J -->|Fetch| K3[Social Notifications]
        J -->|Fetch| K4[Geographic Notifications]
        
        K1 & K2 & K3 & K4 -->|Merge & Sort| L[Combined Feed]
        L -->|Track| M[Mixpanel Analytics]
        L -->|Return| N[Client Display]
        
        O[Read Status Update] -->|Triggers| P[updateNotifications]
        P -->|5s Delay| Q[Update Last Read]
    end

    G -.->|Read| J
    H -.->|Fetch| J
```

</details>

> **Note**: This diagram is rendered using [Mermaid](https://mermaid.js.org/). GitHub automatically renders Mermaid diagrams in Markdown files. If you're viewing this on another platform, you may need to use a Mermaid-compatible viewer.

### Publishing Flow Components
1. **User Action (A)**
   - Corresponds to user interactions documented in [Notification Types](#notification-types) section
   - Examples: likes, comments, follows

2. **Message Publishing (B → C)**
   - Detailed in [Publishing Messages](#publishing-messages) section
   - Uses the following format:
   ```javascript
   publishMessage('notificationCenter', JSON.stringify({...objectToSave, type: 'commentOnPost'}), pubSubClient);
   ```

3. **Event Processing (D → E)**
   - Handled by Cloud Functions described in [Architecture](#architecture) section
   - Routes events to appropriate handlers based on type

4. **Event Handlers (F1 → F5)**
   - Implementation details found in [Data Structure](#data-structure) section
   - Each handler processes specific notification types

5. **Storage and API (G, H)**
   - Firebase DB operations detailed in [Unread Notifications Counter](#unread-notifications-counter) section
   - External API calls documented in [API Integration](#api-integration) section

### Retrieval Flow Components
1. **Client Request (I)**
   - Initiates notification fetch process
   - Triggers `getNotifications` function

2. **Notification Sources (K1 → K4)**
   - Details in [Getting Notifications](#getting-notifications) section
   - Includes:
     - Chat notifications
     - Global notifications
     - Social notifications
     - Geographic notifications

3. **Feed Generation (L)**
   - Process described in [Notification Sorting and Merging](#notification-sorting-and-merging) section
   - Combines multiple notification sources
   - Applies sorting and relevance rules

4. **Analytics and Display (M, N)**
   - Analytics integration detailed in [Analytics Integration](#analytics-integration) section
   - Client display guidelines in [Best Practices](#best-practices) section

5. **Read Status Management (O → Q)**
   - Process documented in [Unread Status Management](#unread-status-management) section
   - Includes delay mechanism for processing

## Overview
The Notification Center is responsible for handling various types of user interactions and sending appropriate notifications. It uses Google Cloud Pub/Sub for message publishing and Firebase for data storage and real-time updates.

## Architecture
- **Publisher**: Services publish messages to the 'notificationCenter' topic
- **Subscriber**: Cloud Function handles the published messages and processes different notification types
- **Storage**: Uses Firebase Realtime Database and Firestore
- **External API**: Notifications are sent to an external API endpoint (kc.retailpulse.ai)

## Notification Types
The system handles the following types of notifications:

1. **Post Likes** (`likesOnPost`)
   - Triggered when a user likes a post
   - Updates unread notification counter for post owner
   - Sends notification with liker's details

2. **Comment Likes** (`likesOnComment`)
   - Triggered when a user likes a comment
   - Updates unread notification counter for comment owner
   - Includes comment content in notification

3. **Post Comments** (`commentOnPost`)
   - Triggered when a user comments on a post
   - Differentiates between direct comments and replies
   - Truncates comments longer than 100 characters

4. **Comment Replies** (`commentOnPost` with `is_reply: true`)
   - Handles replies to existing comments
   - Updates notification counter for parent comment owner
   - Includes truncated reply content

5. **User Follows** (`follow`)
   - Triggered when a user follows another user
   - Updates notification counter for followed user
   - Includes follower's profile information

6. **Mandi Post Comments** (`commentOnMandiPost`)
   - Similar to regular post comments but specific to Mandi posts
   - Handles both direct comments and replies
   - Uses same truncation rules as regular comments

## Publishing Messages
To publish a notification event, use the following format:

```javascript
publishMessage(
    'notificationCenter',
    JSON.stringify({
        ...objectToSave,
        type: 'commentOnPost' // Notification type
    }),
    pubSubClient
);
```

## Data Structure
Each notification type requires specific data fields:

### Post Like Notification
```javascript
{
    news_id: string,
    post_user_id: string,
    liker_user_id: string,
    liker_user_name: string,
    liker_user_pic: string,
    created_at: timestamp
}
```

### Comment Notification
```javascript
{
    news_id: string,
    post_user_id: string,
    comment: string,
    commenter_user_id: string,
    commenter_user_name: string,
    commenter_user_pic: string,
    created_at: timestamp,
    comment_id: string (optional)
}
```

### Follow Notification
```javascript
{
    followee_user_id: string,
    follower_user_id: string,
    follower_user_name: string,
    follower_user_pic: string,
    created_at: timestamp
}
```

## Unread Notifications Counter
- Each notification updates the recipient's unread notification counter
- Counter is stored in Firebase at path: `/users/${userId}/no_of_unread_notifications`
- Uses Firebase's `ServerValue.increment(1)` for atomic updates

## Error Handling
- Each notification type has its own try-catch block
- Failed notification counter updates are logged but don't stop the notification flow
- API call failures are captured and logged

## Profile Image Handling
The system includes a utility function `getWebpUrls()` that:
- Converts regular image URLs to WebP format
- Handles both KPL and thumbnail path conversions
- Falls back to a default dummy profile picture if no image is provided

## API Integration
Notifications are sent to external endpoints:
- Base URL: `https://kc.retailpulse.ai/api`
- Endpoints:
  - `/postLikes`
  - `/comment`
  - `/reply`
  - `/follow`
  - `/mandiComment`
  - `/mandiCommentReply`

## Environment Configuration
The system supports multiple environments:
- Production: Uses `productionApp` Firebase instance
- Staging: Uses `stageApp` Firebase instance
- Meta: Uses `metaApp` Firebase instance for metadata

## Notification Retrieval Flow

## Getting Notifications
The system provides a cloud function `getNotifications` that fetches notifications from multiple sources:

1. **Chat Notifications**
   - Fetched from `/p2p/unreadMessages/{userId}`
   - Includes information about last message sender and unread count
   - Chat notifications appear with a special 'p2p' tag

2. **Global Notifications**
   - Retrieved from `/notificationCenter/global`
   - Version-dependent (disabled for app versions >= 5.9.0)
   - Sorted by creation timestamp

3. **Social Notifications**
   - Fetched via `/allNotifications` endpoint
   - Supports multiple notification types:
     - Mandi comments and replies
     - Post likes
     - Comment likes
     - Post comments
     - Reply notifications
     - Follower activities
     - Published posts

4. **Geographic Notifications**
   - Based on user's city and cluster
   - Retrieved from `/userProfile/{uid}/district` and `/userProfile/{uid}/geo_coded_cluster`

## Notification Sorting and Merging
1. **Priority Based Placement**
   - Social notifications with relevance scores are placed first
   - Other notifications are filled in remaining slots
   - Chat notifications are appended at the end

2. **Timestamp Sorting**
   - All notifications are sorted by `created_at` timestamp
   - Most recent notifications appear first

## Unread Status Management
- System tracks last read time at `/notificationCenter/users/{uid}/readTime`
- Notifications created after last read time are marked as unread
- Unread count is calculated and returned with notifications
- Updates user's unread message count via `updateUnreadMsgCount`

## Analytics Integration
- Mixpanel events are tracked for notification delivery
- Event "Notifications Pushed In Notification Center" includes:
  - User ID (distinct_id)
  - Full notification payload

## Notification Update Function
The system provides `updateNotifications` function that:
- Takes a readTime parameter
- Updates the user's last read timestamp
- Includes a 5-second delay for processing
- Updates notification read status in the database
