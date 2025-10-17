# Findie iOS App - Architecture Documentation

## Tech Stack Overview

### Core Technologies

- **UI Framework**: SwiftUI (iOS 16+)
- **Language**: Swift 5.9+
- **Backend**: Firebase (Firestore, Auth, Storage, Functions, ML)
- **Async Programming**: Swift Concurrency (async/await) + Combine
- **Dependency Management**: Swift Package Manager (SPM)

### Key Firebase Services

- **Firebase Authentication**: User sign-in and identity management
- **Cloud Firestore**: NoSQL database for games, users, interactions
- **Firebase Storage**: Media hosting (game screenshots, videos, icons)
- **Cloud Functions**: Server-side logic for recommendations and analytics
- **Firebase Analytics**: User behavior tracking and KPI monitoring
- **Firebase ML**: Recommendation model deployment

### Third-Party Integrations

- **Steam API**: Wishlist synchronization
- **itch.io API**: Indie game catalog
- **URLImage/Kingfisher**: Async image loading and caching

## App Architecture

### Pattern: MVVM (Model-View-ViewModel)

```
┌─────────────────────────────────────────┐
│            Views (SwiftUI)              │
│  - DiscoveryView                        │
│  - GameCardView                         │
│  - ProfileView                          │
│  - WishlistView                         │
│  - DeveloperDashboardView               │
└──────────────┬──────────────────────────┘
               │ Bindings (@Published)
┌──────────────▼──────────────────────────┐
│         ViewModels                      │
│  - DiscoveryViewModel                   │
│  - GameCardViewModel                    │
│  - ProfileViewModel                     │
│  - WishlistViewModel                    │
│  - DeveloperViewModel                   │
└──────────────┬──────────────────────────┘
               │ Calls
┌──────────────▼──────────────────────────┐
│         Services Layer                  │
│  - GameService                          │
│  - UserService                          │
│  - RecommendationService                │
│  - WishlistService                      │
│  - AnalyticsService                     │
└──────────────┬──────────────────────────┘
               │ Firebase SDK
┌──────────────▼──────────────────────────┐
│         Firebase Backend                │
│  - Firestore                            │
│  - Auth                                 │
│  - Storage                              │
│  - Functions                            │
└─────────────────────────────────────────┘
```

## Project Structure

```
Findie/
├── App/
│   ├── FindieApp.swift              # App entry point
│   ├── AppDelegate.swift            # Firebase configuration
│   └── Environment.swift            # Environment variables
│
├── Core/
│   ├── Models/
│   │   ├── Game.swift
│   │   ├── User.swift
│   │   ├── Developer.swift
│   │   ├── SwipeInteraction.swift
│   │   └── Wishlist.swift
│   │
│   ├── Services/
│   │   ├── GameService.swift
│   │   ├── UserService.swift
│   │   ├── RecommendationService.swift
│   │   ├── WishlistService.swift
│   │   ├── AnalyticsService.swift
│   │   └── AuthService.swift
│   │
│   └── Utilities/
│       ├── NetworkMonitor.swift
│       ├── ImageCache.swift
│       └── Extensions/
│
├── Features/
│   ├── Discovery/
│   │   ├── Views/
│   │   │   ├── DiscoveryView.swift
│   │   │   ├── GameCardView.swift
│   │   │   └── SwipeOverlayView.swift
│   │   └── ViewModels/
│   │       └── DiscoveryViewModel.swift
│   │
│   ├── Profile/
│   │   ├── Views/
│   │   │   ├── ProfileView.swift
│   │   │   ├── PreferencesView.swift
│   │   │   └── StatsView.swift
│   │   └── ViewModels/
│   │       └── ProfileViewModel.swift
│   │
│   ├── Wishlist/
│   │   ├── Views/
│   │   │   ├── WishlistView.swift
│   │   │   ├── WishlistItemView.swift
│   │   │   └── PlatformSyncView.swift
│   │   └── ViewModels/
│   │       └── WishlistViewModel.swift
│   │
│   ├── Developer/
│   │   ├── Views/
│   │   │   ├── DeveloperDashboardView.swift
│   │   │   ├── GameUploadView.swift
│   │   │   └── AnalyticsView.swift
│   │   └── ViewModels/
│   │       └── DeveloperViewModel.swift
│   │
│   └── Auth/
│       ├── Views/
│       │   ├── LoginView.swift
│       │   ├── SignUpView.swift
│       │   └── OnboardingView.swift
│       └── ViewModels/
│           └── AuthViewModel.swift
│
├── Shared/
│   ├── Components/
│   │   ├── LoadingView.swift
│   │   ├── ErrorView.swift
│   │   ├── TagView.swift
│   │   └── AsyncImageView.swift
│   │
│   └── Theme/
│       ├── Colors.swift
│       ├── Typography.swift
│       └── Layout.swift
│
└── Resources/
    ├── Assets.xcassets
    ├── GoogleService-Info.plist
    └── Localizable.strings
```

## Navigation Flow

### Tab-Based Navigation (Main Container)

```swift
TabView {
    DiscoveryView()
        .tabItem { Label("Discover", systemImage: "gamecontroller") }

    WishlistView()
        .tabItem { Label("Wishlist", systemImage: "heart.fill") }

    ProfileView()
        .tabItem { Label("Profile", systemImage: "person.fill") }
}
```

### Navigation Hierarchy

1. **Discovery Tab** (Root)

   - DiscoveryView → GameDetailView (sheet)
   - DiscoveryView → FilterView (sheet)

2. **Wishlist Tab** (Root)

   - WishlistView → GameDetailView (sheet)
   - WishlistView → PlatformSyncView (sheet)

3. **Profile Tab** (Root)

   - ProfileView → PreferencesView (navigation)
   - ProfileView → StatsView (navigation)
   - ProfileView → DeveloperDashboardView (navigation, if developer)

4. **Auth Flow** (Full-screen cover)
   - LoginView → SignUpView → OnboardingView

## State Management

### ViewModel Pattern

ViewModels use `@Published` properties to communicate with views:

```swift
@MainActor
class DiscoveryViewModel: ObservableObject {
    @Published var games: [Game] = []
    @Published var currentGameIndex: Int = 0
    @Published var isLoading: Bool = false
    @Published var error: Error?

    private let gameService: GameService
    private let recommendationService: RecommendationService

    func loadRecommendedGames() async {
        isLoading = true
        do {
            games = try await recommendationService.fetchPersonalizedGames()
            isLoading = false
        } catch {
            self.error = error
            isLoading = false
        }
    }
}
```

### Singleton Services

Services are instantiated as singletons or environment objects:

```swift
// App.swift
@main
struct FindieApp: App {
    @StateObject private var authService = AuthService.shared

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(authService)
        }
    }
}
```

### Data Flow

1. **View** triggers action (e.g., swipe)
2. **ViewModel** processes action, calls **Service**
3. **Service** interacts with **Firebase**
4. **Service** returns data to **ViewModel**
5. **ViewModel** updates `@Published` properties
6. **View** automatically re-renders via SwiftUI bindings

## Async/Await Pattern

All Firebase operations use Swift Concurrency:

```swift
func fetchGame(id: String) async throws -> Game {
    let document = try await Firestore.firestore()
        .collection("games")
        .document(id)
        .getDocument()

    return try document.data(as: Game.self)
}
```

## Error Handling Strategy

```swift
enum FindieError: LocalizedError {
    case networkError
    case authenticationError
    case dataNotFound
    case invalidData
    case permissionDenied

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
        }
    }
}
```

## Performance Considerations

### Image Loading & Caching

- Use `AsyncImage` with custom caching layer
- Lazy load images in card stack
- Preload next 3 cards in swipe queue

### Data Pagination

- Fetch games in batches of 20
- Implement infinite scroll for wishlist
- Cache game metadata locally (UserDefaults/CoreData)

### Offline Support

- Cache last 50 games locally
- Queue interactions when offline
- Sync when connection restored

### Memory Management

- Dispose of off-screen game cards
- Limit concurrent image downloads (max 5)
- Use weak references in closures

## Security Best Practices

1. **API Keys**: Store Firebase config in `GoogleService-Info.plist`
2. **Authentication**: Require auth for all write operations
3. **Firestore Rules**: Validate user permissions server-side
4. **Data Validation**: Sanitize all user inputs
5. **HTTPS Only**: All network requests use TLS

## Testing Strategy

### Unit Tests

- ViewModel business logic
- Service layer methods
- Model validation

### UI Tests

- Swipe gesture recognition
- Navigation flows
- Critical user journeys

### Integration Tests

- Firebase interactions (using emulator)
- Recommendation algorithm accuracy
- Sync operations

## Build Configurations

### Debug

- Firebase emulator support
- Verbose logging
- Mock data available

### Release

- Production Firebase project
- Analytics enabled
- Crash reporting active

## Deployment

### App Store Distribution

1. TestFlight beta testing (phases 1-3)
2. Staged rollout (10% → 50% → 100%)
3. Monitoring via Firebase Crashlytics

### Backend Deployment

- Cloud Functions: Auto-deploy via GitHub Actions
- Firestore indexes: Deploy via Firebase CLI
- Security rules: Version controlled and tested

---

**Last Updated**: October 2025  
**Target iOS Version**: 16.0+  
**Swift Version**: 5.9+
