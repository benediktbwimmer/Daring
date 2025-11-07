# Connection Key Mobile App Architecture

## Product Vision
Create a respectful dating-style messaging platform where thoughtful effort is rewarded. Before direct messages are unlocked, senders complete a short "Connection Key" that proves they have read the receiver's profile and agree to their consent preferences. The app must work on Android and iOS from a single cross-platform codebase.

## Cross-Platform Technology Choices
- **Mobile client**: [React Native](https://reactnative.dev/) with Expo Router for fast iteration, OTA updates, and access to native capabilities when needed.
- **State management**: Zustand for local state + React Query for server cache synchronization.
- **Design system**: Restyle or Tamagui for theme tokens, accessibility, and dark-mode support.
- **Authentication**: Passwordless sign-in (email + magic link) using Supabase Auth or Cognito; supports future WebAuthn.
- **Backend**: Kotlin/Java Spring Boot or Node.js NestJS (cloud-agnostic) offering a GraphQL API + REST fallbacks.
- **Data**: PostgreSQL for relational data, Redis for ephemeral quiz stores, OpenSearch/Elastic for retrieval.
- **ML services**: Python microservices (FastAPI) for embeddings, quiz generation, and fairness monitoring.
- **Messaging**: WebSocket gateway (e.g., AWS AppSync subscriptions or Socket.io) for real-time intro updates.

## Core User Journeys
1. **Onboarding & Cold Start**
   - Collect consent, minimal demographics, dealbreakers.
   - 10–15 "vibe picks" to seed preference embeddings.
   - Optional ID verification for trust badge.

2. **Browsing Matches**
   - Two-tower retrieval surfaces a diverse candidate carousel.
   - Users can filter (distance, pet preference, etc.) and see "Why you're seeing this" hints.

3. **Connection Key Quiz**
   - Triggered before the sender composes the first message.
   - 3–5 questions: profile comprehension, values overlap, situational trade-off, consent acknowledgement, micro free-response.
   - Time boxed to 60 seconds; accessibility toggle offers audio playback + speech-to-text.

4. **Messaging Unlock**
   - Passing threshold unlocks the message composer and shows both parties an "Alignment Card" summarizing quiz results.
   - Near misses allow a short 200-character intro; low scores queue in requests.

5. **Receiver Controls**
   - Adjustable difficulty, do-not-disturb schedule, auto-decline rules, analytics on inbound quality.

## Mobile App Structure
```
app/
 ├─ (auth)/          // login, verification
 ├─ (onboarding)/    // multi-step onboarding
 ├─ (match)/         // discovery feed, candidate details
 ├─ (quiz)/          // Connection Key wizard
 ├─ (inbox)/         // conversations & requests
 └─ settings/        // consent, privacy, account
```
Key shared modules live in `src/`:
- `src/api/` GraphQL hooks (React Query).
- `src/components/` design system primitives and quiz-specific widgets.
- `src/ml/` client-side feature logging + experiment toggles.
- `src/state/` global stores for auth session, device preferences.
- `src/utils/` timers, formatting, accessibility helpers.

## Connection Key Quiz Flow
1. **Fetch quiz package**: Client calls `POST /connection-keys` with target profile ID.
2. **Render questions**: Components handle multiple-choice, Likert sliders, scenario cards, consent toggles, and one mini free-response.
3. **Timer & instrumentation**: 60s countdown, track blur/focus, copy/paste, answer changes.
4. **Submit**: `POST /connection-keys/{id}/responses`. Response includes score, threshold, and message unlock state.
5. **Feedback**: If failed, UI surfaces specific explanations referencing the profile section, and optionally a targeted prompt for retry.

### Anti-Gaming Measures
- Questions reference non-obvious profile elements with rotating decoys.
- Ephemeral quiz payload signed server-side; expires after 5 minutes.
- Free-response evaluated by LLM rubric (respectful tone, cites detail) + toxicity classifier.
- Rate-limit attempts per sender/receiver pair; escalate to manual review for suspicious patterns.

## Matching & Ranking Architecture
1. **Embedding Service**
   - Generates user interest vectors from text + quiz signals using Sentence Transformers.
   - Stores vectors in a vector DB (e.g., Pinecone, pgvector) keyed by user ID.

2. **Candidate Retrieval**
   - Two-tower model retrieves top-N candidates; ensures demographic fairness via constrained sampling.

3. **Re-ranking**
   - Cross-encoder re-ranker considers compatibility, consent alignment, novelty, and fairness constraints.
   - Caps influence of quiz score to avoid echo chambers; ensures exposure for new/less-engaged users.

4. **Feedback Loop**
   - Quiz answers, message outcomes, reports feed into offline training pipeline.
   - Scheduled calibration checks for disparate impact across demographics.

## Safety & Consent Features
- Pre-message moderation (quiz + intros) using moderation ML and human-in-the-loop review.
- Consent pact required before messaging; receiver can enforce boundaries.
- Transparent UI: "Why this question" tooltips, skip options for sensitive items.
- Explainability logs stored for audit + appeals.

## Monetization & Operations
- One-time €/$5 onboarding fee (Stripe/RevenueCat) covering bot deterrence + moderation.
- Optional add-ons: ID verification badge, event tickets, priority moderation review.
- No pay-to-message; core communication remains free after Connection Key.

## Analytics & Experimentation
Track:
- Quiz completion time, pass rate, drop-off reasons.
- Message acceptance rate, report/block rate post-intro.
- Trust metrics: share of intros with consent pact, receiver satisfaction surveys.
- Fairness metrics: exposure parity, reply parity, calibration drift.
- Abuse metrics: harassment incidents per 1k intros, time-to-action.

Use Feature Flag system (e.g., LaunchDarkly) to test quiz variations, threshold tuning, and onboarding experiments.

## Implementation Roadmap
1. **M1: Foundation (6–8 weeks)**
   - Set up repo, CI/CD, Expo app shell, auth integration, basic matching feed.
   - Implement onboarding & vibe picks, establish backend GraphQL schema.

2. **M2: Connection Key MVP (8 weeks)**
   - Build quiz UI components, backend question generator, scoring service.
   - Add consent pact, unlock logic, analytics pipelines.
   - Release closed beta, collect qualitative feedback.

3. **M3: Hardening & Fairness (6 weeks)**
   - Anti-gaming heuristics, fairness monitoring dashboards, accessibility upgrades.
   - Introduce receiver controls, intro request fallback, moderation tooling.

4. **M4: Monetization & Scale (6+ weeks)**
   - Launch one-time fee + optional add-ons.
   - Optimize embeddings, implement A/B testing, polish alignment cards.

## Testing Strategy
- Automated: Jest + React Native Testing Library for UI; Detox for end-to-end flows.
- Manual: Accessibility testing with screen readers and voice input.
- Security: Regular pen tests, quiz package signing verification.
- ML: Offline evaluation suite for retrieval/re-ranking, red-teaming question generator.

## Deployment & DevOps
- CI via GitHub Actions: lint, unit tests, build artifacts.
- Expo EAS for OTA updates + store builds.
- Backend deployment on AWS (EKS or ECS) with infrastructure as code (Terraform).
- Observability with OpenTelemetry, Grafana, PagerDuty for incident response.

## Open Questions
- Balance between one-time fee and trial experience (free passes for referrals?).
- How to include voice-based Connection Keys for visually impaired users.
- Path to web client parity (React Native Web vs. dedicated Next.js frontend).

