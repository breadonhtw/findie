# Findie iOS App - API Services Documentation

## Service Layer Architecture

The service layer abstracts all Firebase and external API interactions, providing a clean interface for ViewModels to consume. Each service is a singleton that handles a specific domain of functionality.

---

## 1. GameService

### Responsibilities

- Fetch games from Firestore
- Upload and manage game data
- Update game metadata
- Handle game media (images, videos)

### Interface

```swift
protocol GameServiceProtocol {
    func fetchGame(id: String) async throws -> Game
    func fetchGames(limit: Int) async throws -> [Game]
    func fetchGamesByDeveloper(developerId: String) async throws -> [Game]
    func fetchGamesByGenres(_ genres: [Genre], limit: Int) async throws -> [Game]
    func searchGames(query: String, filters: GameFilters?) async throws -> [Game]

    func createGame(_ game: Game) async throws -> String
    func updateGame(_ game: Game) async throws
    func deleteGame(id: String) async throws

    func uploadGameImage(_ image: UIImage, gameId: String, type: ImageType) async throws -> String
    func incrementViewCount(gameId: String) async throws
    func incrementLikeCount(gameId: String) async throws
    func incrementWishlistCount(gameId: String, increment: Bool) async throws
}

class GameService: GameServiceProtocol {
    static let shared = GameService()
    private let db = Firestore.firestore()
    private let storage = Storage.storage()

    private init() {}
}
```

### Implementation Examples

#### Fetch Game by ID

```swift
func fetchGame(id: String) async throws -> Game {
    let document = try await db.collection("games")
        .document(id)
        .getDocument()

    guard document.exists else {
        throw FindieError.dataNotFound
    }

    return try document.data(as: Game.self)
}
```

#### Search Games with Filters

```swift
func searchGames(query: String, filters: GameFilters?) async throws -> [Game] {
    var queryRef = db.collection("games")
        .whereField("isActive", isEqualTo: true)

    // Apply filters
    if let genres = filters?.genres, !genres.isEmpty {
        queryRef = queryRef.whereField("genres", arrayContainsAny: genres.map { $0.rawValue })
    }

    if let maxPrice = filters?.maxPrice {
        queryRef = queryRef.whereField("price", isLessThanOrEqualTo: maxPrice)
    }

    if let platforms = filters?.platforms {
        queryRef = queryRef.whereField("platforms", arrayContainsAny: platforms.map { $0.rawValue })
    }

    let snapshot = try await queryRef.limit(to: 50).getDocuments()

    var games = snapshot.documents.compactMap { try? $0.data(as: Game.self) }

    // Client-side title filtering (Firestore doesn't support text search)
    if !query.isEmpty {
        games = games.filter {
            $0.title.localizedCaseInsensitiveContains(query)
        }
    }

    return games
}
```

#### Upload Game Image

```swift
func uploadGameImage(_ image: UIImage, gameId: String, type: ImageType) async throws -> String {
    guard let imageData = image.jpegData(compressionQuality: 0.8) else {
        throw FindieError.invalidData
    }

    let path = "games/\(gameId)/\(type.filename)"
    let storageRef = storage.reference().child(path)

    let metadata = StorageMetadata()
    metadata.contentType = "image/jpeg"

    _ = try await storageRef.putDataAsync(imageData, metadata: metadata)
    let downloadURL = try await storageRef.downloadURL()

    return downloadURL.absoluteString
}
```

### Data Models

```swift
struct GameFilters {
    var genres: [Genre]?
    var platforms: [Platform]?
    var minPrice: Decimal?
    var maxPrice: Decimal?
    var releaseStatus: [ReleaseStatus]?
}

enum ImageType {
    case icon, cover, screenshot(Int), banner

    var filename: String {
        switch self {
        case .icon: return "icon.jpg"
        case .cover: return "cover.jpg"
        case .screenshot(let index): return "screenshot_\(index).jpg"
        case .banner: return "banner.jpg"
        }
    }
}
```

---

## 2. UserService

### Responsibilities

- Manage user profiles
- Update user preferences
- Track user stats and engagement
- Handle friend relationships

### Interface

```swift
protocol UserServiceProtocol {
    func fetchUser(id: String) async throws -> User
    func fetchCurrentUser() async throws -> User?
    func createUser(_ user: User) async throws
    func updateUser(_ user: User) async throws
    func updatePreferences(_ preferences: UserPreferences) async throws

    func incrementGamesDiscovered(userId: String) async throws
    func incrementGamesLiked(userId: String) async throws
    func incrementGamesWishlisted(userId: String, increment: Bool) async throws

    func sendFriendRequest(to userId: String) async throws
    func acceptFriendRequest(from userId: String) async throws
    func removeFriend(userId: String) async throws
    func fetchFriends(userId: String) async throws -> [User]

    func updateAvatar(_ image: UIImage) async throws -> String
}

class UserService: UserServiceProtocol {
    static let shared = UserService()
    private let db = Firestore.firestore()
    private let storage = Storage.storage()

    private init() {}
}
```

### Implementation Examples

#### Fetch Current User

```swift
func fetchCurrentUser() async throws -> User? {
    guard let currentUserId = Auth.auth().currentUser?.uid else {
        return nil
    }

    return try await fetchUser(id: currentUserId)
}
```

#### Update User Preferences

```swift
func updatePreferences(_ preferences: UserPreferences) async throws {
    guard let userId = Auth.auth().currentUser?.uid else {
        throw FindieError.authenticationError
    }

    try await db.collection("users").document(userId).updateData([
        "favoriteGenres": preferences.favoriteGenres.map { $0.rawValue },
        "preferredPlatforms": preferences.preferredPlatforms.map { $0.rawValue },
        "priceRange": preferences.priceRange.rawValue,
        "updatedAt": FieldValue.serverTimestamp()
    ])
}
```

#### Send Friend Request

```swift
func sendFriendRequest(to userId: String) async throws {
    guard let currentUserId = Auth.auth().currentUser?.uid else {
        throw FindieError.authenticationError
    }

    // Create friend request notification
    let notification = Notification(
        id: UUID().uuidString,
        userId: userId,
        type: .newFollower,
        title: "New Friend Request",
        message: "Someone wants to connect with you",
        timestamp: Date(),
        isRead: false,
        actionURL: "findie://profile/\(currentUserId)"
    )

    try await db.collection("notifications")
        .document(notification.id)
        .setData(from: notification)
}
```

---

## 3. RecommendationService

### Responsibilities

- Generate personalized game recommendations
- Fetch trending games
- Handle ML model interactions
- Cache recommendation results

### Interface

```swift
protocol RecommendationServiceProtocol {
    func fetchPersonalizedGames(userId: String, limit: Int) async throws -> [Game]
    func fetchTrendingGames(limit: Int) async throws -> [Game]
    func fetchSimilarGames(to gameId: String, limit: Int) async throws -> [Game]
    func fetchNewReleases(limit: Int) async throws -> [Game]
    func refreshRecommendations(userId: String) async throws
}

class RecommendationService: RecommendationServiceProtocol {
    static let shared = RecommendationService()
    private let db = Firestore.firestore()
    private let functions = Functions.functions()
    private let cache = RecommendationCache()

    private init() {}
}
```

### Implementation Examples

#### Fetch Personalized Games

```swift
func fetchPersonalizedGames(userId: String, limit: Int) async throws -> [Game] {
    // Check cache first
    if let cached = cache.get(for: userId), !cached.isExpired {
        return Array(cached.games.prefix(limit))
    }

    // Call Cloud Function to generate recommendations
    let callable = functions.httpsCallable("generateRecommendations")
    let result = try await callable.call(["userId": userId, "limit": limit])

    guard let data = result.data as? [String: Any],
          let gameIds = data["gameIds"] as? [String] else {
        throw FindieError.invalidData
    }

    // Fetch game details
    var games: [Game] = []
    for gameId in gameIds {
        if let game = try? await GameService.shared.fetchGame(id: gameId) {
            games.append(game)
        }
    }

    // Cache results
    cache.set(games, for: userId, expiresIn: 6 * 3600) // 6 hours

    return games
}
```

#### Fetch Trending Games

```swift
func fetchTrendingGames(limit: Int) async throws -> [Game] {
    // Trending = high engagement in last 7 days
    let sevenDaysAgo = Calendar.current.date(byAdding: .day, value: -7, to: Date())!

    let snapshot = try await db.collection("games")
        .whereField("isActive", isEqualTo: true)
        .whereField("updatedAt", isGreaterThan: Timestamp(date: sevenDaysAgo))
        .order(by: "wishlistCount", descending: true)
        .limit(to: limit)
        .getDocuments()

    return snapshot.documents.compactMap { try? $0.data(as: Game.self) }
}
```

---

## 4. WishlistService

### Responsibilities

- Manage user wishlists
- Add/remove games from wishlist
- Sync with external platforms (Steam, itch.io)
- Track price changes

### Interface

```swift
protocol WishlistServiceProtocol {
    func fetchWishlist(userId: String) async throws -> Wishlist
    func addToWishlist(gameId: String, userId: String) async throws
    func removeFromWishlist(gameId: String, userId: String) async throws
    func updateWishlistItem(_ item: WishlistItem, userId: String) async throws

    func syncSteamWishlist(userId: String, steamId: String) async throws -> SyncResult
    func syncItchioWishlist(userId: String, itchioUsername: String) async throws -> SyncResult

    func checkPriceDrops(userId: String) async throws -> [PriceChange]
}

class WishlistService: WishlistServiceProtocol {
    static let shared = WishlistService()
    private let db = Firestore.firestore()
    private let functions = Functions.functions()

    private init() {}
}
```

### Implementation Examples

#### Add to Wishlist

```swift
func addToWishlist(gameId: String, userId: String) async throws {
    let game = try await GameService.shared.fetchGame(id: gameId)

    let item = WishlistItem(
        id: UUID().uuidString,
        gameId: gameId,
        gameTitle: game.title,
        gameIconURL: game.iconURL,
        developerName: game.developerName,
        addedDate: Date(),
        priority: .medium,
        notifyOnRelease: game.releaseStatus == .upcoming,
        notifyOnDiscount: !game.isFree,
        priceAtAdd: game.price,
        currentPrice: game.price,
        isOnSale: false,
        releaseDate: game.releaseDate
    )

    try await db.collection("wishlists").document(userId).updateData([
        "items": FieldValue.arrayUnion([try Firestore.Encoder().encode(item)]),
        "lastUpdated": FieldValue.serverTimestamp()
    ])

    // Increment game's wishlist count
    try await GameService.shared.incrementWishlistCount(gameId: gameId, increment: true)

    // Increment user's stat
    try await UserService.shared.incrementGamesWishlisted(userId: userId, increment: true)
}
```

#### Sync Steam Wishlist

```swift
func syncSteamWishlist(userId: String, steamId: String) async throws -> SyncResult {
    let callable = functions.httpsCallable("syncSteamWishlist")
    let result = try await callable.call([
        "userId": userId,
        "steamId": steamId
    ])

    guard let data = result.data as? [String: Any],
          let added = data["added"] as? Int,
          let skipped = data["skipped"] as? Int else {
        throw FindieError.invalidData
    }

    return SyncResult(added: added, skipped: skipped, platform: .steam)
}
```

### Data Models

```swift
struct SyncResult {
    let added: Int
    let skipped: Int
    let platform: Platform
}

struct PriceChange {
    let game: Game
    let oldPrice: Decimal
    let newPrice: Decimal
    let discountPercent: Int
}
```

---

## 5. AnalyticsService

### Responsibilities

- Track user interactions
- Log events for developers
- Monitor KPIs
- Handle Firebase Analytics integration

### Interface

```swift
protocol AnalyticsServiceProtocol {
    func logGameView(gameId: String, userId: String, sessionId: String) async throws
    func logSwipeAction(gameId: String, userId: String, action: SwipeAction, viewDuration: TimeInterval) async throws
    func logGameDetail(gameId: String, userId: String) async throws
    func logExternalLinkClick(gameId: String, userId: String, platform: Platform) async throws

    func fetchGameAnalytics(gameId: String, dateRange: DateRange) async throws -> GameAnalytics
    func fetchDeveloperAnalytics(developerId: String, dateRange: DateRange) async throws -> DeveloperAnalytics

    func trackKPI(_ event: KPIEvent) async throws
}

class AnalyticsService: AnalyticsServiceProtocol {
    static let shared = AnalyticsService()
    private let db = Firestore.firestore()
    private let analytics = Analytics.self

    private init() {}
}
```

### Implementation Examples

#### Log Swipe Action

```swift
func logSwipeAction(
    gameId: String,
    userId: String,
    action: SwipeAction,
    viewDuration: TimeInterval
) async throws {
    // Get user preferences for ML context
    let user = try await UserService.shared.fetchUser(id: userId)
    let game = try await GameService.shared.fetchGame(id: gameId)

    let interaction = SwipeInteraction(
        id: UUID().uuidString,
        userId: userId,
        gameId: gameId,
        action: action,
        timestamp: Date(),
        sessionId: SessionManager.shared.currentSessionId,
        cardPosition: 0, // Set by ViewModel
        viewDuration: viewDuration,
        userGenrePreferences: user.favoriteGenres,
        gameGenres: game.genres,
        gameTags: game.tags
    )

    // Store in Firestore
    try await db.collection("interactions")
        .document(interaction.id)
        .setData(from: interaction)

    // Log to Firebase Analytics
    analytics.logEvent("swipe_action", parameters: [
        "game_id": gameId,
        "action": action.rawValue,
        "view_duration": viewDuration
    ])
}
```

#### Fetch Game Analytics

```swift
func fetchGameAnalytics(gameId: String, dateRange: DateRange) async throws -> GameAnalytics {
    let startDate = dateRange.startDate
    let endDate = dateRange.endDate

    let snapshot = try await db.collection("analytics")
        .whereField("gameId", isEqualTo: gameId)
        .whereField("timestamp", isGreaterThanOrEqualTo: Timestamp(date: startDate))
        .whereField("timestamp", isLessThanOrEqualTo: Timestamp(date: endDate))
        .getDocuments()

    let events = snapshot.documents.compactMap { try? $0.data(as: AnalyticsEvent.self) }

    // Aggregate metrics
    let views = events.filter { $0.eventType == .gameView }.count
    let likes = events.filter { $0.eventType == .gameLike }.count
    let wishlists = events.filter { $0.eventType == .gameWishlist }.count
    let shares = events.filter { $0.eventType == .gameShare }.count
    let clicks = events.filter { $0.eventType == .externalLinkClick }.count

    let likeRate = views > 0 ? Double(likes) / Double(views) : 0.0

    return GameAnalytics(
        gameId: gameId,
        views: views,
        likes: likes,
        wishlists: wishlists,
        shares: shares,
        externalClicks: clicks,
        likeRate: likeRate,
        dateRange: dateRange
    )
}
```

### Data Models

```swift
struct GameAnalytics {
    let gameId: String
    let views: Int
    let likes: Int
    let wishlists: Int
    let shares: Int
    let externalClicks: Int
    let likeRate: Double
    let dateRange: DateRange
}

struct DeveloperAnalytics {
    let developerId: String
    let totalViews: Int
    let totalLikes: Int
    let totalWishlists: Int
    let topGames: [GameAnalytics]
    let dateRange: DateRange
}

struct DateRange {
    let startDate: Date
    let endDate: Date

    static var last7Days: DateRange {
        DateRange(
            startDate: Calendar.current.date(byAdding: .day, value: -7, to: Date())!,
            endDate: Date()
        )
    }

    static var last30Days: DateRange {
        DateRange(
            startDate: Calendar.current.date(byAdding: .day, value: -30, to: Date())!,
            endDate: Date()
        )
    }
}

enum KPIEvent {
    case userSignup
    case dailyActive
    case weeklyActive
    case gameDiscovery
    case wishlistSync
}
```

---

## 6. AuthService

### Responsibilities

- Handle user authentication
- Manage auth state
- Sign in/sign up flows
- Session management

### Interface

```swift
protocol AuthServiceProtocol {
    var currentUser: Firebase.User? { get }
    var isAuthenticated: Bool { get }

    func signUp(email: String, password: String, username: String) async throws -> User
    func signIn(email: String, password: String) async throws -> User
    func signInWithApple(credential: AuthCredential) async throws -> User
    func signInAnonymously() async throws -> User
    func signOut() throws

    func sendPasswordReset(email: String) async throws
    func updateEmail(_ email: String) async throws
    func updatePassword(_ password: String) async throws

    func deleteAccount() async throws
}

class AuthService: ObservableObject, AuthServiceProtocol {
    static let shared = AuthService()

    @Published var currentUser: Firebase.User?
    @Published var isAuthenticated: Bool = false

    private let auth = Auth.auth()

    private init() {
        currentUser = auth.currentUser
        isAuthenticated = currentUser != nil

        // Listen for auth state changes
        auth.addStateDidChangeListener { [weak self] _, user in
            self?.currentUser = user
            self?.isAuthenticated = user != nil
        }
    }
}
```

### Implementation Examples

#### Sign Up with Email

```swift
func signUp(email: String, password: String, username: String) async throws -> User {
    // Create Firebase Auth user
    let authResult = try await auth.createUser(withEmail: email, password: password)

    // Create Firestore user document
    let user = User(
        id: authResult.user.uid,
        username: username,
        email: email,
        displayName: username,
        accountType: .player,
        gamesDiscovered: 0,
        gamesLiked: 0,
        gamesWishlisted: 0,
        joinedDate: Date(),
        lastActiveDate: Date(),
        friendIds: [],
        followerCount: 0,
        followingCount: 0,
        isProfilePublic: true,
        allowFriendRequests: true,
        favoriteGenres: [],
        preferredPlatforms: [],
        priceRange: .any,
        notificationsEnabled: true,
        emailUpdates: false,
        shareDiscoveries: false
    )

    try await UserService.shared.createUser(user)

    // Send verification email
    try await authResult.user.sendEmailVerification()

    return user
}
```

#### Sign In with Apple

```swift
func signInWithApple(credential: AuthCredential) async throws -> User {
    let authResult = try await auth.signIn(with: credential)

    // Check if user document exists
    if let existingUser = try? await UserService.shared.fetchUser(id: authResult.user.uid) {
        return existingUser
    }

    // Create new user document
    let user = User(
        id: authResult.user.uid,
        username: authResult.user.displayName ?? "User\(String(authResult.user.uid.prefix(6)))",
        email: authResult.user.email ?? "",
        displayName: authResult.user.displayName ?? "Anonymous",
        accountType: .player,
        // ... rest of user initialization
    )

    try await UserService.shared.createUser(user)

    return user
}
```

---

## 7. External API Integration Services

### SteamAPIService

```swift
class SteamAPIService {
    static let shared = SteamAPIService()
    private let apiKey: String
    private let baseURL = "https://api.steampowered.com"

    func fetchWishlist(steamId: String) async throws -> [SteamGame] {
        let url = "\(baseURL)/IWishlistService/GetWishlist/v1/?steamid=\(steamId)&key=\(apiKey)"
        // Implement API call
    }

    func fetchGameDetails(appId: String) async throws -> SteamGame {
        let url = "\(baseURL)/ISteamApps/GetAppDetails/v1/?appids=\(appId)"
        // Implement API call
    }
}
```

### ItchioAPIService

```swift
class ItchioAPIService {
    static let shared = ItchioAPIService()
    private let baseURL = "https://itch.io/api/1"

    func fetchGamesByDeveloper(username: String) async throws -> [ItchioGame] {
        let url = "\(baseURL)/\(username)/games"
        // Implement API call (public API)
    }
}
```

---

## Error Handling

All services use a common error type:

```swift
enum FindieError: LocalizedError {
    case networkError
    case authenticationError
    case dataNotFound
    case invalidData
    case permissionDenied
    case rateLimitExceeded

    var errorDescription: String? {
        switch self {
        case .networkError:
            return "Unable to connect. Check your internet connection."
        case .authenticationError:
            return "Please sign in to continue."
        case .dataNotFound:
            return "The requested content could not be found."
        case .invalidData:
            return "The data received is invalid."
        case .permissionDenied:
            return "You don't have permission to access this content."
        case .rateLimitExceeded:
            return "Too many requests. Please try again later."
        }
    }
}
```

---

## Service Testing

Each service should have corresponding unit tests:

```swift
final class GameServiceTests: XCTestCase {
    var sut: GameService!

    override func setUp() {
        super.setUp()
        sut = GameService.shared
    }

    func testFetchGame() async throws {
        let game = try await sut.fetchGame(id: "test_game_id")
        XCTAssertNotNil(game)
        XCTAssertEqual(game.id, "test_game_id")
    }
}
```

---

**Last Updated**: October 2025  
**Service Architecture**: Singleton Pattern with Protocol-Oriented Design
