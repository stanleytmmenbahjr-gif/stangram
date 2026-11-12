# Stangram v1 Implementation Plan

## 1. Product definition

Build a mobile-first social app for a friends-and-community audience using the existing Expo Router project.

### Required v1 capabilities

- Sign up and sign in with Google or Apple through Clerk.
- Require users to be at least 16 years old.
- Require a unique username and display name during onboarding. Profile photo and bio are optional.
- Let users create a post with one photo or short video, selected from their library or captured with the device camera.
- Let users choose public or private profile visibility.
- Require approval for follow requests to private accounts.
- Show a chronological Home feed containing posts from approved follows.
- Provide Explore search for users and basic suggestions or recent public content.
- Support likes and comments, with deletion limited to the content owner.
- Support direct messages containing text and images. Require recipient approval to start a conversation, and disallow messaging between teen and adult accounts.
- Support blocking and reporting.
- Let users edit their profile and permanently delete their account.
- Send push notifications for follow requests, likes/comments, and messages.
- Provide a protected web admin page to review reports, remove content, and suspend accounts.

### Main navigation

- **Home:** Following feed.
- **Messages:** Requests and approved conversations.
- **Explore:** User search and public content.
- **Profile:** Current user’s profile and editing.

All content requires sign-in to view. V1 has no ads or subscription billing.

## 2. Existing project and technology constraints

- The repository is an Expo Router starter with routes under `src/app/`.
- Current package versions include Expo SDK 58, React Native 0.88, and React 19.
- Use Clerk for Google and Apple authentication.
- Use Convex for backend functions, database, live updates, and media storage.
- Use NativeWind for styling.
- Preserve the existing Expo Router structure and configure native behavior through Expo configuration or config plugins; do not hand-edit generated `ios/` or `android/` directories.
- Before implementing any Expo, EAS, or React Native API integration, read the matching Expo SDK 58 documentation and the linked current documentation for the relevant API. Confirm NativeWind and auth/backend package compatibility with this SDK before choosing package versions.
- Keep service credentials in the appropriate local and deployment environment configuration. Never commit secrets.

## 3. Implementation phases

### Phase 1: Confirm technical compatibility and configure environments

1. Confirm the exact NativeWind version and setup that supports this project’s Expo SDK, React Native, and web targets.
2. Confirm Clerk’s current Expo setup for Google and Apple sign-in across iOS, Android, and web.
3. Confirm Convex’s current Clerk integration and supported authentication flow.
4. Identify any development-build requirements for the selected auth, media, or notification packages. Keep Expo Go limitations visible in developer setup instructions.
5. Define local, preview/staging, and production environment variables for Clerk, Convex, OAuth, and push notifications.
6. Document the required Google and Apple provider configuration and redirect URLs for each target platform.
7. Record the expected low-spend service limits and establish usage alerts where available.

**Acceptance criteria**

- The app starts for iOS, Android, and web using the documented setup.
- Development, preview, and production configurations are distinguishable.
- No service secret is committed to the repository.

### Phase 2: Establish app shell and design system

1. Add a consistent NativeWind styling foundation: colors, spacing, typography, shared controls, loading states, empty states, and error presentation.
2. Create the four-tab Expo Router navigation shell: Home, Messages, Explore, and Profile.
3. Define authenticated and unauthenticated route handling.
4. Add shared UI elements for avatars, post cards, buttons, text inputs, confirmation dialogs, and notification badges.
5. Ensure layouts adapt to mobile and web without changing the core navigation model.

**Acceptance criteria**

- All four tabs are reachable and have clear loading, empty, and error states.
- Navigation and content remain usable on narrow mobile screens and web.
- Authentication state determines access to protected routes.

### Phase 3: Authentication, onboarding, and user profiles

1. Integrate Clerk sign-in and sign-up with Google and Apple.
2. On first sign-in, collect date of birth, username, and display name.
3. Enforce the 16+ minimum and prevent entry when the declared birth date is below the configured minimum.
4. Enforce unique usernames and the selected username change cooldown.
5. Allow users to add or edit a profile photo and bio after or during onboarding; do not make these fields mandatory.
6. Create the corresponding Convex user profile, linked to the Clerk user identifier.
7. Support public/private profile selection and profile editing.
8. Implement self-service account deletion, including Clerk identity deletion and cleanup or anonymization of related Convex data and stored media according to the approved deletion policy.

**Acceptance criteria**

- New users cannot reach the main app before completing required onboarding.
- Returning users reach the app without repeating onboarding.
- Duplicate usernames and users below the minimum age are rejected.
- Profile access and editing are limited to the authenticated user.
- Account deletion does not leave orphaned private media or inaccessible records.

### Phase 4: Convex data model and authorization

Create Convex schema and server-side functions for these entities:

- **Users:** Clerk identity reference, username, display name, optional bio and avatar, birth date or age category, profile visibility, suspension/deletion state, timestamps.
- **Follows:** follower, followed user, status (`pending` or `accepted`), timestamps.
- **Posts:** author, media reference/type, caption, creation time, deletion state.
- **Likes:** post, user, creation time; enforce one like per user per post.
- **Comments:** post, author, text, creation time, deletion state.
- **Conversations:** participants, request/approval state, timestamps.
- **Messages:** conversation, sender, text and/or image reference, creation time, read state if included.
- **Blocks:** blocking user, blocked user, creation time.
- **Reports:** reporter, reported account or content, category/details, status, review metadata.
- **Admin authorization:** protected role or allowlist for report review and account/content actions.
- **Device tokens and notification preferences:** user, platform, token, enabled state and timestamps.

Implement authorization in Convex functions, not only in the client. Every read and mutation must enforce identity, visibility, relationship, suspension, block, and age rules as applicable.

**Acceptance criteria**

- A user cannot edit, delete, or impersonate another user’s content or profile.
- Private content is only readable by the owner and approved followers, subject to blocking and suspension rules.
- Pending follow requests do not expose private posts.
- Blocked users cannot continue interaction or messaging.
- Admin functions reject callers without admin authorization.

### Phase 5: Media upload and post creation

1. Implement library selection and camera capture using packages compatible with the project’s Expo SDK; verify required APIs in version-matched documentation.
2. Request camera or photo-library permissions only when the user selects the corresponding action.
3. Upload media to Convex storage and create a post record only after the upload succeeds.
4. Support one image or one video per post and an optional caption.
5. Display upload progress and allow cancel/retry when feasible.
6. Define and enforce video duration, file-size, and supported-format limits before implementation.
7. Render images efficiently and ensure video playback behavior works on iOS, Android, and web.
8. Delete stored media when its post is deleted, subject to cleanup behavior and retry handling.

**Acceptance criteria**

- Users can create image and video posts from a library selection or capture flow.
- Failed uploads do not create broken post records.
- Users see progress and a recoverable error if upload fails.
- Deleted posts are no longer visible and their media is eventually removed.

### Phase 6: Follow graph, visibility, feeds, and Explore

1. Implement follow/unfollow for public accounts.
2. Implement pending, accept, and decline follow requests for private accounts.
3. Build the Home feed as a chronological query over posts from accepted follows, excluding content hidden by blocks or deletions.
4. Build Explore user search and basic suggestions/recent public content. Do not add personalized ranking in v1.
5. Enforce profile visibility on all profile, post, and media queries.
6. Add pagination or cursor-based loading for feeds and search results.
7. Provide empty states for users with no follows and searches with no results.

**Acceptance criteria**

- Home shows only eligible posts in chronological order.
- Private account content remains hidden until a follow request is accepted.
- Explore does not reveal private posts or blocked users’ content.
- Follow and request states update consistently across profile and feed screens.

### Phase 7: Likes, comments, and safety controls

1. Add and remove likes, maintaining a single like per user per post.
2. Add comments and allow only the comment author to delete their comment.
3. Allow post owners to delete their posts.
4. Allow users to block and unblock accounts.
5. Allow users to report accounts, posts, comments, or messages.
6. Provide clear confirmations and feedback for deletion, blocking, and report submission.
7. Apply visibility, block, and suspension rules to all interactions.

**Acceptance criteria**

- Like state and counts remain consistent after repeated taps and live updates.
- Users cannot delete another person’s comment or post.
- Blocked relationships prevent further access and interaction according to the visibility policy.
- Reports are stored and visible only to authorized admins and the reporting user where applicable.

### Phase 8: Messaging and age-aware access

1. Implement message requests and recipient approval before a conversation becomes active.
2. Support text and image messages.
3. Enforce the 16+ minimum and prohibit conversations between teen and adult age categories, including initiation and acceptance paths.
4. Ensure blocked users cannot message or continue existing conversations.
5. Use Convex live updates for active conversations.
6. Provide request, empty, loading, offline, and send-failure states.
7. Decide whether read receipts are part of v1 before implementation; default to omitting them unless explicitly required.

**Acceptance criteria**

- A new message request does not create an approved conversation until accepted.
- Messages are visible only to conversation participants, subject to safety and moderation access.
- Cross-age conversations cannot be created, accepted, or continued.
- Failed message sends are clearly shown and can be retried without duplicate messages.

### Phase 9: Notifications

1. Register device push tokens for supported platforms and store them in Convex.
2. Use the appropriate Expo-supported push delivery path after verifying SDK 58 documentation and current provider requirements.
3. Send notifications for follow requests/approvals, likes/comments, and new messages.
4. Respect user notification preferences and remove invalid tokens.
5. Avoid including sensitive message contents in lock-screen notification text by default.
6. Make notification taps navigate to the relevant authenticated screen.

**Acceptance criteria**

- Notifications are delivered only to the intended account and respect preferences.
- Invalid or revoked device tokens do not prevent other notifications.
- Tapping a notification opens the correct content or conversation after authentication.

### Phase 10: Admin report review

1. Build a protected web-only admin route or page.
2. Let authorized admins list, inspect, and resolve reports.
3. Support actions to remove reported content and suspend or restore accounts.
4. Record reviewer, action, and timestamp for moderation actions.
5. Ensure moderation actions are enforced by Convex authorization and reflected in app queries.
6. Document how admins receive access and how report-review responsibility is assigned.

**Acceptance criteria**

- Non-admin users cannot load report details or perform moderation actions.
- Admins can review and resolve reports and see the resulting content/account state.
- Suspended accounts cannot continue posting, commenting, following, or messaging.

### Phase 11: Reliability, privacy, and release readiness

1. Add privacy-conscious error reporting and operational logging for auth, uploads, Convex functions, push delivery, and moderation actions.
2. Avoid logging credentials, full message content, or unnecessary birth-date data.
3. Add pagination and media loading behavior suitable for mobile networks.
4. Verify account deletion, private-account visibility, block behavior, and teen messaging rules end to end.
5. Configure app identifiers, icons, deep links, permissions, OAuth domains, and store metadata.
6. Prepare public iOS and Android store releases and deploy a usable web companion.
7. Document build, release, rollback, and service-limit monitoring procedures.

**Acceptance criteria**

- Critical privacy and authorization rules are checked on the backend.
- The app works on current supported iOS, Android, and web environments.
- Store builds and web deployment use production service configuration.
- No critical-flow regression remains open at release.

## 4. Test plan

Automate tests for:

- Sign-in state, onboarding completion, minimum-age enforcement, and username uniqueness.
- Public versus private profile visibility.
- Follow requests, approval, decline, follow/unfollow, and feed membership.
- Post creation, upload failure, deletion, and media access.
- Like uniqueness, comment creation, and owner-only deletion.
- Message request approval, participant-only reads, and cross-age restrictions.
- Block and report behavior.
- Admin-only report review, content removal, and account suspension.
- Account deletion and related data/media cleanup.

Perform manual device and browser QA for:

- iOS, Android, and web sign-in with Google and Apple.
- Camera and photo-library permission grant/deny flows.
- Image and video selection, capture, upload, playback, and retry.
- Push permission grant/deny, notification delivery, and notification deep links.
- Responsive layouts, empty states, slow network behavior, and app background/foreground transitions.
- Public store builds and production web configuration.

## 5. Decisions and defaults

- Public signup with Google or Apple.
- Minimum age: 16.
- Sign-in required to view all app content.
- Username and display name required; photo and bio optional.
- User-selectable public/private profiles.
- Chronological following feed; basic Explore search and recent public content.
- One photo or short video per post.
- Messages support text and images, require recipient approval, and are limited to the same age category.
- Convex is the backend, database, live-update, and media-storage service.
- Push notifications cover follow requests/approvals, likes/comments, and messages.
- Report review uses a minimal protected web admin page.
- Admins may remove content and suspend accounts.
- Low-spend community launch, public iOS/Android stores, usable web companion, no fixed release date.
- No ads or subscription billing in v1.

## 6. Open risks to resolve before implementation

- Set maximum video duration, upload size, supported formats, and any processing/transcoding approach; validate against Convex storage and delivery limits.
- Confirm NativeWind, Clerk, Convex, and push-notification package versions and compatibility with the installed Expo SDK 58 / React Native 0.88 setup.
- Verify age, privacy, moderation, and store requirements for the intended launch regions, including the 16+ signup policy.
- Define who will review reports and how quickly, and establish an admin access process.
- Estimate public-launch media, database, and push usage against the low-spend budget and configure usage alerts.
- Define the exact data-retention behavior for deleted users’ messages, reports, and moderation records.
