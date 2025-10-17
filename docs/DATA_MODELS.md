# Findie iOS App - Data Models Documentation

## Overview

This document defines the core data structures used throughout the Findie iOS app. All models are designed to work seamlessly with Firebase Firestore and follow Swift's `Codable` protocol for easy serialization.

## Core Models

### 1. Game Model

The primary entity representing an indie game in the platform.

```swift
struct Game: Identifiable, Codable {
    // Identity
    let id: String                          // Firestore document ID
    let title: String
    let subtitle: String?                   // Short tagline

    // Description
    let description: String                 // Full description
    let shortDescription: String            // 140 chars for cards

    // Media
    let iconURL: String                     // App icon / logo
    let coverImageURL: String               // Main card image
    let screenshotURLs: [String]            // Gameplay screenshots (max 10)
    let trailerURL: String?                 // Video trailer URL

    // Creator Info
    let developerId: String                 // Reference to Developer
    let developerName: String               // Denormalized for quick access
    let studioName: String?

    // Classification
    let genres: [Genre]                     // Primary genres
    let tags: [String]                      // Custom tags (e.g., "pixel-art", "roguelike")
    let platforms: [Platform]               // Where it's available

    // Release Info
    let releaseDate: Date?                  // nil if unreleased
    let releaseStatus: ReleaseStatus        // released, early-access, upcoming

    // Pricing
    let price: Decimal?                     // nil if free
    let currency: String                    // USD, EUR, etc.
    let isFree: Bool

    // Metadata
    let createdAt: Date
    let updatedAt: Date
    let isActive: Bool                      // For soft deletion

    // Engagement Metrics (calculated)
    let viewCount: Int
    let likeCount: Int
    let wishlistCount: Int
    let shareCount: Int

    // External Links
    let steamURL: String?
    let itchioURL: String?
    let officialWebsiteURL: String?
    let discordURL: String?
}

// Supporting Enums
enum Genre: String, Codable, CaseIterable {
    case action, adventure, puzzle, rpg, strategy
    case simulation, platformer, shooter, horror, indie
    case casual, racing, sports, fighting, rhythm
}

enum Platform: String, Codable {
    case windows, mac, linux, steam, itchio, epicGames, gog
}

enum ReleaseStatus: String, Codable {
    case released, earlyAccess, upcoming, cancelled
}
```

### 2. User Model

Represents a player using the Findie app.

```swift
struct User: Identifiable, Codable {
    // Identity
    let id: String                          // Firebase Auth UID
    let username: String
    let email: String

    // Profile
    var displayName: String
    var avatarURL: String?
    var bio: String?

    // Preferences
    var favoriteGenres: [Genre]             // For recommendations
    var preferredPlatforms: [Platform]
    var priceRange: PricePreference         // free, under10, under20, any

    // Settings
    var notificationsEnabled: Bool
    var emailUpdates: Bool
    var shareDiscoveries: Bool              // Auto-share likes

    // Account Type
    var accountType: AccountType            // player, developer, both
    var isDeveloper: Bool {
        accountType == .developer || accountType == .both
    }

    // Stats
    var gamesDiscovered: Int                // Total games viewed
    var gamesLiked: Int
    var gamesWishlisted: Int
    var joinedDate: Date
    var lastActiveDate: Date

    // Social
    var friendIds: [String]                 // User IDs of friends
    var followerCount: Int
    var followingCount: Int

    // Sync Integrations
    var steamId: String?
    var itchioUsername: String?

    // Privacy
    var isProfilePublic: Bool
    var allowFriendRequests: Bool
}

enum AccountType: String, Codable {
    case player, developer, both
}

enum PricePreference: String, Codable {
    case free
    case under10
    case under20
    case under30
    case any

    var maxPrice: Decimal? {
        switch self {
        case .free: return 0
        case .under10: return 10
        case .under20: return 20
        case .under30: return 30
        case .any: return nil
        }
    }
}
```

### 3. SwipeInteraction Model

Tracks user swipe actions for recommendations and analytics.

```swift
struct SwipeInteraction: Identifiable, Codable {
    let id: String                          // Auto-generated
    let userId: String
    let gameId: String

    let action: SwipeAction
    let timestamp: Date

    // Context
    let sessionId: String                   // Group interactions by session
    let cardPosition: Int                   // Position in feed (0-indexed)
    let viewDuration: TimeInterval          // Seconds spent viewing card

    // Metadata for ML
    let userGenrePreferences: [Genre]       // Snapshot at time of swipe
    let gameGenres: [Genre]
    let gameTags: [String]
}

enum SwipeAction: String, Codable {
    case like           // Swipe right
    case dislike        // Swipe left
    case superLike      // Swipe up (bookmark for later)
    case skip           // Swipe down (not interested, no signal)
    case wishlist       // Tap wishlist button
    case share          // Tap share button
    case viewDetails    // Tap to open game details
}
```

### 4. Wishlist Model

User's saved games across different platforms.

```swift
struct Wishlist: Identifiable, Codable {
    let id: String                          // User ID (one wishlist per user)
    let userId: String

    var items: [WishlistItem]
    var lastUpdated: Date

    // Sync status
    var steamSyncEnabled: Bool
    var steamLastSynced: Date?
    var itchioSyncEnabled: Bool
    var itchioLastSynced: Date?
}

struct WishlistItem: Identifiable, Codable {
    let id: String                          // Unique item ID
    let gameId: String

    // Denormalized for quick display
    let gameTitle: String
    let gameIconURL: String
    let developerName: String

    let addedDate: Date
    var notes: String?                      // Personal notes
    var priority: Priority                  // high, medium, low
    var notifyOnRelease: Bool
    var notifyOnDiscount: Bool

    // External platform info
    var steamAppId: String?
    var itchioGameId: String?

    // Tracking
    var priceAtAdd: Decimal?
    var currentPrice: Decimal?
    var isOnSale: Bool
    var releaseDate: Date?
}

enum Priority: String, Codable {
    case high, medium, low
}
```

### 5. Developer Model

Represents indie game creators and studios.

```swift
struct Developer: Identifiable, Codable {
    // Identity
    let id: String                          // Same as User ID if linked
    let userId: String?                     // Link to User account

    // Studio Info
    let studioName: String
    let displayName: String
    var logoURL: String?
    var bannerURL: String?

    // Description
    var bio: String
    var website: String?
    var location: String?
    var foundedYear: Int?

    // Published Games
    var publishedGameIds: [String]
    var gameCount: Int {
        publishedGameIds.count
    }

    // Social Links
    var twitterHandle: String?
    var discordURL: String?
    var youtubeChannel: String?

    // Platform Presence
    var steamDeveloperId: String?
    var itchioDeveloperId: String?

    // Verification
    var isVerified: Bool
    var verificationDate: Date?

    // Stats (aggregated)
    var totalViews: Int
    var totalLikes: Int
    var totalWishlists: Int
    var followerCount: Int

    // Metadata
    let joinedDate: Date
    var lastActive: Date
    var isActive: Bool
}
```

### 6. Tag Model

Custom tagging system for enhanced discovery.

```swift
struct Tag: Identifiable, Codable {
    let id: String
    let name: String                        // e.g., "pixel-art", "cozy"
    let category: TagCategory

    var usageCount: Int                     // How many games use this
    var isOfficial: Bool                    // Curated by Findie team
    var createdAt: Date
}

enum TagCategory: String, Codable {
    case artStyle       // pixel-art, hand-drawn, 3d, voxel
    case mood           // cozy, intense, relaxing, scary
    case gameplay       // roguelike, metroidvania, bullet-hell
    case theme          // fantasy, scifi, historical, abstract
    case playerCount    // singleplayer, multiplayer, coop
    case difficulty     // casual, hardcore, challenging
}
```

### 7. AnalyticsEvent Model

Developer analytics and KPI tracking.

```swift
struct AnalyticsEvent: Codable {
    let eventId: String
    let gameId: String
    let developerId: String

    let eventType: EventType
    let timestamp: Date

    // User context (anonymized)
    let anonymousUserId: String             // Hashed user ID
    let userGenrePreferences: [Genre]?

    // Session context
    let sessionId: String
    let deviceType: String                  // iPhone, iPad
    let appVersion: String
}

enum EventType: String, Codable {
    case gameView           // Card appeared in feed
    case gameClick          // User tapped for details
    case gameLike           // Swiped right
    case gameDislike        // Swiped left
    case gameWishlist       // Added to wishlist
    case gameShare          // Shared with others
    case externalLinkClick  // Clicked Steam/itch.io link
}
```

### 8. Notification Model

In-app notifications for users and developers.

```swift
struct Notification: Identifiable, Codable {
    let id: String
    let userId: String

    let type: NotificationType
    let title: String
    let message: String

    let timestamp: Date
    var isRead: Bool

    // Action context
    var actionURL: String?                  // Deep link
    var gameId: String?
    var developerId: String?
}

enum NotificationType: String, Codable {
    case gameReleased           // Wishlisted game released
    case gameOnSale             // Wishlisted game discounted
    case newFollower            // Someone followed you
    case developerUpdate        // Developer posted update
    case friendRecommendation   // Friend liked a game
    case weeklyDigest           // Curated recommendations
}
```

## Firestore Document References

### Collections Structure

```
users/
  {userId}/
    - User document

developers/
  {developerId}/
    - Developer document

games/
  {gameId}/
    - Game document

interactions/
  {interactionId}/
    - SwipeInteraction document

wishlists/
  {userId}/
    - Wishlist document

tags/
  {tagId}/
    - Tag document

notifications/
  {notificationId}/
    - Notification document

analytics/
  {eventId}/
    - AnalyticsEvent document
```

## Data Validation Rules

### Game Validation

- `title`: 3-100 characters
- `shortDescription`: max 140 characters
- `screenshotURLs`: 2-10 images required
- `genres`: 1-3 required
- `tags`: max 10

### User Validation

- `username`: 3-20 characters, alphanumeric + underscore
- `displayName`: 1-50 characters
- `bio`: max 300 characters
- `favoriteGenres`: 1-5 selections

### Developer Validation

- `studioName`: 2-50 characters
- `bio`: 50-500 characters required
- `publishedGameIds`: max 100 games

## Denormalization Strategy

For performance, we denormalize frequently accessed fields:

```swift
// Instead of fetching Developer for every Game card
struct Game {
    let developerId: String           // Reference
    let developerName: String         // Denormalized
    let studioName: String?           // Denormalized
}

// Update pattern: When developer name changes,
// Cloud Function updates all related games
```

## Timestamps & Dates

All dates use ISO 8601 format and are stored as Firestore Timestamps:

```swift
// SwiftUI/Firebase conversion
extension Date {
    var firestoreTimestamp: Timestamp {
        Timestamp(date: self)
    }
}
```

## Model Extensions

### Computed Properties

```swift
extension Game {
    var isPremium: Bool {
        guard let price = price else { return false }
        return price > 0
    }

    var isReleased: Bool {
        releaseStatus == .released
    }

    var displayPrice: String {
        if isFree { return "Free" }
        guard let price = price else { return "TBA" }
        return "$\(price)"
    }
}

extension User {
    var hasCompletedOnboarding: Bool {
        !favoriteGenres.isEmpty && !preferredPlatforms.isEmpty
    }
}
```

---

**Last Updated**: October 2025  
**Schema Version**: 1.0
