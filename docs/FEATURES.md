# Findie iOS App - Features Documentation

## Feature Overview

Findie is organized around five core feature pillars that deliver on our mission to make discovering indie games exciting and engaging.

---

## 1. Swipe Discovery Feed

### Overview

The heart of Findie - a Tinder-style card interface where users discover indie games through intuitive gestures.

### User Experience

#### Card Design

- **Full-screen cards** with high-quality cover art
- **Parallax scrolling** for depth perception
- **Auto-playing video previews** when card is focused (muted by default)
- **Smooth animations** with spring physics

#### Information Hierarchy

1. **Primary**: Game cover art (hero image)
2. **Secondary**: Game title, developer name, genre tags
3. **Tertiary**: Short description (140 chars), price, platforms

#### Gesture Controls

| Gesture     | Action     | Outcome                                      |
| ----------- | ---------- | -------------------------------------------- |
| Swipe Right | Like       | Add to liked games, train algorithm          |
| Swipe Left  | Dislike    | Skip game, negative signal for algorithm     |
| Swipe Up    | Super Like | Instant wishlist add, strong positive signal |
| Swipe Down  | Skip       | Neutral skip, minimal algorithm impact       |
| Tap Card    | Details    | Open full game details modal                 |
| Tap Heart   | Wishlist   | Add to wishlist without swiping              |
| Tap Share   | Share      | Share game via iOS share sheet               |

#### Visual Feedback

- **Like**: Green overlay with heart icon
- **Dislike**: Red overlay with X icon
- **Super Like**: Blue overlay with star icon
- **Skip**: Gray fade-out

### Technical Implementation

#### Card Stack Architecture

```swift
struct DiscoveryView: View {
    @StateObject private var viewModel: DiscoveryViewModel
    @State private var dragOffset: CGSize = .zero
    @State private var rotation: Double = 0

    var body: some View {
        ZStack {
            // Background cards (next 2 cards)
            ForEach(viewModel.upcomingGames) { game in
                GameCardView(game: game)
                    .zIndex(Double(game.position))
                    .offset(y: CGFloat(game.position) * 10)
                    .scaleEffect(1 - (Double(game.position) * 0.05))
            }

            // Top card (active)
            if let currentGame = viewModel.currentGame {
                GameCardView(game: currentGame)
                    .offset(dragOffset)
                    .rotationEffect(.degrees(rotation))
                    .gesture(dragGesture)
            }
        }
    }
}
```

#### Smart Prefetching

- Preload next 5 games in queue
- Download images for next 3 cards
- Buffer video trailers for upcoming cards
- Lazy load screenshots on detail view

#### Feed Algorithm

1. **First 10 cards**: Based on user's selected genres during onboarding
2. **Next cards**: ML-powered recommendations using interaction history
3. **Variety injection**: Every 15 cards, inject a random indie gem from different genre
4. **Trending boost**: Include trending games (high engagement in last 7 days)

### Discovery Filters

Users can refine their feed with:

- **Genres**: Multi-select from all available genres
- **Platforms**: Windows, Mac, Linux, Steam, itch.io
- **Price Range**: Free, Under $10, Under $20, Any
- **Release Status**: Released only, Include Early Access, Include Upcoming

### Performance Metrics

- **Target**: 60 FPS during swipes
- **Max load time**: <500ms for next card
- **Image cache**: Keep last 20 viewed cards in memory

---

## 2. Personalized Recommendations

### Overview

ML-powered recommendation engine that learns user preferences and surfaces relevant indie games.

### Recommendation Strategy

#### Collaborative Filtering

- Find users with similar taste profiles
- Recommend games they've liked
- Weight by recency of interactions

#### Content-Based Filtering

- Analyze game attributes (genres, tags, visual style)
- Match against user's interaction history
- Consider developer reputation

#### Hybrid Approach

```
Final Score = (0.6 × Collaborative Score) +
              (0.3 × Content Score) +
              (0.1 × Trending Boost)
```

### Signal Weighting

| Action                  | Weight | Impact               |
| ----------------------- | ------ | -------------------- |
| Super Like (Swipe Up)   | +10    | Very Strong Positive |
| Wishlist Add            | +8     | Strong Positive      |
| Like (Swipe Right)      | +5     | Positive             |
| View Details            | +3     | Mild Positive        |
| View Time >30s          | +2     | Mild Positive        |
| Skip (Swipe Down)       | 0      | Neutral              |
| Dislike (Swipe Left)    | -3     | Negative             |
| View Time <5s + Dislike | -5     | Strong Negative      |

### Cold Start Solutions

#### New Users

1. **Onboarding quiz**: Select 3-5 favorite genres
2. **Popular starters**: Show top-rated games from selected genres
3. **Fast learning**: Update model after first 20 interactions

#### New Games

1. **Developer boost**: Show to followers of developer
2. **Genre matching**: Show to users who like similar genres
3. **Tag similarity**: Match against popular games with similar tags

### Recommendation Refresh

- **Real-time**: Update after every 5 interactions
- **Background**: Regenerate feed every 6 hours
- **Manual**: Pull-to-refresh on Discovery tab

### Diversity & Serendipity

- **80/20 rule**: 80% safe recommendations, 20% exploratory
- **Genre diversity**: Max 3 consecutive games from same genre
- **Developer diversity**: Max 2 consecutive games from same developer

---

## 3. Wishlist Management

### Overview

Curated collection of games users want to play, with cross-platform sync and price tracking.

### Wishlist Features

#### Core Functionality

- Add games from Discovery feed or Search
- Remove games with swipe-to-delete
- Reorder by drag-and-drop
- Set priority levels (High, Medium, Low)
- Add personal notes per game

#### Smart Notifications

- **Release alerts**: Notify when upcoming game releases
- **Price drops**: Notify when wishlisted game goes on sale
- **Platform availability**: Notify when game launches on preferred platform
- **Developer updates**: Notify when developer posts major update

#### Wishlist Views

```swift
enum WishlistView {
    case all          // All wishlisted games
    case byPriority   // Grouped by High/Medium/Low
    case byGenre      // Grouped by genre
    case byPrice      // Grouped by price ranges
    case upcoming     // Only unreleased games
}
```

### Platform Sync

#### Steam Integration

```
1. User authorizes Findie with Steam OpenID
2. Fetch Steam wishlist via Web API
3. Match Steam AppIDs to Findie catalog
4. Two-way sync: Findie ←→ Steam
5. Check for price changes daily
```

#### itch.io Integration

```
1. User provides itch.io username
2. Scrape public wishlist (with user consent)
3. Match games to Findie catalog
4. One-way sync: itch.io → Findie
5. Manual refresh every 24 hours
```

### Wishlist Analytics (For Users)

- Total games wishlisted
- Estimated total cost
- Average discount received
- Games purchased vs. still waiting
- Most wishlisted genres

### Export & Share

- **Export**: CSV/JSON export of wishlist
- **Share**: Generate shareable wishlist link
- **Collaborative wishlists**: Share with friends (Phase 3)

---

## 4. Developer Dashboard

### Overview

Suite of tools for indie developers to upload games, track performance, and understand their audience.

### Game Upload Flow

#### Step 1: Basic Info

- Game title and subtitle
- Short description (140 chars)
- Full description (markdown support)
- Developer/studio name
- Genre selection (1-3 required)
- Custom tags (up to 10)

#### Step 2: Media Assets

- App icon (512×512, required)
- Cover image (1920×1080, required)
- Screenshots (2-10 images, 1920×1080)
- Trailer video (optional, max 2 minutes)
- Drag-and-drop upload with progress indicators

#### Step 3: Release Details

- Release status (Released, Early Access, Upcoming)
- Release date
- Platforms available
- Pricing (Free or paid with amount)
- External links (Steam, itch.io, website, Discord)

#### Step 4: Review & Publish

- Preview game card as users will see it
- Submit for review (if new developer)
- Instant publish (if verified developer)

### Analytics Dashboard

#### Overview Metrics (Cards)

- **Total Views**: Impressions in Discovery feed
- **Like Rate**: % of viewers who liked
- **Wishlist Adds**: Total wishlist additions
- **External Clicks**: Clicks to Steam/itch.io
- **Share Count**: Times shared by users

#### Time-Series Charts

```
Views, Likes, Wishlists over time
- Last 7 days
- Last 30 days
- Last 90 days
- All time
```

#### Audience Insights

- **Top Genres**: What genres do fans like?
- **Platform Breakdown**: Windows vs Mac vs Linux
- **Geographic Distribution**: Where are players located?
- **Discovery Source**: Feed vs Search vs Share

#### Funnel Analysis

```
Discovery View → Detail View → Like → Wishlist → External Click
```

#### Comparison Tools

- Compare with similar games (same genre)
- Benchmark against indie average
- Track rank within genre

### Developer Features

#### Game Updates

- Edit game details anytime
- Add new screenshots/trailer
- Announce updates to followers
- Mark sales/discounts

#### Communication

- Post dev logs (Phase 3)
- Respond to player comments (Phase 3)
- Direct message with superfans (Phase 3)

#### Verification Badge

- Apply for verified status after 100 total wishlists
- Manual review by Findie team
- Badge displayed on game cards and profile

---

## 5. Social Features

### Overview

Community-driven discovery through friend recommendations and social proof.

### Phase 1: Basic Social (MVP)

#### Friend System

- Send/accept friend requests
- View friends' profiles
- See what friends are playing (opt-in)

#### Sharing

- Share game cards via iOS share sheet
- Deep links open game detail in-app
- Share includes card preview image

#### Social Proof on Cards

- "3 friends wishlisted this"
- "Sarah liked this game"
- Display at bottom of game card

### Phase 2: Enhanced Social

#### Activity Feed

- See friends' recent likes and wishlists
- Comment on friends' discoveries
- Like friends' activity

#### Collaborative Wishlists

- Create shared wishlists with friends
- Vote on games to play together
- Plan co-op/multiplayer sessions

#### Following Developers

- Follow favorite indie developers
- Get notified of new releases
- Priority in feed for followed devs

### Phase 3: Community Features

#### Collections

- Curated game collections by theme
- User-created collections (e.g., "Best Pixel Art RPGs")
- Featured collections by Findie team

#### Reviews & Ratings (Light)

- Simple thumbs up/down after playing
- Optional short review (300 chars)
- No numeric ratings (to reduce toxicity)

#### Leaderboards & Gamification

- Discovery streak (days in a row)
- Explorer badges (genres discovered)
- Early adopter badge (wishlist before release)

---

## 6. Search & Browse

### Search Features

#### Quick Search

- Search games by title
- Search developers by studio name
- Instant results as you type
- Recent searches saved

#### Advanced Filters

- Filter by genres, tags, platforms
- Filter by price range
- Filter by release status
- Sort by: Trending, New, Most Wishlisted, Alphabetical

#### Browse by Category

- Trending This Week
- New Releases
- Coming Soon
- Hidden Gems (low views, high like rate)
- Editor's Picks

### Search Algorithm

```
Relevance Score =
  (0.4 × Title Match) +
  (0.2 × Tag Match) +
  (0.2 × Genre Match) +
  (0.1 × Engagement Score) +
  (0.1 × Recency Boost)
```

---

## 7. Profile & Settings

### User Profile

#### Public Profile

- Username and display name
- Avatar image
- Bio (300 chars)
- Stats: Games discovered, liked, wishlisted
- Favorite genres
- Recent activity (if enabled)

#### Privacy Settings

- Make profile public/private
- Allow friend requests
- Share discoveries automatically
- Show activity to friends

### App Settings

#### Notifications

- Push notifications on/off
- Email updates
- Notify on game releases
- Notify on price drops
- Weekly digest email

#### Preferences

- Preferred platforms
- Price range willing to spend
- Content preferences (violent, mature, etc.)

#### Account Management

- Change email/password
- Link/unlink Steam account
- Link/unlink itch.io account
- Export data (GDPR compliance)
- Delete account

---

## 8. Onboarding Flow

### First Launch Experience

#### Screen 1: Welcome

- Findie logo and tagline
- "Find your next obsession — one swipe at a time"
- Continue button

#### Screen 2: Sign Up/Login

- Sign up with email
- Sign in with Apple
- Continue as guest (limited features)

#### Screen 3: Genre Selection

- "What games do you love?"
- Grid of genre cards with icons
- Select 3-5 favorites
- Can skip (will use popular games)

#### Screen 4: Platform Preference

- "Where do you play?"
- Select platforms: Windows, Mac, Linux, etc.
- Multiple selections allowed

#### Screen 5: Price Preference

- "What's your budget?"
- Free only, Under $10, Under $20, Any
- Single selection

#### Screen 6: Tutorial

- Quick 3-card swipe tutorial
- Explain gestures with animations
- Interactive practice swipes

#### Screen 7: Discovery Begins

- Land on Discovery feed with first card
- Show tooltip for first few actions
- Celebration animation after first like

### Returning User Experience

- Automatic login with biometric auth
- Resume at last viewed card
- Show notification badge if new updates

---

## Feature Roadmap by Phase

### Phase 1: MVP (Months 1-2)

- ✅ Swipe Discovery Feed
- ✅ Basic Recommendations (genre-based)
- ✅ Wishlist Core
- ✅ User Profiles
- ✅ Developer Game Upload
- ✅ Basic Analytics
- ✅ Onboarding

### Phase 2: Growth (Months 3-4)

- ✅ ML-Powered Recommendations
- ✅ Platform Sync (Steam, itch.io)
- ✅ Enhanced Developer Dashboard
- ✅ Friend System
- ✅ Search & Browse
- ✅ Push Notifications

### Phase 3: Community (Months 5-6)

- 🔜 Collaborative Wishlists
- 🔜 Activity Feed
- 🔜 User Reviews
- 🔜 Collections
- 🔜 Developer Communication
- 🔜 Gamification

---

**Last Updated**: October 2025  
**Current Phase**: Planning
