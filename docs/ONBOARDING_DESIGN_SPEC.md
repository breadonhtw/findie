# Findie Onboarding - Design Specification

## Overview

This document provides detailed specifications for implementing the Findie onboarding and authentication flow based on the design inspiration.

---

## Design Principles

1. **Dark theme** - Create immersive, game-focused experience
2. **Visual depth** - Blurred indie game backgrounds for context
3. **Simplicity** - Clear, minimal interface with focused actions
4. **Brand consistency** - Use findie logo and tagline throughout
5. **Accessibility** - High contrast, readable text, clear touch targets

---

## Color Palette

### Background
- **Primary Background**: `#1A1A1A` (dark charcoal)
- **Background Overlay**: `rgba(0, 0, 0, 0.6)` (60% black overlay on game images)

### Text
- **Primary Text**: `#FFFFFF` (pure white)
- **Secondary Text**: `#B8B8B8` (light gray)
- **Link Text**: `#6B9FFF` (light blue)

### Buttons
- **Primary Button Background**: `#FFFFFF` (white)
- **Primary Button Text**: `#1A1A1A` (dark)
- **Secondary Button Border**: `#FFFFFF` (white)
- **Secondary Button Text**: `#FFFFFF` (white)
- **Secondary Button Background**: `rgba(255, 255, 255, 0.1)` (10% white)

### Inputs
- **Input Background**: `#FFFFFF` (white)
- **Input Text**: `#1A1A1A` (dark)
- **Input Placeholder**: `#999999` (medium gray)
- **Input Border**: `#E0E0E0` (light gray)
- **Input Border Focused**: `#6B9FFF` (light blue)

---

## Typography

### Font Family
- **Primary**: San Francisco (iOS system font)
- **Display**: SF Pro Display (for logo)
- **Text**: SF Pro Text (for body)

### Text Styles

```swift
// Logo
.font(.system(size: 32, weight: .bold, design: .rounded))

// Tagline
.font(.system(size: 24, weight: .medium))

// Button Text
.font(.system(size: 17, weight: .semibold))

// Input Label
.font(.system(size: 13, weight: .medium))

// Body Text
.font(.system(size: 15, weight: .regular))

// Small Text (Terms)
.font(.system(size: 11, weight: .regular))

// Link Text
.font(.system(size: 13, weight: .medium))
```

---

## Layout Specifications

### Spacing
- **Screen Padding**: 24pt (horizontal), 32pt (vertical)
- **Section Spacing**: 32pt (between major sections)
- **Element Spacing**: 16pt (between related elements)
- **Button Stack Spacing**: 12pt (between stacked buttons)
- **Input Stack Spacing**: 16pt (between input fields)

### Component Sizes
- **Button Height**: 52pt
- **Input Field Height**: 48pt
- **Logo Size**: 120pt × 120pt (icon + text)
- **Back Button Size**: 44pt × 44pt (touch target)
- **Provider Icon Size**: 20pt × 20pt

### Safe Areas
- Respect iOS safe area insets
- Bottom content should be 24pt above safe area bottom
- Top content should respect notch/status bar

---

## Screen 1: Welcome Screen

### Layout
```
┌─────────────────────────────────┐
│                                 │
│         [Blurred Game BG]       │
│                                 │
│         [findie logo]           │
│                                 │
│                                 │
│                                 │
│                                 │
│      [GET STARTED Button]       │
│      [SIGN IN Button]           │
│                                 │
│   [Terms & Privacy Links]       │
└─────────────────────────────────┘
```

### Components

#### Background
- Blurred indie game screenshot (randomly selected)
- 60% black overlay for readability
- Gaussian blur radius: 20pt

#### Logo
- Position: Centered, 25% from top
- Icon above text
- White color

#### Buttons
1. **GET STARTED** (Primary)
   - Full width (screen width - 48pt)
   - Height: 52pt
   - Background: White
   - Text: Dark (#1A1A1A)
   - Border radius: 12pt
   - Shadow: 0 4pt 12pt rgba(0, 0, 0, 0.15)

2. **SIGN IN** (Secondary)
   - Full width (screen width - 48pt)
   - Height: 52pt
   - Background: rgba(255, 255, 255, 0.1)
   - Border: 1.5pt white
   - Text: White
   - Border radius: 12pt

#### Terms Text
- Position: Bottom, 24pt from safe area
- Font size: 11pt
- Color: #B8B8B8
- Text: "By creating an account, you agree to our Terms. Learn how we process your data in our Privacy Policy and Cookies Policy."
- Links: Underlined, #6B9FFF

### Interactions
- Tap "GET STARTED" → Navigate to Screen 2
- Tap "SIGN IN" → Navigate to Screen 3

---

## Screen 2: Sign-Up Options

### Layout
```
┌─────────────────────────────────┐
│ [←]                             │
│         [Blurred Game BG]       │
│                                 │
│         [findie logo]           │
│                                 │
│    "Play beyond the             │
│     mainstream."                │
│                                 │
│   [🍎 Sign up with Apple]       │
│   [G Continue with Google]      │
│   [Sign up with email]          │
│                                 │
│   [Terms & Privacy Links]       │
└─────────────────────────────────┘
```

### Components

#### Back Button
- Position: Top-left, 16pt from edges
- Icon: Left chevron (<)
- Size: 44pt × 44pt (touch target)
- Icon size: 20pt
- Color: White
- Background: rgba(255, 255, 255, 0.15)
- Border radius: 22pt (circular)

#### Logo
- Position: Top-center, below back button
- Same styling as Screen 1
- Smaller size: 80pt total height

#### Tagline
- Position: Below logo, 24pt spacing
- Text: "Play beyond the mainstream."
- Font: 24pt, medium weight
- Color: White
- Text alignment: Center
- Line height: 1.2

#### Sign-Up Buttons
All buttons:
- Full width (screen width - 48pt)
- Height: 52pt
- Background: White
- Border radius: 12pt
- Text alignment: Left-aligned with icon
- Icon size: 20pt
- Spacing between icon and text: 12pt

1. **Sign up with Apple**
   - Apple logo icon (black)
   - Text: "Sign up with Apple"
   - Text color: Black

2. **Continue with Google**
   - Google "G" logo icon (multicolor)
   - Text: "Continue with Google"
   - Text color: Black

3. **Sign up with email**
   - No icon
   - Text: "Sign up with email"
   - Text color: Black
   - Text alignment: Center

#### Terms Text
- Same as Screen 1

### Interactions
- Tap back arrow → Navigate back to Screen 1
- Tap "Sign up with Apple" → Trigger Apple Sign In
- Tap "Continue with Google" → Trigger Google Sign In
- Tap "Sign up with email" → Navigate to email sign-up form
- Links in terms → Open in Safari/WebView

---

## Screen 3: Email/Password Login

### Layout
```
┌─────────────────────────────────┐
│ [←]                             │
│         [Blurred Game BG]       │
│                                 │
│         [findie logo]           │
│                                 │
│    "Play beyond the             │
│     mainstream."                │
│                                 │
│         [email input]           │
│         [password input]        │
│                                 │
│     "I forgot my password"      │
│                                 │
│         [Log in Button]         │
│                                 │
│   [Terms & Privacy Links]       │
└─────────────────────────────────┘
```

### Components

#### Back Button
- Same as Screen 2

#### Logo & Tagline
- Same as Screen 2

#### Input Fields
Both inputs:
- Full width (screen width - 48pt)
- Height: 48pt
- Background: White
- Border: 1pt #E0E0E0
- Border radius: 8pt
- Padding: 16pt horizontal
- Font size: 16pt (to prevent zoom on iOS)

1. **Email Input**
   - Label: "email" (inside field, top-left as placeholder)
   - Keyboard type: Email address
   - Autocorrect: Off
   - Autocapitalization: None
   - Text content type: emailAddress

2. **Password Input**
   - Label: "password" (inside field, top-left as placeholder)
   - Secure text entry: Yes
   - Eye icon (show/hide password): Right side
   - Text content type: password

#### Forgot Password Link
- Position: Below password input, 8pt spacing
- Text: "I forgot my password"
- Font: 13pt, medium weight
- Color: White
- Text alignment: Right
- Underlined on hover/press

#### Log In Button
- Full width (screen width - 48pt)
- Height: 52pt
- Background: White
- Text: "Log in"
- Text color: Black
- Border radius: 12pt
- Font: 17pt, semibold
- Disabled state: 50% opacity

#### Terms Text
- Same as Screen 1 & 2

### Input States

#### Default
- Border: 1pt #E0E0E0
- Background: White
- Placeholder: #999999

#### Focused
- Border: 2pt #6B9FFF
- Background: White
- Placeholder: Fade out
- Label: Animate to top-left inside field

#### Error
- Border: 2pt #FF453A (red)
- Background: White
- Error message below: Red text, 13pt

#### Filled
- Border: 1pt #E0E0E0
- Background: White
- Text: Black

### Interactions
- Tap back arrow → Navigate to Screen 2
- Enter email → Validate format
- Enter password → Enable/disable Log in button
- Tap "I forgot my password" → Navigate to password reset flow
- Tap "Log in" → Validate credentials, show loading, navigate to onboarding wizard
- Invalid credentials → Show error message below inputs

### Validation

#### Email
- Must contain "@" and "."
- Minimum 5 characters
- No spaces
- Error: "Please enter a valid email address"

#### Password
- Minimum 8 characters
- Error: "Password must be at least 8 characters"

---

## Animations

### Screen Transitions
- Type: Push/Pop with slide
- Duration: 0.35 seconds
- Curve: Ease-in-out
- Direction: Right-to-left (forward), Left-to-right (back)

### Button Press
- Scale: 0.97 (scale down on press)
- Duration: 0.15 seconds
- Curve: Ease-out

### Input Focus
- Border color transition: 0.2 seconds, ease-in-out
- Label animation: Move to top-left, 0.25 seconds, ease-out

### Loading State
- Show activity indicator in button
- Disable button interaction
- Fade out button text, fade in spinner
- Duration: 0.2 seconds

### Error Shake
- Shake input left-right (±8pt)
- Duration: 0.4 seconds
- Curve: Ease-in-out

---

## Accessibility

### VoiceOver Labels
- Logo: "Findie - Discover indie games"
- Get Started button: "Get started, button"
- Sign In button: "Sign in, button"
- Back button: "Back, button"
- Email input: "Email, text field"
- Password input: "Password, secure text field"
- Show password toggle: "Show password, button"
- Forgot password link: "I forgot my password, link"
- Log in button: "Log in, button"

### Dynamic Type
- Support all text size categories
- Minimum touch target: 44pt × 44pt
- Layout should reflow with larger text

### Color Contrast
- All text on dark background: Minimum 7:1 ratio (AAA)
- Button text on white: Minimum 4.5:1 ratio (AA)

### Keyboard Navigation
- Tab order: Top to bottom, left to right
- Focus indicators: Blue border (2pt)
- Return key: "Next" for inputs, "Go" for last input

---

## Implementation Notes

### SwiftUI Structure

```swift
// Screen 1: Welcome
WelcomeView()
  ├─ BackgroundView(game: randomGame)
  ├─ VStack
  │   ├─ Spacer()
  │   ├─ LogoView()
  │   ├─ Spacer()
  │   ├─ PrimaryButton("GET STARTED")
  │   ├─ SecondaryButton("SIGN IN")
  │   └─ TermsTextView()

// Screen 2: Sign-Up Options
SignUpOptionsView()
  ├─ BackgroundView(game: randomGame)
  ├─ BackButton()
  ├─ VStack
  │   ├─ LogoView(size: .small)
  │   ├─ TaglineView()
  │   ├─ Spacer()
  │   ├─ AppleSignInButton()
  │   ├─ GoogleSignInButton()
  │   ├─ EmailSignUpButton()
  │   ├─ Spacer()
  │   └─ TermsTextView()

// Screen 3: Email Login
EmailLoginView()
  ├─ BackgroundView(game: randomGame)
  ├─ BackButton()
  ├─ VStack
  │   ├─ LogoView(size: .small)
  │   ├─ TaglineView()
  │   ├─ Spacer()
  │   ├─ EmailTextField()
  │   ├─ PasswordTextField()
  │   ├─ ForgotPasswordLink()
  │   ├─ Spacer()
  │   ├─ PrimaryButton("Log in")
  │   └─ TermsTextView()
```

### Reusable Components

Create these shared components:
1. `BackgroundView` - Blurred game image with overlay
2. `LogoView` - Findie logo with sizing options
3. `TaglineView` - "Play beyond the mainstream."
4. `PrimaryButton` - White filled button
5. `SecondaryButton` - Outlined button
6. `AuthProviderButton` - Button with icon and text
7. `BackButton` - Circular back navigation button
8. `TermsTextView` - Terms and privacy links
9. `FormTextField` - Custom text field with animations
10. `LoadingButton` - Button with loading state

### Assets Needed

1. **Findie Logo**
   - Icon: SVG/PDF (vector)
   - Wordmark: SVG/PDF (vector)
   - Combined: @1x, @2x, @3x PNG

2. **Provider Icons**
   - Apple logo: SF Symbol `applelogo`
   - Google logo: Custom asset @1x, @2x, @3x PNG

3. **UI Icons**
   - Back chevron: SF Symbol `chevron.left`
   - Show password: SF Symbol `eye` / `eye.slash`

4. **Background Games**
   - 10-15 curated indie game screenshots
   - Resolution: 1242×2688 (iPhone 13 Pro Max)
   - Format: JPG, optimized for performance

---

## Testing Checklist

### Visual Testing
- [ ] Test on iPhone SE (smallest screen)
- [ ] Test on iPhone 15 Pro Max (largest screen)
- [ ] Test on iPhone with notch
- [ ] Test on iPhone with Dynamic Island
- [ ] Test with smallest Dynamic Type size
- [ ] Test with largest Dynamic Type size
- [ ] Test in light mode (should force dark)
- [ ] Test in dark mode

### Interaction Testing
- [ ] Back navigation works correctly
- [ ] Buttons respond to taps with animation
- [ ] Inputs focus and blur correctly
- [ ] Password show/hide works
- [ ] Form validation works
- [ ] Error messages display correctly
- [ ] Loading states show correctly
- [ ] Keyboard dismisses on tap outside
- [ ] Screen transitions smooth

### Accessibility Testing
- [ ] VoiceOver reads all elements
- [ ] All buttons have 44pt touch targets
- [ ] Color contrast passes WCAG AA
- [ ] Dynamic Type reflows layout
- [ ] Focus indicators visible
- [ ] All images have alt text

### Edge Cases
- [ ] Very long email address
- [ ] Special characters in password
- [ ] Network error during login
- [ ] Apple Sign In cancellation
- [ ] Google Sign In error
- [ ] Rapid button tapping
- [ ] Background app and return
- [ ] Low memory warning

---

## Next Steps

After implementing the onboarding flow:
1. Create onboarding wizard (genre selection, platform preference, etc.)
2. Implement persistent authentication (keychain)
3. Add biometric authentication (Face ID / Touch ID)
4. Implement password reset flow
5. Add email verification flow
6. Create account deletion flow

---

**Design Reference**: `/Users/brandon/findie/Findie.png`
**Last Updated**: October 2025
**Status**: Ready for Implementation
