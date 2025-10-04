# GA4 Events and Funnels Implementation Plan

This document provides a comprehensive mapping of Google Analytics 4 events, parameters, and funnel definitions for the Skiller learning platform, with specific code locations for implementation.

## Conventions

- **Event Names**: snake_case format
- **Parameters**: Flat, GA-friendly primitives
- **user_id**: Session user ID
- **page**: Pathname (optional; GA has page_location/page_referrer)
- **method**: 'password' | 'google' | 'otp'
- **status**: 'success' | 'error'
- **source**: 'dashboard' | 'roadmap_detail' | etc.
- **values**: Numeric where meaningful (score, durations, token counts)

---

## 1. Authentication & Onboarding Events

### signup_submitted
- **Parameters**: `{ method, page }`
- **Hook Location**: `app/sign_up/SignupForm.tsx` in `handleSubmit` right before calling `registerUser`

### otp_sent
- **Parameters**: `{ email_domain, page }`
- **Hook Location**: `app/sign_up/SignupForm.tsx` after `sendOtp(email)` success

### otp_verified (CONVERSION)
- **Parameters**: `{ status, page }`
- **Hook Location**: `app/sign_up/SignupForm.tsx` in `handleVerify`, on `result.success`

### login_submitted
- **Parameters**: `{ method: 'password', page }`
- **Hook Location**: `app/login/LoginForm.tsx` in `handleSubmit`, before `signIn`

### login_success (CONVERSION)
- **Parameters**: `{ method, page }`
- **Hook Location**: `app/login/LoginForm.tsx` after `await getSession()` and before `router.replace("/dashboard")`

### onboarding_completed (CONVERSION)
- **Parameters**: `{ goal, weekly_commitment, wants_ai_reco, wants_notifications, wants_community }`
- **Hook Location**: `app/onboarding/onboardingClient.tsx` after `await createUserProfile(formattedData)` and before redirect

**Optional User Properties**: Set user properties when known (e.g., goal, weekly_commitment) after onboarding success.

---

## 2. Roadmap Creation Events

### create_roadmap_cta_click
- **Parameters**: `{ page: 'dashboard' }`
- **Hook Location**: `components/dashboard/welcome-section.tsx` in `onClick={handleOpenModal}`

### create_roadmap_modal_open
- **Parameters**: `{ source: 'dashboard' }`
- **Hook Location**: `components/dashboard/dashboard.tsx` when `setShowModal(true)` is called

### roadmap_generation_started
- **Parameters**: `{ topic, level, daily_hours, total_weeks, srs_enabled }`
- **Hook Location**: `components/dashboard/create-roadmap.tsx` at start of `handleSubmit`

### roadmap_created (CONVERSION)
- **Parameters**: `{ roadmap_id, topic, level, daily_hours, total_weeks, srs_enabled }`
- **Hook Location**: `components/dashboard/create-roadmap.tsx` right after `const roadmap = await createPlan(...);`

### milestones_generated
- **Parameters**: `{ roadmap_id, milestones_count, steps_count }`
- **Hook Location**: `actions/dashboard/helper.ts` at end of `showmilestone` (aggregate counts)

---

## 3. Learning Progress Events

### roadmap_view
- **Parameters**: `{ roadmap_id }`
- **Hook Location**: `components/roadmap/roadmap-view.tsx` after the data loads (`dispatch(setRoadmap(...))`)

### step_started
- **Parameters**: `{ roadmap_id, milestone_id, step_id, step_type }`
- **Hook Locations**: 
  - `components/roadmap/roadmap-step-detail.tsx` when clicking Start button that navigates to step
  - `app/roadmaps/[roadmapId]/milestone/[milestoneId]/step/[stepId]/page.tsx` after step is found

### step_completed (CONVERSION)
- **Parameters**: `{ roadmap_id, milestone_id, step_id, completion_mode: 'quiz' | 'manual' }`
- **Hook Locations**:
  - Manual: `components/roadmap/roadmap-step-detail.tsx` after `await markStepComplete(...)`
  - Quiz path: `actions/quiz/submit-quiz.ts` after step status update

### milestone_completed
- **Parameters**: `{ roadmap_id, milestone_id }`
- **Hook Location**: `actions/quiz/submit-quiz.ts` when last step in milestone triggers milestone update

### roadmap_completed (CONVERSION)
- **Parameters**: `{ roadmap_id }`
- **Hook Location**: `components/milestone/steps/QuizStep.tsx` after `updateRoadmapStatus({ status: "COMPLETED" })`

---

## 4. Quiz Events

### quiz_started
- **Parameters**: `{ roadmap_id, milestone_id, step_id, item_count }`
- **Hook Location**: `components/milestone/steps/QuizStep.tsx` when quiz UI is first rendered/activated for the step

### quiz_submitted (CONVERSION)
- **Parameters**: `{ roadmap_id, milestone_id, step_id, score, correct_count, total }`
- **Hook Location**: `components/milestone/steps/QuizStep.tsx` in `handleSubmit` after computing `correctCount`

### quiz_regenerated
- **Parameters**: `{ roadmap_id, milestone_id, step_id, reason?: 'quality' }`
- **Hook Location**: `components/milestone/steps/QuizStep.tsx` in the regenerate handler

---

## 5. Reviews (SRS) Events

### review_modal_open
- **Parameters**: `{ topic_id, source_page }`
- **Hook Location**: `app/reviews/Reviews.tsx` when `TopicReviewModal` opens

### review_started
- **Parameters**: `{ topic_id, content_type: 'FLASHCARD' | 'QUIZ' }`
- **Hook Location**: `components/reviews/TopicReviewModal.tsx` when active tab becomes flashcard or quiz and content rendered

### review_completed (CONVERSION)
- **Parameters**: `{ topic_id, content_type, score?: number, grade?: 'good'|'easy'|'hard' }`
- **Hook Locations**: 
  - `components/reviews/TopicReviewModal.tsx` in `handleFlashCardReviewComplete` and `handleQuizReviewComplete` after successful saves
  - `actions/srs/review-actions.ts` after update

### review_exit
- **Parameters**: `{ topic_id, content_type }`
- **Hook Location**: `components/reviews/TopicReviewModal.tsx` in `handleExit`

---

## 6. Summary Generation Events

### summary_generate_clicked
- **Parameters**: `{ roadmap_id, milestone_id, step_id }`
- **Hook Location**: `components/milestone/steps/QuizStep.tsx` at start of `handleGenerateSummary`

### summary_generated (CONVERSION)
- **Parameters**: `{ roadmap_id, milestone_id, step_id, content_length }`
- **Hook Location**: `components/milestone/steps/QuizStep.tsx` after `saveQuizSummary(...)` and redux `summaryUnlocked: true`

### summary_generation_failed
- **Parameters**: `{ roadmap_id, milestone_id, step_id, error_message }`
- **Hook Location**: `components/milestone/steps/QuizStep.tsx` failure branch in `handleGenerateSummary`

---

## 7. Chat & Token Usage Events

### chat_opened
- **Parameters**: `{ context_type, context_id, roadmap_id }`
- **Hook Location**: `app/(component)/chat-component/chatWindow.tsx` when component mounts and welcome is hidden

### chat_message_sent
- **Parameters**: `{ context_type, context_id, roadmap_id, conversation_id }`
- **Hook Location**: `chatWindow.tsx` right before calling `sendMessage`

### chat_response_received
- **Parameters**: `{ conversation_id, tokens_used, billable_tokens }`
- **Hook Location**: `app/(component)/chat-component/action.ts` after `saveMessagesToConversation`

### chat_quota_exceeded
- **Parameters**: `{ conversation_id, feature: 'ai_chat', limit, used, next_reset }`
- **Hook Location**: `chatWindow.tsx` where `result.quotaExceeded` is handled

### token_overhead_warn
- **Parameters**: `{ conversation_id, total_actual_tokens, billable_tokens, overhead_tokens, overhead_ratio }`
- **Hook Location**: `app/(component)/chat-component/db.ts` where `logOverheadEvent` is called and ratio > 0.35

---

## 8. UI & Engagement Events

### create_roadmap_modal_close
- **Parameters**: `{ source: 'dashboard', outcome: 'dismiss'|'submit' }`
- **Hook Location**: `components/dashboard/dashboard.tsx` where `setShowModal(false)` is called

### next_step_clicked
- **Parameters**: `{ roadmap_id, from_step_id, to_step_id }`
- **Hook Location**: `components/milestone/step-content.tsx` in `handleGoToNextStep` before `router.push`

### step_tab_changed
- **Parameters**: `{ step_id, tab: 'content'|'quiz'|'summary' }`
- **Hook Location**: Wherever tab switching is handled in `milestone-step-layout.tsx` and related slices

### dashboard_view
- **Parameters**: `{ user_id }`
- **Hook Location**: `app/dashboard/page.tsx` or `components/dashboard/dashboard.tsx` on initial render

---

## 9. Web Vitals (Performance) Events

### web_vital
- **Parameters**: `{ metric_name: 'LCP'|'CLS'|'INP'|'FCP'|'TTFB', value, id, page }`
- **Hook Location**: Create `lib/web-vitals.ts` utility and wire via Next's web-vitals listener in client; import once in `app/layout.tsx` provider

---

## 10. GA4 Funnels (Explorations → Funnel Exploration)

### Acquisition & Onboarding Funnel
1. `signup_submitted`
2. `otp_sent`
3. `otp_verified` (conversion)
4. `login_success` (conversion)
5. `onboarding_completed` (conversion)

### Roadmap Creation Funnel
1. `dashboard_view`
2. `create_roadmap_cta_click`
3. `create_roadmap_modal_open`
4. `roadmap_generation_started`
5. `roadmap_created` (conversion)
6. `milestones_generated`

### Learning Progress Funnel
1. `roadmap_view`
2. `step_started`
3. `step_completed` (conversion)
4. `quiz_started`
5. `quiz_submitted` (conversion)
6. `milestone_completed`
7. `roadmap_completed` (conversion)

### Reviews (SRS) Funnel
1. `review_modal_open`
2. `review_started`
3. `review_completed` (conversion)
4. `review_exit` (as a drop-off signal)

### Chat Engagement Funnel
1. `chat_opened`
2. `chat_message_sent`
3. `chat_response_received`
4. `chat_quota_exceeded` (anti-goal step to analyze drop-offs)

---

## 11. Conversion Events to Mark in GA4

Mark these events as conversions in GA Admin → Conversions:

- `otp_verified`
- `login_success`
- `onboarding_completed`
- `roadmap_created`
- `step_completed`
- `quiz_submitted`
- `milestone_completed`
- `roadmap_completed`
- `review_completed`
- `summary_generated`

---

## 12. Parameters Taxonomy

### Global Parameters
- `user_id`, `page`, `session_id?`

### Authentication Parameters
- `method`, `status`, `error_code?`

### Roadmap Parameters
- `roadmap_id`, `topic`, `level`, `daily_hours`, `total_weeks`, `srs_enabled`

### Step/Milestone Parameters
- `milestone_id`, `step_id`, `step_type`, `completion_mode`

### Quiz Parameters
- `score`, `correct_count`, `total`

### Review Parameters
- `topic_id`, `content_type`, `score?`, `grade?`

### Summary Parameters
- `content_length`

### Chat/Token Parameters
- `conversation_id`, `tokens_used`, `billable_tokens`, `total_actual_tokens`, `overhead_tokens`, `overhead_ratio`, `limit`, `used`, `next_reset`

### UI Parameters
- `source`, `outcome`, `from_step_id`, `to_step_id`, `tab`

---

## 13. Implementation Notes

### Key Implementation Guidelines
- Mark listed "conversion" events in GA Admin → Conversions
- Engaged sessions and engagement rate are computed in GA; these events anchor user intent
- Keep PII out of event params (avoid raw emails)
- For reliability, duplicate key conversions on both client (fast UX feedback) and server (authoritative), de-duplicated by event id if you add one

### File-by-File Hook Locations

#### `app/sign_up/SignupForm.tsx`
- `handleSubmit` → `signup_submitted`
- After `sendOtp` success → `otp_sent`
- `handleVerify` success → `otp_verified`

#### `app/login/LoginForm.tsx`
- `handleSubmit` start → `login_submitted`
- After `getSession` & before redirect → `login_success`

#### `app/onboarding/onboardingClient.tsx`
- `handleSubmit` success → `onboarding_completed` (+ set user properties)

#### `components/dashboard/welcome-section.tsx`
- Create New Roadmap button → `create_roadmap_cta_click`

#### `components/dashboard/dashboard.tsx`
- When modal opens/closes → `create_roadmap_modal_open` / `create_roadmap_modal_close`
- On initial render → `dashboard_view`

#### `components/dashboard/create-roadmap.tsx`
- `handleSubmit` start → `roadmap_generation_started`
- After `createPlan` → `roadmap_created`

#### `actions/dashboard/helper.ts`
- End of `showmilestone` → `milestones_generated` (aggregate counts)

#### `components/roadmap/roadmap-view.tsx`
- After `setRoadmap` → `roadmap_view`

#### `app/roadmaps/[...]/step/[...]/page.tsx`
- After resolving step → `step_started`

#### `components/roadmap/roadmap-step-detail.tsx`
- After `markStepComplete` → `step_completed` (completion_mode: 'manual')

#### `components/milestone/step-content.tsx`
- Before push to next step → `next_step_clicked`

#### `components/milestone/steps/QuizStep.tsx`
- On show/activate → `quiz_started`
- `handleSubmit` → `quiz_submitted`, `step_completed` (completion_mode: 'quiz')
- If isLast... → `milestone_completed` / `roadmap_completed`
- Regenerate handler → `quiz_regenerated`
- `handleGenerateSummary` start → `summary_generate_clicked`
- After `saveQuizSummary` → `summary_generated`

#### `actions/quiz/submit-quiz.ts`
- After prisma updates → `step_completed` (server-confirmed) and `milestone_completed` if applicable

#### `app/reviews/Reviews.tsx`
- When `TopicReviewModal` opens → `review_modal_open`

#### `components/reviews/TopicReviewModal.tsx`
- When tab activates → `review_started`
- On complete → `review_completed`
- On exit → `review_exit`

#### `actions/srs/review-actions.ts`
- After prisma review update → `review_completed` (server-confirmed)

#### `app/(component)/chat-component/chatWindow.tsx`
- On mount/welcome hidden → `chat_opened`
- Before `sendMessage` → `chat_message_sent`
- On quota exceeded branch → `chat_quota_exceeded`

#### `app/(component)/chat-component/action.ts`
- After saving messages/result → `chat_response_received`

#### `app/(component)/chat-component/db.ts`
- Inside overhead logging path → `token_overhead_warn`

#### Performance/Web Vitals
- `lib/web-vitals.ts` (new util) → `web_vital` from web-vitals listener; import once from a client root (e.g., `app/layout.tsx` or a provider)

---

## 14. Next Steps

1. **Phase 1**: Implement core conversion events (authentication, roadmap creation, step completion)
2. **Phase 2**: Add engagement tracking (reviews, summaries, chat)
3. **Phase 3**: Implement advanced analytics (token usage, performance metrics)
4. **Phase 4**: Set up GA4 funnels and conversion goals
5. **Phase 5**: Create custom reports and dashboards

This comprehensive tracking plan will provide valuable insights into user behavior, conversion rates, and areas for improvement in the Skiller learning platform.
