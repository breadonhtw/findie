# Findie iOS App - ML Recommendations Documentation

## Overview

Findie's recommendation engine is the core intelligence that matches users with indie games they'll love. It combines collaborative filtering, content-based filtering, and contextual signals to deliver personalized discovery experiences.

---

## Recommendation Strategy

### Hybrid Approach

We use a **hybrid recommendation system** that combines multiple signals:

```
Final Score = (W1 × Collaborative Score) +
              (W2 × Content Score) +
              (W3 × Contextual Score) +
              (W4 × Popularity Score)

Where:
W1 = 0.40 (Collaborative filtering weight)
W2 = 0.30 (Content-based filtering weight)
W3 = 0.20 (Contextual signals weight)
W4 = 0.10 (Popularity/trending weight)
```

### Why Hybrid?

- **Collaborative Filtering**: "Users like you also liked..."
- **Content-Based**: "Because you liked [Game X]..."
- **Contextual**: Time of day, session behavior, trending games
- **Popularity**: Cold start for new users, diversity injection

---

## 1. Collaborative Filtering

### Concept

Find users with similar taste profiles and recommend games they've liked.

### User-User Similarity

Calculate similarity between users based on interaction history:

```python
# User interaction matrix
# Rows = Users, Columns = Games, Values = Interaction scores

User-Game Matrix:
         Game1  Game2  Game3  Game4
User A     5      0      3      0
User B     4      2      0      5
User C     0      0      3      4
User D     5      0      2      5

# Calculate cosine similarity between users
similarity(User A, User D) =
    (5×5 + 0×0 + 3×2 + 0×5) / (√(25+9) × √(25+4+25))
  = 35 / (5.83 × 7.35) = 0.82 (highly similar!)
```

### Interaction Scoring

Different actions have different weights:

| Action                | Score |
| --------------------- | ----- |
| Super Like (Swipe Up) | 10    |
| Wishlist Add          | 8     |
| Like (Swipe Right)    | 5     |
| View Details          | 3     |
| Long View (>30s)      | 2     |
| Skip (Swipe Down)     | 0     |
| Dislike (Swipe Left)  | -3    |

### Item-Item Collaborative Filtering

Alternative approach: "Games similar to games you liked"

```
If User liked Game A and Game B,
Find users who also liked Game A,
Recommend what they liked besides Game B
```

### Implementation

```python
# Firestore Cloud Function (Python)
def calculate_user_similarity(user_id: str, k: int = 50):
    """
    Find k most similar users to given user
    """
    # Fetch user's interaction history
    user_interactions = get_user_interactions(user_id)
    user_vector = build_interaction_vector(user_interactions)

    # Fetch candidate users (who interacted with similar games)
    candidate_users = get_candidate_users(user_interactions)

    # Calculate similarity scores
    similarities = []
    for candidate_id in candidate_users:
        candidate_vector = build_interaction_vector(
            get_user_interactions(candidate_id)
        )
        score = cosine_similarity(user_vector, candidate_vector)
        similarities.append((candidate_id, score))

    # Return top k similar users
    return sorted(similarities, key=lambda x: x[1], reverse=True)[:k]

def recommend_collaborative(user_id: str, n: int = 20):
    """
    Generate collaborative filtering recommendations
    """
    # Find similar users
    similar_users = calculate_user_similarity(user_id, k=50)

    # Get games liked by similar users
    candidate_games = {}
    for similar_user_id, similarity_score in similar_users:
        liked_games = get_liked_games(similar_user_id)
        for game_id, interaction_score in liked_games:
            if game_id not in candidate_games:
                candidate_games[game_id] = 0
            candidate_games[game_id] += interaction_score * similarity_score

    # Remove games already seen by user
    seen_games = get_seen_games(user_id)
    candidate_games = {
        g: s for g, s in candidate_games.items()
        if g not in seen_games
    }

    # Sort and return top n
    recommendations = sorted(
        candidate_games.items(),
        key=lambda x: x[1],
        reverse=True
    )[:n]

    return [game_id for game_id, score in recommendations]
```

---

## 2. Content-Based Filtering

### Concept

Recommend games with similar attributes to games the user has liked.

### Feature Extraction

Extract features from each game:

```python
class GameFeatures:
    # Categorical features
    genres: List[str]           # ['rpg', 'adventure']
    tags: List[str]             # ['pixel-art', 'story-rich']
    platforms: List[str]        # ['windows', 'mac']
    release_status: str         # 'released'

    # Numerical features
    price: float
    view_count: int
    like_count: int
    like_rate: float

    # Text features (TF-IDF vectors)
    description_vector: np.array

    # Developer features
    developer_id: str
    developer_follower_count: int
```

### Feature Encoding

Convert categorical features to numerical vectors:

```python
# One-hot encoding for genres
genres_encoded = [
    1 if 'rpg' in game.genres else 0,
    1 if 'action' in game.genres else 0,
    1 if 'puzzle' in game.genres else 0,
    # ... for all genres
]

# Multi-hot encoding for tags
tags_encoded = [
    1 if 'pixel-art' in game.tags else 0,
    1 if 'roguelike' in game.tags else 0,
    # ... for all tags
]

# Normalize numerical features
price_normalized = game.price / 60.0  # Assuming $60 max
like_rate_normalized = game.like_rate  # Already 0-1
```

### Similarity Calculation

Calculate similarity between games:

```python
def calculate_game_similarity(game_a: Game, game_b: Game):
    """
    Calculate content similarity between two games
    """
    # Genre similarity (Jaccard index)
    genre_similarity = len(set(game_a.genres) & set(game_b.genres)) / \
                       len(set(game_a.genres) | set(game_b.genres))

    # Tag similarity (Jaccard index)
    tag_similarity = len(set(game_a.tags) & set(game_b.tags)) / \
                     len(set(game_a.tags) | set(game_b.tags))

    # Price similarity (inverse absolute difference)
    price_diff = abs(game_a.price - game_b.price)
    price_similarity = 1 / (1 + price_diff / 10)

    # Description similarity (cosine of TF-IDF vectors)
    desc_similarity = cosine_similarity(
        game_a.description_vector,
        game_b.description_vector
    )

    # Weighted combination
    total_similarity = (
        0.35 * genre_similarity +
        0.25 * tag_similarity +
        0.15 * price_similarity +
        0.25 * desc_similarity
    )

    return total_similarity
```

### User Profile Building

Build a profile of what the user likes:

```python
def build_user_profile(user_id: str):
    """
    Build a feature profile based on user's liked games
    """
    liked_games = get_liked_games(user_id)

    # Aggregate genre preferences
    genre_scores = defaultdict(float)
    tag_scores = defaultdict(float)

    for game, interaction_score in liked_games:
        weight = interaction_score / 10.0  # Normalize

        for genre in game.genres:
            genre_scores[genre] += weight

        for tag in game.tags:
            tag_scores[tag] += weight

    # Calculate average price range
    avg_price = np.mean([g.price for g, _ in liked_games])

    return UserProfile(
        preferred_genres=dict(genre_scores),
        preferred_tags=dict(tag_scores),
        avg_price_range=avg_price
    )

def recommend_content_based(user_id: str, n: int = 20):
    """
    Generate content-based recommendations
    """
    user_profile = build_user_profile(user_id)
    seen_games = get_seen_games(user_id)

    # Fetch candidate games
    all_games = get_all_active_games()

    # Score each game
    scored_games = []
    for game in all_games:
        if game.id in seen_games:
            continue

        # Calculate match score
        genre_match = sum(
            user_profile.preferred_genres.get(g, 0)
            for g in game.genres
        )
        tag_match = sum(
            user_profile.preferred_tags.get(t, 0)
            for t in game.tags
        )
        price_match = 1 / (1 + abs(game.price - user_profile.avg_price_range) / 10)

        total_score = (
            0.5 * genre_match +
            0.3 * tag_match +
            0.2 * price_match
        )

        scored_games.append((game.id, total_score))

    # Sort and return top n
    recommendations = sorted(
        scored_games,
        key=lambda x: x[1],
        reverse=True
    )[:n]

    return [game_id for game_id, score in recommendations]
```

---

## 3. Contextual Signals

### Session Context

Track user behavior within current session:

```python
class SessionContext:
    session_id: str
    session_start: datetime
    games_viewed: int
    swipe_velocity: float       # Swipes per minute
    avg_view_duration: float    # Average time on card
    genre_focus: List[str]      # Genres viewed this session

def adjust_for_context(base_score: float, context: SessionContext) -> float:
    """
    Adjust recommendation score based on session context
    """
    score = base_score

    # User is swiping fast → recommend familiar genres
    if context.swipe_velocity > 10:  # >10 swipes/min
        if is_familiar_genre(game, context.genre_focus):
            score *= 1.2

    # User viewing longer → recommend deeper games
    if context.avg_view_duration > 30:  # >30 seconds
        if game.has_tag('story-rich') or game.has_tag('complex'):
            score *= 1.3

    # Session fatigue → boost visually striking games
    if context.games_viewed > 30:
        if game.has_tag('beautiful') or game.has_high_quality_art():
            score *= 1.15

    return score
```

### Time-Based Context

```python
def get_time_boost(game: Game, current_time: datetime) -> float:
    """
    Apply time-based boosts
    """
    hour = current_time.hour
    day_of_week = current_time.weekday()

    boost = 1.0

    # Evening hours → boost story-driven, relaxing games
    if 20 <= hour <= 23:
        if 'narrative' in game.tags or 'cozy' in game.tags:
            boost *= 1.2

    # Weekend → boost multiplayer and longer games
    if day_of_week in [5, 6]:  # Saturday, Sunday
        if 'multiplayer' in game.tags:
            boost *= 1.15

    # Lunch break → boost quick, casual games
    if 12 <= hour <= 13:
        if 'casual' in game.tags or game.avg_play_session < 30:
            boost *= 1.1

    return boost
```

---

## 4. Cold Start Problem

### New User Cold Start

**Problem**: No interaction history for new users.

**Solutions**:

1. **Onboarding Quiz**

   ```python
   def generate_onboarding_recommendations(selected_genres: List[str]):
       """
       Generate initial recommendations based on genre selection
       """
       games = []
       for genre in selected_genres:
           # Get top-rated games in this genre
           top_games = get_games_by_genre(
               genre,
               min_like_rate=0.7,
               min_wishlist_count=100,
               limit=10
           )
           games.extend(top_games)

       # Add variety
       games.extend(get_trending_games(limit=5))
       games.extend(get_editor_picks(limit=5))

       return shuffle(games)
   ```

2. **Popular Defaults**

   - Show universally liked indie games
   - Include award winners and editor's picks
   - Mix genres to discover preferences

3. **Fast Learning**
   - Update recommendations after every 5 interactions
   - Weight early interactions more heavily
   - Encourage diverse exploration

### New Game Cold Start

**Problem**: No interaction data for newly added games.

**Solutions**:

1. **Developer Follower Boost**

   ```python
   if game.is_new and developer.follower_count > 100:
       # Show to developer's followers first
       follower_ids = get_developer_followers(developer.id)
       for follower_id in follower_ids:
           inject_into_feed(follower_id, game.id, position=3)
   ```

2. **Genre/Tag Targeting**

   ```python
   # Target users who like similar games
   similar_games = find_similar_games(new_game, top_k=10)
   for similar_game in similar_games:
       users_who_liked = get_users_who_liked(similar_game.id)
       for user_id in users_who_liked:
           recommend_to_user(user_id, new_game.id)
   ```

3. **A/B Testing Pool**
   - Inject new games into random 10% of users' feeds
   - Collect initial engagement data
   - Use data to refine targeting

---

## 5. Firebase ML Integration

### Model Architecture

```
Input Layer (User + Context Features)
    ↓
Dense Layer (256 units, ReLU)
    ↓
Dropout (0.3)
    ↓
Dense Layer (128 units, ReLU)
    ↓
Dropout (0.2)
    ↓
Dense Layer (64 units, ReLU)
    ↓
Output Layer (Game embeddings)
    ↓
Ranking by cosine similarity
```

### Training Pipeline

```python
# TensorFlow model training (runs on Cloud)
def train_recommendation_model(interaction_data):
    """
    Train neural collaborative filtering model
    """
    # Prepare data
    user_ids = interaction_data['user_id'].values
    game_ids = interaction_data['game_id'].values
    ratings = interaction_data['interaction_score'].values

    # Create embeddings
    user_embedding = Embedding(num_users, embedding_dim=64)
    game_embedding = Embedding(num_games, embedding_dim=64)

    # Model architecture
    user_input = Input(shape=(1,))
    game_input = Input(shape=(1,))

    user_vec = Flatten()(user_embedding(user_input))
    game_vec = Flatten()(game_embedding(game_input))

    concat = Concatenate()([user_vec, game_vec])
    dense1 = Dense(256, activation='relu')(concat)
    dropout1 = Dropout(0.3)(dense1)
    dense2 = Dense(128, activation='relu')(dropout1)
    dropout2 = Dropout(0.2)(dense2)
    dense3 = Dense(64, activation='relu')(dropout2)
    output = Dense(1, activation='sigmoid')(dense3)

    model = Model(inputs=[user_input, game_input], outputs=output)
    model.compile(optimizer='adam', loss='mse', metrics=['mae'])

    # Train
    model.fit(
        [user_ids, game_ids],
        ratings,
        epochs=10,
        batch_size=256,
        validation_split=0.2
    )

    # Convert to TensorFlow Lite
    converter = tf.lite.TFLiteConverter.from_keras_model(model)
    tflite_model = converter.convert()

    return tflite_model
```

### Deployment to Firebase

```bash
# Upload trained model to Firebase ML
firebase ml:models:upload \
  --file recommendation_model.tflite \
  --model-name findie_recommendations_v2

# Deploy Cloud Function to serve predictions
firebase deploy --only functions:generateRecommendations
```

### Inference on Device (Optional)

For offline recommendations, deploy TFLite model to iOS app:

```swift
import TensorFlowLite

class LocalRecommendationEngine {
    private var interpreter: Interpreter?

    func loadModel() throws {
        guard let modelPath = Bundle.main.path(
            forResource: "recommendation_model",
            ofType: "tflite"
        ) else { return }

        interpreter = try Interpreter(modelPath: modelPath)
        try interpreter?.allocateTensors()
    }

    func predict(userId: Int, gameIds: [Int]) throws -> [Float] {
        var scores: [Float] = []

        for gameId in gameIds {
            // Prepare input tensors
            let userInput = [Int32(userId)]
            let gameInput = [Int32(gameId)]

            try interpreter?.copy(Data(bytes: userInput, count: 4), toInputAt: 0)
            try interpreter?.copy(Data(bytes: gameInput, count: 4), toInputAt: 1)

            // Run inference
            try interpreter?.invoke()

            // Get output
            let outputTensor = try interpreter?.output(at: 0)
            let score = outputTensor?.data.toArray(type: Float.self)[0] ?? 0
            scores.append(score)
        }

        return scores
    }
}
```

---

## 6. Diversity & Exploration

### Exploration vs Exploitation

Balance between showing "safe" recommendations and exploring new territory:

```python
def apply_exploration(recommendations: List[str], exploration_rate: float = 0.2):
    """
    Inject exploratory (diverse) games into recommendations
    """
    n_exploit = int(len(recommendations) * (1 - exploration_rate))
    n_explore = len(recommendations) - n_exploit

    # Keep top recommendations (exploitation)
    final_recs = recommendations[:n_exploit]

    # Add diverse games (exploration)
    seen_genres = get_genres_from_games(final_recs)
    unexplored_games = get_games_excluding_genres(seen_genres, limit=100)
    diverse_picks = random.sample(unexplored_games, n_explore)

    final_recs.extend(diverse_picks)

    return shuffle(final_recs)
```

### Genre Diversity

Ensure variety in recommendations:

```python
def enforce_genre_diversity(games: List[Game], max_consecutive: int = 3):
    """
    Reorder games to prevent genre fatigue
    """
    result = []
    genre_streak = defaultdict(int)

    for game in games:
        primary_genre = game.genres[0]

        # Check if we've shown too many of this genre
        if genre_streak[primary_genre] >= max_consecutive:
            # Find next game with different genre
            for other_game in games:
                if other_game.genres[0] != primary_genre:
                    result.append(other_game)
                    genre_streak[other_game.genres[0]] += 1
                    genre_streak[primary_genre] = 0
                    games.remove(other_game)
                    break
        else:
            result.append(game)
            genre_streak[primary_genre] += 1

    return result
```

---

## 7. Model Evaluation

### Offline Metrics

```python
# Precision@K: How many recommendations are relevant?
def precision_at_k(recommendations: List[str], ground_truth: List[str], k: int):
    relevant = set(recommendations[:k]) & set(ground_truth)
    return len(relevant) / k

# Recall@K: What fraction of relevant items are recommended?
def recall_at_k(recommendations: List[str], ground_truth: List[str], k: int):
    relevant = set(recommendations[:k]) & set(ground_truth)
    return len(relevant) / len(ground_truth) if ground_truth else 0

# Mean Average Precision
def mean_average_precision(recommendations: List[str], ground_truth: Set[str]):
    ap = 0.0
    num_relevant = 0

    for k, game_id in enumerate(recommendations, 1):
        if game_id in ground_truth:
            num_relevant += 1
            ap += num_relevant / k

    return ap / len(ground_truth) if ground_truth else 0
```

### Online Metrics (KPIs)

- **Click-Through Rate (CTR)**: % of recommended games that user views details

  - **Target**: ≥ 25%

- **Like Rate**: % of recommended games that user likes

  - **Target**: ≥ 15%

- **Wishlist Conversion**: % of recommended games added to wishlist

  - **Target**: ≥ 8%

- **Session Length**: Average time spent in Discovery feed

  - **Target**: ≥ 10 minutes

- **Discovery Rate**: Average games discovered per session
  - **Target**: ≥ 10 games

---

## 8. A/B Testing Framework

### Experiment Setup

```python
class RecommendationExperiment:
    def __init__(self, name: str, control_algorithm: str, variant_algorithm: str):
        self.name = name
        self.control = control_algorithm
        self.variant = variant_algorithm
        self.user_assignments = {}  # user_id → 'control' or 'variant'

    def assign_user(self, user_id: str) -> str:
        """
        Randomly assign user to control or variant (50/50 split)
        """
        if user_id not in self.user_assignments:
            self.user_assignments[user_id] = random.choice(['control', 'variant'])
        return self.user_assignments[user_id]

    def get_recommendations(self, user_id: str) -> List[str]:
        """
        Get recommendations based on user's assignment
        """
        assignment = self.assign_user(user_id)

        if assignment == 'control':
            return run_algorithm(self.control, user_id)
        else:
            return run_algorithm(self.variant, user_id)
```

### Example Experiments

1. **Collaborative vs Content Weight**

   - Control: 40% collaborative, 30% content
   - Variant: 30% collaborative, 40% content

2. **Diversity Injection Rate**

   - Control: 80% exploitation, 20% exploration
   - Variant: 70% exploitation, 30% exploration

3. **Session Context Impact**
   - Control: No session context adjustments
   - Variant: Apply session context boosts

---

## 9. Performance Optimization

### Caching Strategy

```python
class RecommendationCache:
    def __init__(self):
        self.cache = {}  # user_id → (recommendations, timestamp)
        self.ttl = 6 * 3600  # 6 hours

    def get(self, user_id: str) -> Optional[List[str]]:
        if user_id in self.cache:
            recs, timestamp = self.cache[user_id]
            if time.time() - timestamp < self.ttl:
                return recs
        return None

    def set(self, user_id: str, recommendations: List[str]):
        self.cache[user_id] = (recommendations, time.time())

    def invalidate(self, user_id: str):
        if user_id in self.cache:
            del self.cache[user_id]
```

### Incremental Updates

```python
def update_recommendations_incremental(user_id: str, new_interaction: Interaction):
    """
    Update recommendations after each interaction without full recompute
    """
    cached_recs = cache.get(user_id)

    if new_interaction.action in [SwipeAction.LIKE, SwipeAction.SUPER_LIKE]:
        # Find games similar to the liked game
        similar_games = find_similar_games(new_interaction.game_id, k=10)

        # Inject into cached recommendations
        if cached_recs:
            # Insert similar games at positions 5-15
            for i, game_id in enumerate(similar_games):
                cached_recs.insert(5 + i, game_id)

            cache.set(user_id, cached_recs)
```

### Batch Processing

```python
# Cloud Function scheduled to run daily
def batch_update_all_recommendations():
    """
    Regenerate recommendations for all active users
    """
    active_users = get_active_users(days=7)  # Active in last 7 days

    for user_id in active_users:
        try:
            recommendations = generate_recommendations(user_id, limit=100)
            cache.set(user_id, recommendations)
        except Exception as e:
            log_error(f"Failed to generate recs for {user_id}: {e}")
```

---

## Implementation Timeline

### Phase 1: Basic Recommendations (MVP)

- ✅ Genre-based filtering
- ✅ Simple collaborative filtering
- ✅ Popular game fallbacks

### Phase 2: ML-Powered (Month 3-4)

- 🔜 Train neural collaborative filtering model
- 🔜 Deploy TFLite model to Firebase
- 🔜 Implement hybrid scoring
- 🔜 Add contextual signals

### Phase 3: Advanced (Month 5-6)

- 🔜 A/B testing framework
- 🔜 Real-time model updates
- 🔜 On-device inference
- 🔜 Advanced diversity algorithms

---

**Last Updated**: October 2025  
**Recommendation Engine Version**: 1.0 (Planning)
