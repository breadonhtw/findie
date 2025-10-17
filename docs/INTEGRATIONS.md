# Findie iOS App - Integrations Documentation

## Overview

Findie integrates with multiple third-party services to enrich the game discovery experience, sync wishlists, and provide comprehensive analytics.

---

## 1. Steam Integration

### Purpose

- Sync user wishlists bidirectionally
- Enrich game metadata
- Provide direct purchase links
- Track price changes

### Authentication

#### Steam OpenID

Steam uses OpenID 2.0 for authentication (no OAuth2).

```swift
class SteamAuthService {
    private let steamOpenIDURL = "https://steamcommunity.com/openid/login"

    func initiateAuth(completion: @escaping (Result<URL, Error>) -> Void) {
        var components = URLComponents(string: steamOpenIDURL)
        components?.queryItems = [
            URLQueryItem(name: "openid.ns", value: "http://specs.openid.net/auth/2.0"),
            URLQueryItem(name: "openid.mode", value: "checkid_setup"),
            URLQueryItem(name: "openid.return_to", value: "findie://auth/steam"),
            URLQueryItem(name: "openid.realm", value: "findie://"),
            URLQueryItem(name: "openid.identity", value: "http://specs.openid.net/auth/2.0/identifier_select"),
            URLQueryItem(name: "openid.claimed_id", value: "http://specs.openid.net/auth/2.0/identifier_select")
        ]

        guard let url = components?.url else {
            completion(.failure(FindieError.invalidData))
            return
        }

        completion(.success(url))
    }

    func handleCallback(url: URL) -> String? {
        // Extract Steam ID from callback URL
        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
              let queryItems = components.queryItems else {
            return nil
        }

        // Parse claimed_id to extract Steam ID
        if let claimedId = queryItems.first(where: { $0.name == "openid.claimed_id" })?.value {
            // Extract Steam ID from: https://steamcommunity.com/openid/id/{STEAM_ID}
            let steamId = claimedId.components(separatedBy: "/").last
            return steamId
        }

        return nil
    }
}
```

### Steam Web API

#### API Key Setup

1. Register for Steam API key: https://steamcommunity.com/dev/apikey
2. Store securely in Firebase environment config
3. Never expose in client code

#### Wishlist Endpoint

```swift
class SteamAPIService {
    static let shared = SteamAPIService()
    private let baseURL = "https://api.steampowered.com"

    func fetchWishlist(steamId: String) async throws -> [SteamGame] {
        // Note: Steam wishlist API is not officially public
        // Use store.steampowered.com/wishlist/profiles/{steamId}/wishlistdata
        let url = "https://store.steampowered.com/wishlist/profiles/\(steamId)/wishlistdata/?p=0"

        let (data, _) = try await URLSession.shared.data(from: URL(string: url)!)

        let decoder = JSONDecoder()
        decoder.keyDecodingStrategy = .convertFromSnakeCase

        let wishlistData = try decoder.decode([String: SteamWishlistItem].self, from: data)

        return wishlistData.map { appId, item in
            SteamGame(
                appId: appId,
                name: item.name,
                capsuleURL: item.capsule,
                priority: item.priority,
                added: item.added
            )
        }
    }

    func fetchGameDetails(appId: String) async throws -> SteamGameDetails {
        let url = "\(baseURL)/appdetails/?appids=\(appId)"

        let (data, _) = try await URLSession.shared.data(from: URL(string: url)!)

        let decoder = JSONDecoder()
        decoder.keyDecodingStrategy = .convertFromSnakeCase

        let response = try decoder.decode([String: SteamAppResponse].self, from: data)

        guard let appData = response[appId],
              appData.success,
              let details = appData.data else {
            throw FindieError.dataNotFound
        }

        return details
    }

    func searchGame(title: String) async throws -> [SteamSearchResult] {
        // Use store search API
        let encodedTitle = title.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? ""
        let url = "https://store.steampowered.com/api/storesearch/?term=\(encodedTitle)&cc=US"

        let (data, _) = try await URLSession.shared.data(from: URL(string: url)!)

        let decoder = JSONDecoder()
        let response = try decoder.decode(SteamSearchResponse.self, from: data)

        return response.items
    }
}
```

#### Data Models

```swift
struct SteamGame {
    let appId: String
    let name: String
    let capsuleURL: String  // Cover image
    let priority: Int       // Wishlist priority (0-2)
    let added: Int          // Unix timestamp
}

struct SteamWishlistItem: Codable {
    let name: String
    let capsule: String
    let priority: Int
    let added: Int
}

struct SteamGameDetails: Codable {
    let steamAppid: String
    let name: String
    let type: String
    let shortDescription: String
    let headerImage: String
    let screenshots: [SteamScreenshot]
    let movies: [SteamMovie]?
    let developers: [String]
    let publishers: [String]
    let genres: [SteamGenre]
    let releaseDate: SteamReleaseDate
    let priceOverview: SteamPrice?
}

struct SteamScreenshot: Codable {
    let id: Int
    let pathThumbnail: String
    let pathFull: String
}

struct SteamMovie: Codable {
    let id: Int
    let name: String
    let webm: SteamMovieQuality
}

struct SteamMovieQuality: Codable {
    let max: String  // High quality URL
}

struct SteamGenre: Codable {
    let id: String
    let description: String
}

struct SteamReleaseDate: Codable {
    let comingSoon: Bool
    let date: String
}

struct SteamPrice: Codable {
    let currency: String
    let initial: Int        // Price in cents
    let final: Int          // Discounted price
    let discountPercent: Int
}
```

### Sync Strategy

```swift
class SteamSyncService {
    static let shared = SteamSyncService()

    func syncWishlist(userId: String, steamId: String) async throws -> SyncResult {
        // 1. Fetch Steam wishlist
        let steamGames = try await SteamAPIService.shared.fetchWishlist(steamId: steamId)

        // 2. Fetch user's current Findie wishlist
        let findieWishlist = try await WishlistService.shared.fetchWishlist(userId: userId)

        var added = 0
        var skipped = 0

        // 3. Match Steam games to Findie catalog
        for steamGame in steamGames {
            // Try to find matching game in Findie
            if let findieGame = try? await matchSteamGameToFindie(steamGame: steamGame) {
                // Check if already in wishlist
                if !findieWishlist.items.contains(where: { $0.gameId == findieGame.id }) {
                    // Add to Findie wishlist
                    try await WishlistService.shared.addToWishlist(
                        gameId: findieGame.id,
                        userId: userId
                    )
                    added += 1
                }
            } else {
                skipped += 1
            }
        }

        // 4. Update sync timestamp
        try await updateSyncTimestamp(userId: userId, platform: .steam)

        return SyncResult(added: added, skipped: skipped, platform: .steam)
    }

    private func matchSteamGameToFindie(steamGame: SteamGame) async throws -> Game? {
        // Search Findie catalog by title
        let games = try await GameService.shared.searchGames(
            query: steamGame.name,
            filters: nil
        )

        // Find best match
        // Ideally, games should have steamAppId stored for direct matching
        return games.first { game in
            game.steamURL?.contains(steamGame.appId) ?? false ||
            game.title.lowercased() == steamGame.name.lowercased()
        }
    }
}
```

### Price Tracking

```swift
// Cloud Function (scheduled daily)
async function trackSteamPrices() {
    // Get all wishlisted games with Steam URLs
    const wishlistsSnapshot = await admin.firestore()
        .collection('wishlists')
        .where('steamSyncEnabled', '==', true)
        .get();

    const priceChanges = [];

    for (const wishlistDoc of wishlistsSnapshot.docs) {
        const wishlist = wishlistDoc.data();

        for (const item of wishlist.items) {
            if (item.steamAppId) {
                // Fetch current price from Steam
                const details = await fetchSteamGameDetails(item.steamAppId);

                if (details.priceOverview) {
                    const currentPrice = details.priceOverview.final / 100;

                    // Check if price dropped
                    if (item.currentPrice && currentPrice < item.currentPrice) {
                        priceChanges.push({
                            userId: wishlist.userId,
                            gameId: item.gameId,
                            oldPrice: item.currentPrice,
                            newPrice: currentPrice,
                            discountPercent: details.priceOverview.discountPercent
                        });

                        // Update price in wishlist
                        await updateWishlistPrice(wishlist.userId, item.id, currentPrice);
                    }
                }
            }
        }
    }

    // Send notifications for price drops
    for (const change of priceChanges) {
        await sendPriceDropNotification(change);
    }
}
```

---

## 2. itch.io Integration

### Purpose

- Import games from itch.io
- Sync wishlists
- Support indie developers on itch.io
- Link to purchase pages

### API Limitations

**Note**: itch.io has limited public API. Most integration requires web scraping or manual user input.

#### Available API (Limited)

- Game search (public)
- User profile (public)
- No official wishlist API

### Authentication

itch.io doesn't have OAuth. Users provide username for public data access.

```swift
class ItchioAuthService {
    func validateUsername(_ username: String) async throws -> Bool {
        // Check if user profile exists
        let url = "https://\(username).itch.io"

        let (_, response) = try await URLSession.shared.data(from: URL(string: url)!)

        guard let httpResponse = response as? HTTPURLResponse else {
            return false
        }

        return httpResponse.statusCode == 200
    }
}
```

### Game Search API

```swift
class ItchioAPIService {
    static let shared = ItchioAPIService()
    private let baseURL = "https://itch.io/api/1"

    func searchGames(query: String) async throws -> [ItchioGame] {
        let encodedQuery = query.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? ""
        let url = "https://itch.io/search?q=\(encodedQuery)&format=json"

        let (data, _) = try await URLSession.shared.data(from: URL(string: url)!)

        let decoder = JSONDecoder()
        decoder.keyDecodingStrategy = .convertFromSnakeCase

        let response = try decoder.decode(ItchioSearchResponse.self, from: data)
        return response.games
    }

    func fetchGamesByDeveloper(username: String) async throws -> [ItchioGame] {
        // Scrape developer page (no official API)
        let url = "https://\(username).itch.io"

        let (data, _) = try await URLSession.shared.data(from: URL(string: url)!)

        // Parse HTML (use SwiftSoup or similar)
        // Extract game cards from developer profile

        // This is a simplified example
        // Real implementation would parse HTML structure
        return parseItchioGamesFromHTML(data: data)
    }
}
```

### Data Models

```swift
struct ItchioGame: Codable {
    let id: Int
    let title: String
    let url: String
    let coverUrl: String?
    let shortText: String?
    let price: String?  // e.g., "$9.99" or "Free"
    let author: String
    let published: String?
}

struct ItchioSearchResponse: Codable {
    let games: [ItchioGame]
    let numResults: Int
}
```

### Wishlist Sync (Limited)

```swift
class ItchioSyncService {
    static let shared = ItchioSyncService()

    func syncCollection(userId: String, itchioUsername: String) async throws -> SyncResult {
        // Note: itch.io collections are private by default
        // Users would need to make their collection public
        // Or manually export via CSV

        // Option 1: User exports collection as CSV
        // Option 2: User makes collection public and we scrape
        // Option 3: Manual game-by-game addition

        // For now, implement manual matching
        let itchioGames = try await ItchioAPIService.shared.fetchGamesByDeveloper(username: itchioUsername)

        var added = 0
        var skipped = 0

        for itchioGame in itchioGames {
            if let findieGame = try? await matchItchioGameToFindie(itchioGame: itchioGame) {
                try await WishlistService.shared.addToWishlist(
                    gameId: findieGame.id,
                    userId: userId
                )
                added += 1
            } else {
                skipped += 1
            }
        }

        return SyncResult(added: added, skipped: skipped, platform: .itchio)
    }

    private func matchItchioGameToFindie(itchioGame: ItchioGame) async throws -> Game? {
        let games = try await GameService.shared.searchGames(
            query: itchioGame.title,
            filters: nil
        )

        return games.first { game in
            game.itchioURL?.contains(itchioGame.url) ?? false ||
            game.title.lowercased() == itchioGame.title.lowercased()
        }
    }
}
```

---

## 3. Apple App Store Integration

### Purpose

- App distribution
- In-app purchases (future)
- TestFlight beta testing
- App Store Connect analytics

### App Store Connect API

```swift
import StoreKit

class AppStoreService {
    static let shared = AppStoreService()

    // Request app review
    func requestReview() {
        if let scene = UIApplication.shared.connectedScenes.first as? UIWindowScene {
            SKStoreReviewController.requestReview(in: scene)
        }
    }

    // Open app in App Store
    func openAppStore(appId: String) {
        let url = URL(string: "https://apps.apple.com/app/id\(appId)")!
        UIApplication.shared.open(url)
    }
}
```

### In-App Purchases (Future Feature)

```swift
class IAPService: NSObject, SKProductsRequestDelegate, SKPaymentTransactionObserver {
    static let shared = IAPService()

    private let productIDs = [
        "com.findie.premium.monthly",
        "com.findie.premium.yearly"
    ]

    private var products: [SKProduct] = []

    func fetchProducts() {
        let request = SKProductsRequest(productIdentifiers: Set(productIDs))
        request.delegate = self
        request.start()
    }

    func purchase(productId: String) {
        guard let product = products.first(where: { $0.productIdentifier == productId }) else {
            return
        }

        let payment = SKPayment(product: product)
        SKPaymentQueue.default().add(payment)
    }

    // SKProductsRequestDelegate
    func productsRequest(_ request: SKProductsRequest, didReceive response: SKProductsResponse) {
        products = response.products
    }

    // SKPaymentTransactionObserver
    func paymentQueue(_ queue: SKPaymentQueue, updatedTransactions transactions: [SKPaymentTransaction]) {
        for transaction in transactions {
            switch transaction.transactionState {
            case .purchased:
                // Unlock premium features
                SKPaymentQueue.default().finishTransaction(transaction)
            case .failed:
                SKPaymentQueue.default().finishTransaction(transaction)
            default:
                break
            }
        }
    }
}
```

---

## 4. Analytics Platforms

### Firebase Analytics

```swift
class FirebaseAnalyticsService {
    static let shared = FirebaseAnalyticsService()

    func logEvent(_ name: String, parameters: [String: Any]? = nil) {
        Analytics.logEvent(name, parameters: parameters)
    }

    // Predefined events
    func logGameView(gameId: String, gameName: String) {
        logEvent("view_item", parameters: [
            "item_id": gameId,
            "item_name": gameName,
            "content_type": "game"
        ])
    }

    func logGameLike(gameId: String, gameName: String) {
        logEvent("like_game", parameters: [
            "game_id": gameId,
            "game_name": gameName
        ])
    }

    func logWishlistAdd(gameId: String, gameName: String) {
        logEvent("add_to_wishlist", parameters: [
            "item_id": gameId,
            "item_name": gameName
        ])
    }

    func logShare(gameId: String, method: String) {
        logEvent(AnalyticsEventShare, parameters: [
            "content_type": "game",
            "item_id": gameId,
            "method": method
        ])
    }

    func setUserProperty(name: String, value: String) {
        Analytics.setUserProperty(value, forName: name)
    }
}
```

### Firebase Crashlytics

```swift
import FirebaseCrashlytics

class CrashlyticsService {
    static let shared = CrashlyticsService()

    func log(_ message: String) {
        Crashlytics.crashlytics().log(message)
    }

    func setUserID(_ userId: String) {
        Crashlytics.crashlytics().setUserID(userId)
    }

    func recordError(_ error: Error) {
        Crashlytics.crashlytics().record(error: error)
    }

    func setCustomValue(_ value: Any, forKey key: String) {
        Crashlytics.crashlytics().setCustomValue(value, forKey: key)
    }
}
```

---

## 5. Push Notifications (FCM)

### Firebase Cloud Messaging

```swift
import FirebaseMessaging

class PushNotificationService: NSObject, MessagingDelegate, UNUserNotificationCenterDelegate {
    static let shared = PushNotificationService()

    func setup() {
        UNUserNotificationCenter.current().delegate = self
        Messaging.messaging().delegate = self

        requestAuthorization()
        UIApplication.shared.registerForRemoteNotifications()
    }

    private func requestAuthorization() {
        UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .sound, .badge]) { granted, error in
            if granted {
                print("Notification permission granted")
            }
        }
    }

    // MessagingDelegate
    func messaging(_ messaging: Messaging, didReceiveRegistrationToken fcmToken: String?) {
        guard let token = fcmToken else { return }

        // Save token to Firestore for user
        Task {
            guard let userId = Auth.auth().currentUser?.uid else { return }
            try? await Firestore.firestore()
                .collection("users")
                .document(userId)
                .updateData(["fcmToken": token])
        }
    }

    // UNUserNotificationCenterDelegate
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        willPresent notification: UNNotification,
        withCompletionHandler completionHandler: @escaping (UNNotificationPresentationOptions) -> Void
    ) {
        completionHandler([.banner, .sound, .badge])
    }

    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse,
        withCompletionHandler completionHandler: @escaping () -> Void
    ) {
        let userInfo = response.notification.request.content.userInfo

        // Handle notification tap
        if let gameId = userInfo["gameId"] as? String {
            // Navigate to game detail
            NotificationCenter.default.post(
                name: NSNotification.Name("OpenGameDetail"),
                object: nil,
                userInfo: ["gameId": gameId]
            )
        }

        completionHandler()
    }
}
```

### Send Notifications (Cloud Function)

```javascript
// Cloud Function
const admin = require("firebase-admin");

async function sendGameReleaseNotification(userId, gameId) {
  // Get user's FCM token
  const userDoc = await admin.firestore().collection("users").doc(userId).get();

  const fcmToken = userDoc.data().fcmToken;
  if (!fcmToken) return;

  // Get game details
  const gameDoc = await admin.firestore().collection("games").doc(gameId).get();

  const game = gameDoc.data();

  // Send notification
  const message = {
    token: fcmToken,
    notification: {
      title: "Game Released! 🎮",
      body: `${game.title} is now available!`,
    },
    data: {
      gameId: gameId,
      type: "game_release",
    },
    apns: {
      payload: {
        aps: {
          badge: 1,
          sound: "default",
        },
      },
    },
  };

  await admin.messaging().send(message);
}
```

---

## 6. Deep Linking

### Universal Links

Configure in Xcode and on Firebase Hosting.

**Associated Domains**:

```
applinks:findie.app
applinks:findie-app.web.app
```

**Handle Deep Links**:

```swift
import SwiftUI

struct FindieApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .onOpenURL { url in
                    handleDeepLink(url)
                }
        }
    }

    private func handleDeepLink(_ url: URL) {
        // Parse URL: findie://game/{gameId}
        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false) else {
            return
        }

        let path = components.path

        if path.hasPrefix("/game/") {
            let gameId = String(path.dropFirst("/game/".count))
            // Navigate to game detail
            NotificationCenter.default.post(
                name: NSNotification.Name("OpenGameDetail"),
                object: nil,
                userInfo: ["gameId": gameId]
            )
        }
    }
}
```

---

## 7. External Game Data Enrichment

### IGDB API (Alternative to Steam)

```swift
class IGDBService {
    static let shared = IGDBService()
    private let clientId = "YOUR_TWITCH_CLIENT_ID"
    private let clientSecret = "YOUR_TWITCH_CLIENT_SECRET"
    private var accessToken: String?

    func authenticate() async throws {
        let url = "https://id.twitch.tv/oauth2/token"
        var request = URLRequest(url: URL(string: url)!)
        request.httpMethod = "POST"
        request.setValue("application/x-www-form-urlencoded", forHTTPHeaderField: "Content-Type")

        let body = "client_id=\(clientId)&client_secret=\(clientSecret)&grant_type=client_credentials"
        request.httpBody = body.data(using: .utf8)

        let (data, _) = try await URLSession.shared.data(for: request)
        let response = try JSONDecoder().decode(IGDBAuthResponse.self, from: data)
        accessToken = response.accessToken
    }

    func searchGames(query: String) async throws -> [IGDBGame] {
        guard let token = accessToken else {
            try await authenticate()
            return try await searchGames(query: query)
        }

        let url = "https://api.igdb.com/v4/games"
        var request = URLRequest(url: URL(string: url)!)
        request.httpMethod = "POST"
        request.setValue(clientId, forHTTPHeaderField: "Client-ID")
        request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")

        let body = "search \"\(query)\"; fields name,cover,genres,summary; limit 10;"
        request.httpBody = body.data(using: .utf8)

        let (data, _) = try await URLSession.shared.data(for: request)
        return try JSONDecoder().decode([IGDBGame].self, from: data)
    }
}
```

---

## Integration Testing

### Test Each Integration

```swift
final class IntegrationTests: XCTestCase {
    func testSteamWishlistFetch() async throws {
        let service = SteamAPIService.shared
        let games = try await service.fetchWishlist(steamId: "TEST_STEAM_ID")
        XCTAssertFalse(games.isEmpty)
    }

    func testItchioGameSearch() async throws {
        let service = ItchioAPIService.shared
        let games = try await service.searchGames(query: "celeste")
        XCTAssertTrue(games.contains { $0.title.lowercased().contains("celeste") })
    }

    func testDeepLink() {
        let url = URL(string: "findie://game/12345")!
        // Test deep link handling
    }
}
```

---

## Rate Limiting & Best Practices

### Steam API

- **Rate Limit**: 100,000 calls/day per API key
- **Best Practice**: Cache results, batch requests

### itch.io

- **Rate Limit**: Unofficial, be respectful
- **Best Practice**: Cache aggressively, limit scraping

### Firebase

- **Rate Limit**: Varies by service
- **Best Practice**: Use batching, implement exponential backoff

---

**Last Updated**: October 2025  
**Integrations Status**: Planned
