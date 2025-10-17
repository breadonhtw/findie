# Findie iOS App - Firebase Structure Documentation

## Firebase Project Configuration

### Project Setup

- **Project Name**: Findie Production
- **Project ID**: `findie-app-prod`
- **Region**: `us-central1`
- **iOS Bundle ID**: `com.findie.app`

### Development Environment

- **Project Name**: Findie Development
- **Project ID**: `findie-app-dev`
- **Region**: `us-central1`

## Cloud Firestore Database Structure

### Collection: `users`

Stores user profiles and preferences.

```
users/ (collection)
  {userId}/ (document)
    - id: string
    - username: string (indexed)
    - email: string
    - displayName: string
    - avatarURL: string?
    - bio: string?
    - favoriteGenres: array<string>
    - preferredPlatforms: array<string>
    - priceRange: string
    - notificationsEnabled: boolean
    - emailUpdates: boolean
    - shareDiscoveries: boolean
    - accountType: string
    - gamesDiscovered: number
    - gamesLiked: number
    - gamesWishlisted: number
    - joinedDate: timestamp
    - lastActiveDate: timestamp
    - friendIds: array<string>
    - followerCount: number
    - followingCount: number
    - steamId: string?
    - itchioUsername: string?
    - isProfilePublic: boolean
    - allowFriendRequests: boolean
```

**Indexes:**

- `username` (ascending)
- `accountType` (ascending) + `lastActiveDate` (descending)
- `gamesDiscovered` (descending)

### Collection: `developers`

Stores indie developer/studio information.

```
developers/ (collection)
  {developerId}/ (document)
    - id: string
    - userId: string? (indexed)
    - studioName: string
    - displayName: string
    - logoURL: string?
    - bannerURL: string?
    - bio: string
    - website: string?
    - location: string?
    - foundedYear: number?
    - publishedGameIds: array<string>
    - twitterHandle: string?
    - discordURL: string?
    - youtubeChannel: string?
    - steamDeveloperId: string?
    - itchioDeveloperId: string?
    - isVerified: boolean
    - verificationDate: timestamp?
    - totalViews: number
    - totalLikes: number
    - totalWishlists: number
    - followerCount: number
    - joinedDate: timestamp
    - lastActive: timestamp
    - isActive: boolean
```

**Indexes:**

- `userId` (ascending)
- `isVerified` (descending) + `followerCount` (descending)
- `studioName` (ascending)

### Collection: `games`

Core game catalog.

```
games/ (collection)
  {gameId}/ (document)
    - id: string
    - title: string (indexed)
    - subtitle: string?
    - description: string
    - shortDescription: string
    - iconURL: string
    - coverImageURL: string
    - screenshotURLs: array<string>
    - trailerURL: string?
    - developerId: string (indexed)
    - developerName: string
    - studioName: string?
    - genres: array<string> (indexed)
    - tags: array<string> (indexed)
    - platforms: array<string>
    - releaseDate: timestamp?
    - releaseStatus: string
    - price: number?
    - currency: string
    - isFree: boolean
    - createdAt: timestamp
    - updatedAt: timestamp
    - isActive: boolean
    - viewCount: number
    - likeCount: number
    - wishlistCount: number
    - shareCount: number
    - steamURL: string?
    - itchioURL: string?
    - officialWebsiteURL: string?
    - discordURL: string?
```

**Indexes:**

- `developerId` (ascending) + `createdAt` (descending)
- `genres` (array-contains) + `wishlistCount` (descending)
- `tags` (array-contains) + `likeCount` (descending)
- `releaseStatus` (ascending) + `releaseDate` (descending)
- `isActive` (ascending) + `createdAt` (descending)

### Collection: `interactions`

User swipe interactions for ML recommendations.

```
interactions/ (collection)
  {interactionId}/ (document)
    - id: string
    - userId: string (indexed)
    - gameId: string (indexed)
    - action: string
    - timestamp: timestamp
    - sessionId: string
    - cardPosition: number
    - viewDuration: number
    - userGenrePreferences: array<string>
    - gameGenres: array<string>
    - gameTags: array<string>
```

**Indexes:**

- `userId` (ascending) + `timestamp` (descending)
- `gameId` (ascending) + `action` (ascending)
- `sessionId` (ascending) + `timestamp` (ascending)
- `userId` (ascending) + `action` (ascending) + `timestamp` (descending)

**Note**: Interactions older than 90 days are archived to Cloud Storage.

### Collection: `wishlists`

User wishlists (one document per user).

```
wishlists/ (collection)
  {userId}/ (document)
    - id: string
    - userId: string
    - items: array<map>
      - id: string
      - gameId: string
      - gameTitle: string
      - gameIconURL: string
      - developerName: string
      - addedDate: timestamp
      - notes: string?
      - priority: string
      - notifyOnRelease: boolean
      - notifyOnDiscount: boolean
      - steamAppId: string?
      - itchioGameId: string?
      - priceAtAdd: number?
      - currentPrice: number?
      - isOnSale: boolean
      - releaseDate: timestamp?
    - lastUpdated: timestamp
    - steamSyncEnabled: boolean
    - steamLastSynced: timestamp?
    - itchioSyncEnabled: boolean
    - itchioLastSynced: timestamp?
```

**Note**: Array limited to 500 items per user.

### Collection: `tags`

Custom tags for game discovery.

```
tags/ (collection)
  {tagId}/ (document)
    - id: string
    - name: string (indexed, unique)
    - category: string
    - usageCount: number
    - isOfficial: boolean
    - createdAt: timestamp
```

**Indexes:**

- `name` (ascending)
- `category` (ascending) + `usageCount` (descending)
- `isOfficial` (descending) + `usageCount` (descending)

### Collection: `notifications`

User notifications.

```
notifications/ (collection)
  {notificationId}/ (document)
    - id: string
    - userId: string (indexed)
    - type: string
    - title: string
    - message: string
    - timestamp: timestamp
    - isRead: boolean
    - actionURL: string?
    - gameId: string?
    - developerId: string?
```

**Indexes:**

- `userId` (ascending) + `timestamp` (descending)
- `userId` (ascending) + `isRead` (ascending) + `timestamp` (descending)

**Cleanup**: Notifications older than 30 days are automatically deleted.

### Collection: `analytics`

Developer analytics events.

```
analytics/ (collection)
  {eventId}/ (document)
    - eventId: string
    - gameId: string (indexed)
    - developerId: string (indexed)
    - eventType: string
    - timestamp: timestamp
    - anonymousUserId: string
    - userGenrePreferences: array<string>?
    - sessionId: string
    - deviceType: string
    - appVersion: string
```

**Indexes:**

- `gameId` (ascending) + `eventType` (ascending) + `timestamp` (descending)
- `developerId` (ascending) + `timestamp` (descending)

**Aggregation**: Events are aggregated daily via Cloud Functions.

### Collection: `sessions`

Active user sessions for analytics.

```
sessions/ (collection)
  {sessionId}/ (document)
    - id: string
    - userId: string
    - startTime: timestamp
    - endTime: timestamp?
    - gamesViewed: number
    - interactions: number
    - deviceType: string
    - appVersion: string
```

**Cleanup**: Sessions older than 7 days are archived.

## Firebase Authentication

### Supported Auth Methods

1. **Email/Password**

   - Standard email + password sign-up
   - Email verification required
   - Password reset via email

2. **Apple Sign-In**

   - Native Apple authentication
   - Required for App Store compliance
   - Anonymous email relay supported

3. **Anonymous Authentication**
   - Allow guest browsing
   - Can upgrade to full account later
   - Limited features (no wishlist sync, no developer tools)

### Auth Flow

```swift
// Sign up with email
let authResult = try await Auth.auth().createUser(
    withEmail: email,
    password: password
)

// Create user document in Firestore
let user = User(
    id: authResult.user.uid,
    username: username,
    email: email,
    // ... other fields
)
try await Firestore.firestore()
    .collection("users")
    .document(user.id)
    .setData(from: user)
```

### Custom Claims

Used for role-based access control:

```json
{
  "developer": true,
  "verified": true,
  "admin": false
}
```

Set via Cloud Functions after developer verification.

## Firebase Storage

### Storage Structure

```
gs://findie-app-prod.appspot.com/

games/
  {gameId}/
    icon.jpg              # 512x512 app icon
    cover.jpg             # 1920x1080 cover image
    screenshots/
      screenshot_1.jpg    # 1920x1080 gameplay
      screenshot_2.jpg
      ...
    trailer.mp4           # Optional video

developers/
  {developerId}/
    logo.jpg              # 256x256 studio logo
    banner.jpg            # 1920x400 profile banner

users/
  {userId}/
    avatar.jpg            # 256x256 profile picture
```

### Storage Rules

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    // Games - only developers can upload
    match /games/{gameId}/{allPaths=**} {
      allow read: if true;
      allow write: if request.auth != null
                   && request.auth.token.developer == true;
    }

    // Developers - only own profile
    match /developers/{developerId}/{allPaths=**} {
      allow read: if true;
      allow write: if request.auth.uid == developerId;
    }

    // Users - only own profile
    match /users/{userId}/{allPaths=**} {
      allow read: if true;
      allow write: if request.auth.uid == userId;
    }
  }
}
```

### Image Optimization

All uploaded images are automatically processed via Cloud Functions:

- Resize to appropriate dimensions
- Convert to WebP format
- Generate thumbnails (256x256, 512x512)
- Store multiple sizes for responsive loading

## Cloud Functions

### Deployed Functions

#### 1. `onGameCreate`

**Trigger**: Firestore onCreate (`games/{gameId}`)
**Purpose**:

- Update developer's `publishedGameIds` array
- Send notification to developer's followers
- Initialize game analytics document

#### 2. `onUserInteraction`

**Trigger**: Firestore onCreate (`interactions/{interactionId}`)
**Purpose**:

- Update game engagement metrics (likeCount, viewCount)
- Queue interaction for ML model training
- Update user stats (gamesDiscovered, gamesLiked)

#### 3. `generateRecommendations`

**Trigger**: HTTPS callable
**Purpose**:

- Fetch user's interaction history
- Run ML recommendation model
- Return personalized game list
- Cache results for 6 hours

#### 4. `syncSteamWishlist`

**Trigger**: HTTPS callable
**Purpose**:

- Fetch user's Steam wishlist via API
- Match Steam games to Findie catalog
- Update user's wishlist in Firestore
- Return sync status

#### 5. `aggregateAnalytics`

**Trigger**: Scheduled (daily at 2 AM UTC)
**Purpose**:

- Calculate daily metrics per game
- Update developer dashboard stats
- Generate trending games list
- Archive old events

#### 6. `sendNotifications`

**Trigger**: Scheduled (hourly)
**Purpose**:

- Check for game releases from wishlists
- Check for price drops
- Send push notifications via FCM
- Create in-app notification documents

#### 7. `processImageUpload`

**Trigger**: Storage onCreate (`games/{gameId}/*`)
**Purpose**:

- Validate image dimensions and format
- Generate optimized versions
- Create thumbnails
- Update Firestore with URLs

## Firestore Security Rules

### Core Security Principles

1. Users can only read/write their own data
2. All users can read public game/developer data
3. Only developers can write game data
4. Interactions are write-only (privacy)
5. Server-side validation for all writes

### Security Rules Implementation

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Helper functions
    function isAuthenticated() {
      return request.auth != null;
    }

    function isOwner(userId) {
      return request.auth.uid == userId;
    }

    function isDeveloper() {
      return isAuthenticated()
          && request.auth.token.developer == true;
    }

    function isVerifiedDeveloper() {
      return isDeveloper()
          && request.auth.token.verified == true;
    }

    // Users collection
    match /users/{userId} {
      allow read: if isAuthenticated()
                  && (isOwner(userId) || resource.data.isProfilePublic);
      allow create: if isAuthenticated() && isOwner(userId);
      allow update: if isAuthenticated() && isOwner(userId);
      allow delete: if false; // Soft delete only
    }

    // Developers collection
    match /developers/{developerId} {
      allow read: if true; // Public data
      allow create: if isAuthenticated() && isDeveloper();
      allow update: if isAuthenticated()
                    && resource.data.userId == request.auth.uid;
      allow delete: if false;
    }

    // Games collection
    match /games/{gameId} {
      allow read: if resource.data.isActive == true;
      allow create: if isDeveloper()
                    && request.resource.data.developerId == request.auth.uid;
      allow update: if isDeveloper()
                    && resource.data.developerId == request.auth.uid;
      allow delete: if false; // Soft delete only
    }

    // Interactions collection
    match /interactions/{interactionId} {
      allow read: if false; // Privacy: no direct reads
      allow create: if isAuthenticated()
                    && request.resource.data.userId == request.auth.uid;
      allow update, delete: if false;
    }

    // Wishlists collection
    match /wishlists/{userId} {
      allow read: if isAuthenticated() && isOwner(userId);
      allow write: if isAuthenticated() && isOwner(userId);
    }

    // Tags collection
    match /tags/{tagId} {
      allow read: if true;
      allow write: if isDeveloper(); // Developers can suggest tags
    }

    // Notifications collection
    match /notifications/{notificationId} {
      allow read: if isAuthenticated()
                  && resource.data.userId == request.auth.uid;
      allow create: if false; // Only via Cloud Functions
      allow update: if isAuthenticated()
                    && resource.data.userId == request.auth.uid;
      allow delete: if isAuthenticated()
                    && resource.data.userId == request.auth.uid;
    }

    // Analytics collection
    match /analytics/{eventId} {
      allow read: if isDeveloper()
                  && resource.data.developerId == request.auth.uid;
      allow write: if false; // Only via Cloud Functions
    }
  }
}
```

## Firebase Indexes

### Composite Indexes (Required)

```json
{
  "indexes": [
    {
      "collectionGroup": "games",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "genres", "arrayConfig": "CONTAINS" },
        { "fieldPath": "wishlistCount", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "games",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "isActive", "order": "ASCENDING" },
        { "fieldPath": "releaseDate", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "interactions",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "userId", "order": "ASCENDING" },
        { "fieldPath": "action", "order": "ASCENDING" },
        { "fieldPath": "timestamp", "order": "DESCENDING" }
      ]
    }
  ]
}
```

Deploy via: `firebase deploy --only firestore:indexes`

## Firebase ML

### Recommendation Model

**Model Type**: TensorFlow Lite  
**Input**: User interaction vector (genres, tags, platforms)  
**Output**: Ranked list of game IDs  
**Update Frequency**: Weekly retrain, daily incremental updates

### Model Deployment

```bash
# Upload model to Firebase ML
firebase ml:models:upload \
  --file recommendation_model.tflite \
  --model-name findie_recommendations_v1
```

## Cost Optimization Strategies

1. **Firestore Reads**

   - Cache game data locally (UserDefaults)
   - Batch read operations
   - Use real-time listeners sparingly

2. **Storage**

   - Compress images before upload
   - Use WebP format
   - CDN caching via Firebase Hosting

3. **Cloud Functions**

   - Set memory limits appropriately
   - Use pub/sub for batching
   - Archive old data to Cloud Storage

4. **Analytics**
   - Aggregate events daily
   - Archive raw events after 90 days
   - Use BigQuery export for deep analytics

## Backup & Recovery

- **Automated Daily Backups**: Firestore data exported to Cloud Storage
- **Point-in-Time Recovery**: Available for last 7 days
- **Disaster Recovery**: Restore from latest backup + replay Cloud Functions

---

**Last Updated**: October 2025  
**Firebase SDK Version**: iOS SDK 10.x
