# Schedica — Database Schema & Query Files

Same pattern as shoptimity-remix:
- **One schema file** — `src/lib/db/schema.ts` — all Drizzle table definitions
- **One db client** — `src/lib/db/index.ts` — Drizzle client (equivalent to `db.server.ts`)
- **Separate feature db files** — `src/lib/db/users.ts`, `src/lib/db/bookings.ts`, etc. — typed query functions per feature

---

## File Structure

```
src/lib/db/
  schema.ts            ← ALL table definitions in one file (22 tables)
  index.ts             ← Drizzle client setup
  users.ts             ← DbUsers.get / create / update / delete
  event-types.ts       ← DbEventTypes.get / create / update / delete
  availability.ts      ← DbAvailability.get / upsert / deleteOverride
  bookings.ts          ← DbBookings.create / get / cancel / list
  calendars.ts         ← DbCalendars.connect / getTokens / markDisconnected
  video.ts             ← DbVideo.connect / getTokens / disconnect
  notifications.ts     ← DbNotifications.getPrefs / upsertPrefs / logJob
```

---

## `src/lib/db/schema.ts`

All Drizzle table definitions in one file.

```typescript
import {
  pgTable,
  pgEnum,
  text,
  boolean,
  integer,
  timestamp,
  jsonb,
  index,
} from 'drizzle-orm/pg-core'
import { relations } from 'drizzle-orm'

// ─────────────────────────────────────────────────────────────────────────────
// ENUMS
// ─────────────────────────────────────────────────────────────────────────────

export const themeEnum = pgEnum('theme', ['light', 'dark', 'system'])

export const dateFormatEnum = pgEnum('date_format', [
  'MM/DD/YYYY',
  'DD/MM/YYYY',
  'YYYY-MM-DD',
])

export const timeFormatEnum = pgEnum('time_format', ['12h', '24h'])

export const locationTypeEnum = pgEnum('location_type', [
  'zoom',
  'google_meet',
  'teams',              // Phase 2
  'phone_host_calls',   // host calls the invitee
  'phone_invitee_calls',// invitee calls the host
  'in_person',
  'custom',
  'invitees_choice',
])

export const questionTypeEnum = pgEnum('question_type', [
  // MVP — 5 types
  'short_text',
  'long_text',
  'phone',
  'single_select',
  'dropdown',
  // Phase 2 — 4 types
  'multiple_select',
  'number',
  'date',
  'url',
])

export const dayOfWeekEnum = pgEnum('day_of_week', [
  'monday', 'tuesday', 'wednesday', 'thursday',
  'friday', 'saturday', 'sunday',
])

export const bookingWindowTypeEnum = pgEnum('booking_window_type', [
  'rolling', // "next 60 days"
  'fixed',   // "Jun 1 – Aug 31"
])

export const bookingStatusEnum = pgEnum('booking_status', [
  'confirmed',
  'cancelled',
  'rescheduled', // old booking after reschedule; new booking linked via rescheduledFromId
  'completed',   // meeting time passed, not cancelled
  'no_show',     // Phase 2 auto-detect; manual marking at MVP
])

export const calendarProviderEnum = pgEnum('calendar_provider', [
  'google',
  'outlook',
  'apple',   // Phase 2 — CalDAV / iCloud
  'caldav',  // Phase 2 — generic CalDAV
])

export const calendarStatusEnum = pgEnum('calendar_status', [
  'connected',
  'disconnected', // token refresh failed; booking pages disabled; host email alert sent
])

export const videoProviderEnum = pgEnum('video_provider', [
  'zoom',
  'teams', // Phase 2
])

export const jobTypeEnum = pgEnum('job_type', [
  'send_confirmation_invitee',
  'send_confirmation_host',
  'create_calendar_event',
  'generate_video_link',
  'send_reminder_24h',
  'send_reminder_1h',
  'send_cancellation_invitee',
  'send_cancellation_host',
  'send_reschedule_invitee',
  'send_reschedule_host',
  'noshow_check',     // Phase 2
  'gdpr_data_export', // Phase 2
])

export const jobStatusEnum = pgEnum('job_status', [
  'pending',
  'running',
  'completed',
  'failed',
  'cancelled',
])

// ─────────────────────────────────────────────────────────────────────────────
// USERS — Auth base tables (managed by Better Auth) + Schedica extensions
// ─────────────────────────────────────────────────────────────────────────────

export const users = pgTable('users', {
  id: text('id').primaryKey(),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  emailVerified: boolean('email_verified').notNull().default(false),
  image: text('image'),                        // profile photo S3 key
  username: text('username').unique(),          // booking URL: schedica.com/username
  timezone: text('timezone').default('UTC'),   // IANA timezone
  onboardingStep: integer('onboarding_step').default(0), // 0–5; 5 = complete
  onboardingDone: boolean('onboarding_done').notNull().default(false),
  isPlatformAdmin: boolean('is_platform_admin').notNull().default(false),
  banned: boolean('banned').notNull().default(false),
  banReason: text('ban_reason'),
  banExpires: timestamp('ban_expires'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
})

export const sessions = pgTable('sessions', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  token: text('token').notNull().unique(),
  expiresAt: timestamp('expires_at').notNull(),
  ipAddress: text('ip_address'),
  userAgent: text('user_agent'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
})

export const accounts = pgTable('accounts', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  provider: text('provider').notNull(),         // "google" | "microsoft" | "credential"
  providerAccountId: text('provider_account_id').notNull(),
  accessToken: text('access_token'),
  refreshToken: text('refresh_token'),
  accessTokenExpiresAt: timestamp('access_token_expires_at'),
  refreshTokenExpiresAt: timestamp('refresh_token_expires_at'),
  scope: text('scope'),
  idToken: text('id_token'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
})

export const verifications = pgTable('verifications', {
  id: text('id').primaryKey(),
  identifier: text('identifier').notNull(), // email being verified
  value: text('value').notNull(),           // 6-digit code or magic link token
  expiresAt: timestamp('expires_at').notNull(),
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow(),
})

export const userProfiles = pgTable('user_profiles', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().unique().references(() => users.id, { onDelete: 'cascade' }),
  displayName: text('display_name'),
  jobTitle: text('job_title'),
  company: text('company'),
  bio: text('bio'),           // Phase 2 — up to 200 chars
  websiteUrl: text('website_url'), // Phase 2
  theme: themeEnum('theme').default('system'),
  dateFormat: dateFormatEnum('date_format').default('MM/DD/YYYY'),
  timeFormat: timeFormatEnum('time_format').default('12h'),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
})

export const userBranding = pgTable('user_branding', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().unique().references(() => users.id, { onDelete: 'cascade' }),
  brandPrimaryColor: text('brand_primary_color').default('#0070f3'),
  brandTextColor: text('brand_text_color').default('#ffffff'),
  logoUrl: text('logo_url'),                  // S3 key for org logo
  welcomeMessage: text('welcome_message'),
  confirmationMessage: text('confirmation_message'), // shown after booking + in email
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
})

// When a user changes their username, old URL redirects to new URL for 30 days.
// Without this, any old booking link in an email signature would 404 immediately.
export const usernameRedirects = pgTable('username_redirects', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  oldUsername: text('old_username').notNull(),
  newUsername: text('new_username').notNull(),
  expiresAt: timestamp('expires_at').notNull(), // 30 days from change
  createdAt: timestamp('created_at').notNull().defaultNow(),
})

// ─────────────────────────────────────────────────────────────────────────────
// EVENT TYPES — Meeting templates + availability + questions + policies
// ─────────────────────────────────────────────────────────────────────────────

export const eventTypes = pgTable('event_types', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  availabilityScheduleId: text('availability_schedule_id'), // FK to availability_schedules
  name: text('name').notNull(),
  slug: text('slug').notNull(),              // unique per user; forms booking URL path
  description: text('description'),
  locationType: locationTypeEnum('location_type').notNull().default('zoom'),
  locationValue: text('location_value'),     // video URL, address, phone number, etc.
  color: text('color').default('#0070f3'),
  isActive: boolean('is_active').notNull().default(true),
  isHidden: boolean('is_hidden').notNull().default(false), // bookable via direct link only
  position: integer('position').default(0),  // drag-and-drop order on profile page
  minimumNotice: integer('minimum_notice').default(60),    // minutes
  bookingWindow: integer('booking_window').default(60),    // days ahead
  bookingWindowType: bookingWindowTypeEnum('booking_window_type').default('rolling'),
  bufferBefore: integer('buffer_before').default(0),       // minutes
  bufferAfter: integer('buffer_after').default(0),         // minutes
  maxBookingsPerDay: integer('max_bookings_per_day'),       // null = no limit
  startTimeIncrement: integer('start_time_increment').default(30), // slot interval (min)
  requiresApproval: boolean('requires_approval').notNull().default(false),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
}, (table) => ({
  userSlugIdx: index('event_types_user_slug_idx').on(table.userId, table.slug),
  userActiveIdx: index('event_types_user_active_idx').on(table.userId, table.isActive),
}))

// Multiple durations per event type — e.g. "30 min / 60 min" on the same link
export const eventTypeDurations = pgTable('event_type_durations', {
  id: text('id').primaryKey(),
  eventTypeId: text('event_type_id').notNull().references(() => eventTypes.id, { onDelete: 'cascade' }),
  duration: integer('duration').notNull(),   // minutes
  isDefault: boolean('is_default').notNull().default(false),
})

export const cancellationPolicies = pgTable('cancellation_policies', {
  id: text('id').primaryKey(),
  eventTypeId: text('event_type_id').notNull().unique().references(() => eventTypes.id, { onDelete: 'cascade' }),
  allowCancellation: boolean('allow_cancellation').notNull().default(true),
  cutoffHours: integer('cutoff_hours').default(0),        // 0 = always allowed
  allowRescheduling: boolean('allow_rescheduling').notNull().default(true),
  rescheduleCutoffHours: integer('reschedule_cutoff_hours').default(0),
  maxReschedules: integer('max_reschedules'),              // null = unlimited
  requireCancellationReason: boolean('require_cancellation_reason').notNull().default(false),
  cancellationReasonOptions: jsonb('cancellation_reason_options'), // string[] | null
  showPolicyText: boolean('show_policy_text').notNull().default(true),
  policyText: text('policy_text'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
})

// ─────────────────────────────────────────────────────────────────────────────
// AVAILABILITY — Weekly schedule + date overrides
// ─────────────────────────────────────────────────────────────────────────────

// One global schedule per user at MVP. Multiple named schedules → Phase 2.
export const availabilitySchedules = pgTable('availability_schedules', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  name: text('name').notNull().default('Working Hours'),
  isDefault: boolean('is_default').notNull().default(true),
  timezone: text('timezone').notNull(), // IANA — stored separately from user timezone
  createdAt: timestamp('created_at').notNull().defaultNow(),
})

// Each row = one time block. Multiple rows per day = split schedule (e.g. 9–12, 14–17).
// Times stored as "HH:mm" strings in the schedule's timezone. DST handled at query time.
export const availabilityWindows = pgTable('availability_windows', {
  id: text('id').primaryKey(),
  scheduleId: text('schedule_id').notNull().references(() => availabilitySchedules.id, { onDelete: 'cascade' }),
  dayOfWeek: dayOfWeekEnum('day_of_week').notNull(),
  startTime: text('start_time').notNull(), // "09:00"
  endTime: text('end_time').notNull(),     // "17:00"
})

// Date-specific: block a day or change hours for one specific date
export const availabilityOverrides = pgTable('availability_overrides', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  date: text('date').notNull(),            // "YYYY-MM-DD"
  isBlocked: boolean('is_blocked').notNull().default(true),
  startTime: text('start_time'),           // if not blocked: custom start time for this day
  endTime: text('end_time'),               // if not blocked: custom end time for this day
  reason: text('reason'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
}, (table) => ({
  userDateIdx: index('availability_overrides_user_date_idx').on(table.userId, table.date),
}))

// ─────────────────────────────────────────────────────────────────────────────
// CUSTOM QUESTIONS — Intake form questions per event type
// ─────────────────────────────────────────────────────────────────────────────

export const eventTypeQuestions = pgTable('event_type_questions', {
  id: text('id').primaryKey(),
  eventTypeId: text('event_type_id').notNull().references(() => eventTypes.id, { onDelete: 'cascade' }),
  label: text('label').notNull(),
  type: questionTypeEnum('type').notNull(),
  isRequired: boolean('is_required').notNull().default(false),
  options: jsonb('options'),               // string[] for single_select, dropdown, multiple_select
  placeholder: text('placeholder'),
  position: integer('position').notNull().default(0),
  isActive: boolean('is_active').notNull().default(true),
})

// ─────────────────────────────────────────────────────────────────────────────
// CALENDARS — Connected calendar accounts + event cache
// ─────────────────────────────────────────────────────────────────────────────

export const connectedCalendars = pgTable('connected_calendars', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  provider: calendarProviderEnum('provider').notNull(),
  accountEmail: text('account_email').notNull(),
  accessToken: text('access_token'),
  refreshToken: text('refresh_token'),
  tokenExpiresAt: timestamp('token_expires_at'),
  status: calendarStatusEnum('status').notNull().default('connected'),
  disconnectedAt: timestamp('disconnected_at'), // set when token refresh fails
  calendarId: text('calendar_id'),
  calendarName: text('calendar_name'),
  isPrimary: boolean('is_primary').notNull().default(false),
  isConflictCheck: boolean('is_conflict_check').notNull().default(true),
  isWriteTarget: boolean('is_write_target').notNull().default(false),
  createdAt: timestamp('created_at').notNull().defaultNow(),
}, (table) => ({
  userProviderIdx: index('connected_calendars_user_provider_idx').on(table.userId, table.provider),
}))

// 5-minute TTL cache of external calendar events for free/busy lookups.
// Refreshed by a pg-boss sync job on a schedule.
export const calendarEventsCache = pgTable('calendar_events_cache', {
  id: text('id').primaryKey(),
  connectedCalendarId: text('connected_calendar_id').notNull().references(() => connectedCalendars.id, { onDelete: 'cascade' }),
  externalEventId: text('external_event_id').notNull(),
  startTime: timestamp('start_time').notNull(), // UTC
  endTime: timestamp('end_time').notNull(),      // UTC
  isBusy: boolean('is_busy').notNull().default(true),
  syncedAt: timestamp('synced_at').notNull().defaultNow(),
}, (table) => ({
  calTimeIdx: index('calendar_events_cache_cal_time_idx').on(
    table.connectedCalendarId, table.startTime, table.endTime
  ),
}))

// ─────────────────────────────────────────────────────────────────────────────
// VIDEO — Video conferencing OAuth tokens
// ─────────────────────────────────────────────────────────────────────────────

// Google Meet does NOT need a row here — it uses the Google Calendar token
// already stored in connected_calendars. Only Zoom (P1) and Teams (Phase 2) need this.
export const videoConnections = pgTable('video_connections', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  provider: videoProviderEnum('provider').notNull(),
  accountEmail: text('account_email'),
  accessToken: text('access_token').notNull(),
  refreshToken: text('refresh_token'),
  tokenExpiresAt: timestamp('token_expires_at'),
  providerUserId: text('provider_user_id'), // Zoom user ID, Teams object ID
  createdAt: timestamp('created_at').notNull().defaultNow(),
})

// ─────────────────────────────────────────────────────────────────────────────
// BOOKINGS — Booking records, answers, guests
// ─────────────────────────────────────────────────────────────────────────────

export const bookings = pgTable('bookings', {
  id: text('id').primaryKey(),
  eventTypeId: text('event_type_id').notNull().references(() => eventTypes.id),
  hostUserId: text('host_user_id').notNull().references(() => users.id),

  // Invitee details (denormalized — invitees have no Schedica account)
  inviteeName: text('invitee_name').notNull(),
  inviteeEmail: text('invitee_email').notNull(),
  inviteePhone: text('invitee_phone'),           // collected when locationType = phone
  inviteeTimezone: text('invitee_timezone').notNull(), // IANA — for dual-timezone display

  // Meeting time — always stored as UTC
  startTime: timestamp('start_time').notNull(),
  endTime: timestamp('end_time').notNull(),
  duration: integer('duration').notNull(),        // minutes; denormalized for easy display

  // Location
  locationValue: text('location_value'),          // final video URL, address, or phone
  videoLinkHost: text('video_link_host'),         // host's moderator/start link (Zoom start URL)
  videoLinkInvitee: text('video_link_invitee'),   // invitee's join link

  status: bookingStatusEnum('status').notNull().default('confirmed'),

  // Cancellation & reschedule
  cancelToken: text('cancel_token').notNull().unique(),
  rescheduleToken: text('reschedule_token').notNull().unique(),
  cancellationReason: text('cancellation_reason'),
  cancelledBy: text('cancelled_by'),              // "host" | "invitee"
  cancelledAt: timestamp('cancelled_at'),
  rescheduledFromId: text('rescheduled_from_id'), // previous booking ID when rescheduled
  rescheduleCount: integer('reschedule_count').notNull().default(0),

  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
}, (table) => ({
  hostStartTimeIdx: index('bookings_host_start_time_idx').on(table.hostUserId, table.startTime),
  hostStatusIdx: index('bookings_host_status_idx').on(table.hostUserId, table.status),
  inviteeEmailIdx: index('bookings_invitee_email_idx').on(table.inviteeEmail),
  cancelTokenIdx: index('bookings_cancel_token_idx').on(table.cancelToken),
  rescheduleTokenIdx: index('bookings_reschedule_token_idx').on(table.rescheduleToken),
}))

// Invitee's answers to intake form questions.
// questionLabel is denormalized so answers remain readable even if the question
// label is later changed or the question is deleted.
export const bookingAnswers = pgTable('booking_answers', {
  id: text('id').primaryKey(),
  bookingId: text('booking_id').notNull().references(() => bookings.id, { onDelete: 'cascade' }),
  questionId: text('question_id').references(() => eventTypeQuestions.id, { onDelete: 'set null' }),
  questionLabel: text('question_label').notNull(), // label at time of booking
  answer: text('answer').notNull(),
})

// Additional attendees added by the invitee (up to 10).
// Each guest receives all confirmation, reminder, and follow-up emails.
export const bookingGuests = pgTable('booking_guests', {
  id: text('id').primaryKey(),
  bookingId: text('booking_id').notNull().references(() => bookings.id, { onDelete: 'cascade' }),
  guestEmail: text('guest_email').notNull(),
  guestName: text('guest_name'),
})

// ─────────────────────────────────────────────────────────────────────────────
// NOTIFICATIONS — Email preferences + pg-boss job tracker
// ─────────────────────────────────────────────────────────────────────────────

export const notificationPreferences = pgTable('notification_preferences', {
  id: text('id').primaryKey(),
  userId: text('user_id').notNull().unique().references(() => users.id, { onDelete: 'cascade' }),
  bookingConfirmationEmail: boolean('booking_confirmation_email').notNull().default(true),
  bookingNotificationEmail: boolean('booking_notification_email').notNull().default(true),
  reminderEmail24h: boolean('reminder_email_24h').notNull().default(true),
  reminderEmail1h: boolean('reminder_email_1h').notNull().default(true),
  cancellationEmail: boolean('cancellation_email').notNull().default(true),
  rescheduleEmail: boolean('reschedule_email').notNull().default(true),
  fromName: text('from_name'),        // shown as sender name; default: user's full name
  replyToEmail: text('reply_to_email'), // default: user's account email
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
})

// Tracks pg-boss background jobs per booking.
// singletonKey format: "{bookingId}_{jobType}"
// This lets pg-boss cancel one specific job type for one specific booking
// without touching other bookings' jobs.
export const workflowJobs = pgTable('workflow_jobs', {
  id: text('id').primaryKey(),
  bookingId: text('booking_id').notNull().references(() => bookings.id, { onDelete: 'cascade' }),
  jobType: jobTypeEnum('job_type').notNull(),
  singletonKey: text('singleton_key').notNull(), // "{bookingId}_{jobType}"
  scheduledFor: timestamp('scheduled_for'),       // null = immediate; set for reminder jobs
  status: jobStatusEnum('status').notNull().default('pending'),
  completedAt: timestamp('completed_at'),
  failureReason: text('failure_reason'),
  retryCount: integer('retry_count').notNull().default(0),
  createdAt: timestamp('created_at').notNull().defaultNow(),
}, (table) => ({
  bookingIdx: index('workflow_jobs_booking_idx').on(table.bookingId),
  singletonKeyIdx: index('workflow_jobs_singleton_key_idx').on(table.singletonKey),
  statusIdx: index('workflow_jobs_status_idx').on(table.status),
}))

// ─────────────────────────────────────────────────────────────────────────────
// RELATIONS
// ─────────────────────────────────────────────────────────────────────────────

export const usersRelations = relations(users, ({ one, many }) => ({
  profile: one(userProfiles, { fields: [users.id], references: [userProfiles.userId] }),
  branding: one(userBranding, { fields: [users.id], references: [userBranding.userId] }),
  sessions: many(sessions),
  accounts: many(accounts),
  usernameRedirects: many(usernameRedirects),
  eventTypes: many(eventTypes),
  availabilitySchedules: many(availabilitySchedules),
  connectedCalendars: many(connectedCalendars),
  videoConnections: many(videoConnections),
  notificationPreferences: one(notificationPreferences, { fields: [users.id], references: [notificationPreferences.userId] }),
}))

export const eventTypesRelations = relations(eventTypes, ({ one, many }) => ({
  user: one(users, { fields: [eventTypes.userId], references: [users.id] }),
  durations: many(eventTypeDurations),
  cancellationPolicy: one(cancellationPolicies, { fields: [eventTypes.id], references: [cancellationPolicies.eventTypeId] }),
  questions: many(eventTypeQuestions),
  availabilitySchedule: one(availabilitySchedules, { fields: [eventTypes.availabilityScheduleId], references: [availabilitySchedules.id] }),
  bookings: many(bookings),
}))

export const availabilitySchedulesRelations = relations(availabilitySchedules, ({ one, many }) => ({
  user: one(users, { fields: [availabilitySchedules.userId], references: [users.id] }),
  windows: many(availabilityWindows),
}))

export const bookingsRelations = relations(bookings, ({ one, many }) => ({
  eventType: one(eventTypes, { fields: [bookings.eventTypeId], references: [eventTypes.id] }),
  host: one(users, { fields: [bookings.hostUserId], references: [users.id] }),
  answers: many(bookingAnswers),
  guests: many(bookingGuests),
  jobs: many(workflowJobs),
  rescheduledFrom: one(bookings, {
    fields: [bookings.rescheduledFromId],
    references: [bookings.id],
    relationName: 'rescheduleChain',
  }),
}))

export const connectedCalendarsRelations = relations(connectedCalendars, ({ one, many }) => ({
  user: one(users, { fields: [connectedCalendars.userId], references: [users.id] }),
  cachedEvents: many(calendarEventsCache),
}))
```

---

## `src/lib/db/index.ts`

Drizzle client — equivalent to `db.server.ts` in shoptimity-remix.

```typescript
import { drizzle } from 'drizzle-orm/postgres-js'
import postgres from 'postgres'
import * as schema from './schema'

declare global {
  var dbGlobal: ReturnType<typeof drizzle> | undefined
}

const client = postgres(process.env.DATABASE_URL!)

const db = global.dbGlobal ?? drizzle(client, { schema })

if (process.env.NODE_ENV !== 'production') {
  global.dbGlobal = db
}

export default db
export type DB = typeof db
```

---

## `src/lib/db/users.ts`

```typescript
import db from './index'
import { users, userProfiles, userBranding, usernameRedirects } from './schema'
import { eq, and, gt } from 'drizzle-orm'

export const DbUsers = {
  getById: async (id: string) => {
    return db.query.users.findFirst({
      where: eq(users.id, id),
      with: { profile: true, branding: true },
    })
  },

  getByUsername: async (username: string) => {
    return db.query.users.findFirst({
      where: eq(users.username, username),
      with: { profile: true, branding: true },
    })
  },

  getByEmail: async (email: string) => {
    return db.query.users.findFirst({
      where: eq(users.email, email),
    })
  },

  updateProfile: async (userId: string, data: {
    displayName?: string
    jobTitle?: string
    company?: string
    theme?: 'light' | 'dark' | 'system'
    dateFormat?: 'MM/DD/YYYY' | 'DD/MM/YYYY' | 'YYYY-MM-DD'
    timeFormat?: '12h' | '24h'
  }) => {
    return db
      .insert(userProfiles)
      .values({ id: crypto.randomUUID(), userId, ...data })
      .onConflictDoUpdate({ target: userProfiles.userId, set: { ...data, updatedAt: new Date() } })
  },

  updateBranding: async (userId: string, data: {
    brandPrimaryColor?: string
    logoUrl?: string
    welcomeMessage?: string
    confirmationMessage?: string
  }) => {
    return db
      .insert(userBranding)
      .values({ id: crypto.randomUUID(), userId, ...data })
      .onConflictDoUpdate({ target: userBranding.userId, set: { ...data, updatedAt: new Date() } })
  },

  changeUsername: async (userId: string, newUsername: string, oldUsername: string) => {
    const expiresAt = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000) // 30 days
    return db.transaction(async (tx) => {
      await tx.update(users).set({ username: newUsername, updatedAt: new Date() }).where(eq(users.id, userId))
      await tx.insert(usernameRedirects).values({
        id: crypto.randomUUID(), userId, oldUsername, newUsername, expiresAt,
      })
    })
  },

  resolveUsernameRedirect: async (oldUsername: string) => {
    return db.query.usernameRedirects.findFirst({
      where: and(
        eq(usernameRedirects.oldUsername, oldUsername),
        gt(usernameRedirects.expiresAt, new Date()),
      ),
    })
  },

  ban: async (userId: string, reason: string) => {
    return db.update(users).set({ banned: true, banReason: reason, updatedAt: new Date() }).where(eq(users.id, userId))
  },

  unban: async (userId: string) => {
    return db.update(users).set({ banned: false, banReason: null, updatedAt: new Date() }).where(eq(users.id, userId))
  },

  delete: async (userId: string) => {
    return db.delete(users).where(eq(users.id, userId))
  },
}
```

---

## `src/lib/db/event-types.ts`

```typescript
import db from './index'
import { eventTypes, eventTypeDurations, cancellationPolicies, eventTypeQuestions } from './schema'
import { eq, and } from 'drizzle-orm'

type EventTypeCreateInput = {
  userId: string
  name: string
  slug: string
  description?: string
  locationType: typeof eventTypes.$inferInsert['locationType']
  locationValue?: string
  color?: string
  minimumNotice?: number
  bookingWindow?: number
  bufferBefore?: number
  bufferAfter?: number
  maxBookingsPerDay?: number
  startTimeIncrement?: number
  durations: { duration: number; isDefault: boolean }[]
}

export const DbEventTypes = {
  getById: async (id: string) => {
    return db.query.eventTypes.findFirst({
      where: eq(eventTypes.id, id),
      with: { durations: true, cancellationPolicy: true, questions: true },
    })
  },

  getBySlug: async (userId: string, slug: string) => {
    return db.query.eventTypes.findFirst({
      where: and(eq(eventTypes.userId, userId), eq(eventTypes.slug, slug)),
      with: { durations: true, cancellationPolicy: true, questions: true },
    })
  },

  listByUser: async (userId: string, includeHidden = false) => {
    return db.query.eventTypes.findMany({
      where: and(
        eq(eventTypes.userId, userId),
        eq(eventTypes.isActive, true),
      ),
      with: { durations: true },
      orderBy: (et, { asc }) => [asc(et.position)],
    })
  },

  create: async (data: EventTypeCreateInput) => {
    const { durations, ...eventData } = data
    return db.transaction(async (tx) => {
      const [eventType] = await tx.insert(eventTypes).values({
        id: crypto.randomUUID(), ...eventData,
      }).returning()
      if (durations.length) {
        await tx.insert(eventTypeDurations).values(
          durations.map((d) => ({ id: crypto.randomUUID(), eventTypeId: eventType.id, ...d }))
        )
      }
      return eventType
    })
  },

  update: async (id: string, data: Partial<EventTypeCreateInput>) => {
    const { durations, ...eventData } = data
    return db.transaction(async (tx) => {
      const [updated] = await tx.update(eventTypes)
        .set({ ...eventData, updatedAt: new Date() })
        .where(eq(eventTypes.id, id))
        .returning()
      if (durations) {
        await tx.delete(eventTypeDurations).where(eq(eventTypeDurations.eventTypeId, id))
        await tx.insert(eventTypeDurations).values(
          durations.map((d) => ({ id: crypto.randomUUID(), eventTypeId: id, ...d }))
        )
      }
      return updated
    })
  },

  toggleActive: async (id: string, isActive: boolean) => {
    return db.update(eventTypes).set({ isActive, updatedAt: new Date() }).where(eq(eventTypes.id, id))
  },

  updatePosition: async (updates: { id: string; position: number }[]) => {
    return db.transaction(async (tx) => {
      for (const { id, position } of updates) {
        await tx.update(eventTypes).set({ position }).where(eq(eventTypes.id, id))
      }
    })
  },

  delete: async (id: string) => {
    return db.delete(eventTypes).where(eq(eventTypes.id, id))
  },

  upsertCancellationPolicy: async (eventTypeId: string, data: {
    allowCancellation?: boolean
    cutoffHours?: number
    allowRescheduling?: boolean
    rescheduleCutoffHours?: number
    maxReschedules?: number | null
    requireCancellationReason?: boolean
    cancellationReasonOptions?: string[] | null
    policyText?: string
  }) => {
    return db
      .insert(cancellationPolicies)
      .values({ id: crypto.randomUUID(), eventTypeId, ...data })
      .onConflictDoUpdate({ target: cancellationPolicies.eventTypeId, set: data })
  },

  upsertQuestions: async (eventTypeId: string, questions: {
    id?: string
    label: string
    type: typeof eventTypeQuestions.$inferInsert['type']
    isRequired: boolean
    options?: string[] | null
    placeholder?: string
    position: number
  }[]) => {
    return db.transaction(async (tx) => {
      await tx.delete(eventTypeQuestions).where(eq(eventTypeQuestions.eventTypeId, eventTypeId))
      if (questions.length) {
        await tx.insert(eventTypeQuestions).values(
          questions.map((q) => ({ id: crypto.randomUUID(), eventTypeId, ...q }))
        )
      }
    })
  },
}
```

---

## `src/lib/db/availability.ts`

```typescript
import db from './index'
import { availabilitySchedules, availabilityWindows, availabilityOverrides } from './schema'
import { eq, and, gte, lte } from 'drizzle-orm'

export const DbAvailability = {
  getScheduleByUser: async (userId: string) => {
    return db.query.availabilitySchedules.findFirst({
      where: and(eq(availabilitySchedules.userId, userId), eq(availabilitySchedules.isDefault, true)),
      with: { windows: true },
    })
  },

  upsertSchedule: async (userId: string, data: {
    name?: string
    timezone: string
    windows: { dayOfWeek: typeof availabilityWindows.$inferInsert['dayOfWeek']; startTime: string; endTime: string }[]
  }) => {
    return db.transaction(async (tx) => {
      // Get or create the default schedule
      let schedule = await tx.query.availabilitySchedules.findFirst({
        where: and(eq(availabilitySchedules.userId, userId), eq(availabilitySchedules.isDefault, true)),
      })
      if (!schedule) {
        const [created] = await tx.insert(availabilitySchedules).values({
          id: crypto.randomUUID(), userId, timezone: data.timezone, name: data.name ?? 'Working Hours',
        }).returning()
        schedule = created
      } else {
        await tx.update(availabilitySchedules)
          .set({ timezone: data.timezone })
          .where(eq(availabilitySchedules.id, schedule.id))
      }
      // Replace all windows
      await tx.delete(availabilityWindows).where(eq(availabilityWindows.scheduleId, schedule.id))
      if (data.windows.length) {
        await tx.insert(availabilityWindows).values(
          data.windows.map((w) => ({ id: crypto.randomUUID(), scheduleId: schedule.id, ...w }))
        )
      }
      return schedule
    })
  },

  getOverrides: async (userId: string, fromDate: string, toDate: string) => {
    return db.query.availabilityOverrides.findMany({
      where: and(
        eq(availabilityOverrides.userId, userId),
        gte(availabilityOverrides.date, fromDate),
        lte(availabilityOverrides.date, toDate),
      ),
    })
  },

  upsertOverride: async (userId: string, date: string, data: {
    isBlocked: boolean
    startTime?: string | null
    endTime?: string | null
    reason?: string
  }) => {
    const existing = await db.query.availabilityOverrides.findFirst({
      where: and(eq(availabilityOverrides.userId, userId), eq(availabilityOverrides.date, date)),
    })
    if (existing) {
      return db.update(availabilityOverrides).set(data).where(eq(availabilityOverrides.id, existing.id))
    }
    return db.insert(availabilityOverrides).values({ id: crypto.randomUUID(), userId, date, ...data })
  },

  deleteOverride: async (userId: string, date: string) => {
    return db.delete(availabilityOverrides).where(
      and(eq(availabilityOverrides.userId, userId), eq(availabilityOverrides.date, date))
    )
  },
}
```

---

## `src/lib/db/bookings.ts`

```typescript
import db from './index'
import { bookings, bookingAnswers, bookingGuests } from './schema'
import { eq, and, gte, lte, desc, asc } from 'drizzle-orm'

type BookingCreateInput = {
  eventTypeId: string
  hostUserId: string
  inviteeName: string
  inviteeEmail: string
  inviteePhone?: string
  inviteeTimezone: string
  startTime: Date
  endTime: Date
  duration: number
  locationValue?: string
  cancelToken: string
  rescheduleToken: string
  answers?: { questionId?: string; questionLabel: string; answer: string }[]
  guests?: { guestEmail: string; guestName?: string }[]
}

export const DbBookings = {
  create: async (data: BookingCreateInput) => {
    const { answers = [], guests = [], ...bookingData } = data
    return db.transaction(async (tx) => {
      const [booking] = await tx.insert(bookings).values({
        id: crypto.randomUUID(), ...bookingData,
      }).returning()
      if (answers.length) {
        await tx.insert(bookingAnswers).values(
          answers.map((a) => ({ id: crypto.randomUUID(), bookingId: booking.id, ...a }))
        )
      }
      if (guests.length) {
        await tx.insert(bookingGuests).values(
          guests.map((g) => ({ id: crypto.randomUUID(), bookingId: booking.id, ...g }))
        )
      }
      return booking
    })
  },

  getById: async (id: string) => {
    return db.query.bookings.findFirst({
      where: eq(bookings.id, id),
      with: { answers: true, guests: true, eventType: true },
    })
  },

  getByCancelToken: async (token: string) => {
    return db.query.bookings.findFirst({
      where: eq(bookings.cancelToken, token),
      with: { answers: true, eventType: true },
    })
  },

  getByRescheduleToken: async (token: string) => {
    return db.query.bookings.findFirst({
      where: eq(bookings.rescheduleToken, token),
      with: { answers: true, eventType: true },
    })
  },

  listUpcoming: async (hostUserId: string, limit = 25, offset = 0) => {
    return db.query.bookings.findMany({
      where: and(eq(bookings.hostUserId, hostUserId), eq(bookings.status, 'confirmed'), gte(bookings.startTime, new Date())),
      orderBy: asc(bookings.startTime),
      limit,
      offset,
      with: { answers: true, guests: true },
    })
  },

  listPast: async (hostUserId: string, limit = 25, offset = 0) => {
    return db.query.bookings.findMany({
      where: and(eq(bookings.hostUserId, hostUserId), lte(bookings.startTime, new Date())),
      orderBy: desc(bookings.startTime),
      limit,
      offset,
      with: { answers: true },
    })
  },

  cancel: async (id: string, data: { cancelledBy: 'host' | 'invitee'; cancellationReason?: string }) => {
    return db.update(bookings).set({
      status: 'cancelled',
      cancelledBy: data.cancelledBy,
      cancellationReason: data.cancellationReason,
      cancelledAt: new Date(),
      updatedAt: new Date(),
    }).where(eq(bookings.id, id))
  },

  updateVideoLink: async (id: string, videoLinkHost: string, videoLinkInvitee: string) => {
    return db.update(bookings).set({ videoLinkHost, videoLinkInvitee, updatedAt: new Date() }).where(eq(bookings.id, id))
  },

  incrementRescheduleCount: async (id: string) => {
    const booking = await db.query.bookings.findFirst({ where: eq(bookings.id, id) })
    if (!booking) throw new Error('Booking not found')
    return db.update(bookings).set({ rescheduleCount: booking.rescheduleCount + 1, updatedAt: new Date() }).where(eq(bookings.id, id))
  },
}
```

---

## `src/lib/db/calendars.ts`

```typescript
import db from './index'
import { connectedCalendars, calendarEventsCache } from './schema'
import { eq, and, gte, lte } from 'drizzle-orm'

export const DbCalendars = {
  getByUser: async (userId: string) => {
    return db.query.connectedCalendars.findMany({
      where: and(eq(connectedCalendars.userId, userId), eq(connectedCalendars.status, 'connected')),
    })
  },

  getConflictCheckCalendars: async (userId: string) => {
    return db.query.connectedCalendars.findMany({
      where: and(
        eq(connectedCalendars.userId, userId),
        eq(connectedCalendars.isConflictCheck, true),
        eq(connectedCalendars.status, 'connected'),
      ),
    })
  },

  connect: async (userId: string, data: {
    provider: typeof connectedCalendars.$inferInsert['provider']
    accountEmail: string
    accessToken: string
    refreshToken?: string
    tokenExpiresAt?: Date
    calendarId?: string
    calendarName?: string
    isWriteTarget?: boolean
  }) => {
    return db.insert(connectedCalendars).values({
      id: crypto.randomUUID(), userId, ...data,
    }).onConflictDoUpdate({
      target: [connectedCalendars.userId, connectedCalendars.provider],
      set: { ...data, status: 'connected', disconnectedAt: null },
    })
  },

  updateTokens: async (id: string, accessToken: string, tokenExpiresAt: Date) => {
    return db.update(connectedCalendars).set({ accessToken, tokenExpiresAt }).where(eq(connectedCalendars.id, id))
  },

  markDisconnected: async (id: string) => {
    // Called immediately when token refresh fails.
    // Booking pages are disabled and host alert email is sent by the caller.
    return db.update(connectedCalendars).set({
      status: 'disconnected',
      disconnectedAt: new Date(),
    }).where(eq(connectedCalendars.id, id))
  },

  reconnect: async (id: string, accessToken: string, refreshToken: string, tokenExpiresAt: Date) => {
    return db.update(connectedCalendars).set({
      accessToken, refreshToken, tokenExpiresAt,
      status: 'connected', disconnectedAt: null,
    }).where(eq(connectedCalendars.id, id))
  },

  disconnect: async (id: string) => {
    return db.delete(connectedCalendars).where(eq(connectedCalendars.id, id))
  },

  getCachedBusySlots: async (calendarId: string, from: Date, to: Date) => {
    return db.query.calendarEventsCache.findMany({
      where: and(
        eq(calendarEventsCache.connectedCalendarId, calendarId),
        eq(calendarEventsCache.isBusy, true),
        gte(calendarEventsCache.startTime, from),
        lte(calendarEventsCache.endTime, to),
      ),
    })
  },

  upsertCachedEvent: async (data: {
    connectedCalendarId: string
    externalEventId: string
    startTime: Date
    endTime: Date
    isBusy: boolean
  }) => {
    return db.insert(calendarEventsCache).values({ id: crypto.randomUUID(), ...data })
      .onConflictDoUpdate({
        target: [calendarEventsCache.connectedCalendarId, calendarEventsCache.externalEventId],
        set: { startTime: data.startTime, endTime: data.endTime, isBusy: data.isBusy, syncedAt: new Date() },
      })
  },
}
```

---

## `src/lib/db/video.ts`

```typescript
import db from './index'
import { videoConnections } from './schema'
import { eq, and } from 'drizzle-orm'

export const DbVideo = {
  getByUser: async (userId: string, provider: typeof videoConnections.$inferInsert['provider']) => {
    return db.query.videoConnections.findFirst({
      where: and(eq(videoConnections.userId, userId), eq(videoConnections.provider, provider)),
    })
  },

  connect: async (userId: string, data: {
    provider: typeof videoConnections.$inferInsert['provider']
    accountEmail?: string
    accessToken: string
    refreshToken?: string
    tokenExpiresAt?: Date
    providerUserId?: string
  }) => {
    return db.insert(videoConnections).values({ id: crypto.randomUUID(), userId, ...data })
      .onConflictDoUpdate({
        target: [videoConnections.userId, videoConnections.provider],
        set: { ...data },
      })
  },

  updateTokens: async (id: string, accessToken: string, tokenExpiresAt: Date) => {
    return db.update(videoConnections).set({ accessToken, tokenExpiresAt }).where(eq(videoConnections.id, id))
  },

  disconnect: async (userId: string, provider: typeof videoConnections.$inferInsert['provider']) => {
    return db.delete(videoConnections).where(
      and(eq(videoConnections.userId, userId), eq(videoConnections.provider, provider))
    )
  },
}
```

---

## `src/lib/db/notifications.ts`

```typescript
import db from './index'
import { notificationPreferences, workflowJobs } from './schema'
import { eq, and } from 'drizzle-orm'

export const DbNotifications = {
  getPrefs: async (userId: string) => {
    return db.query.notificationPreferences.findFirst({
      where: eq(notificationPreferences.userId, userId),
    })
  },

  upsertPrefs: async (userId: string, data: {
    bookingConfirmationEmail?: boolean
    bookingNotificationEmail?: boolean
    reminderEmail24h?: boolean
    reminderEmail1h?: boolean
    cancellationEmail?: boolean
    rescheduleEmail?: boolean
    fromName?: string
    replyToEmail?: string
  }) => {
    return db
      .insert(notificationPreferences)
      .values({ id: crypto.randomUUID(), userId, ...data })
      .onConflictDoUpdate({ target: notificationPreferences.userId, set: { ...data, updatedAt: new Date() } })
  },

  logJob: async (data: {
    bookingId: string
    jobType: typeof workflowJobs.$inferInsert['jobType']
    singletonKey: string
    scheduledFor?: Date
  }) => {
    return db.insert(workflowJobs).values({ id: crypto.randomUUID(), ...data })
  },

  markJobComplete: async (singletonKey: string) => {
    return db.update(workflowJobs).set({ status: 'completed', completedAt: new Date() })
      .where(eq(workflowJobs.singletonKey, singletonKey))
  },

  markJobFailed: async (singletonKey: string, failureReason: string, retryCount: number) => {
    return db.update(workflowJobs).set({
      status: retryCount >= 3 ? 'failed' : 'pending',
      failureReason,
      retryCount,
      completedAt: retryCount >= 3 ? new Date() : null,
    }).where(eq(workflowJobs.singletonKey, singletonKey))
  },

  getFailedJobs: async () => {
    return db.query.workflowJobs.findMany({
      where: eq(workflowJobs.status, 'failed'),
      with: { booking: true },
      orderBy: (j, { desc }) => [desc(j.createdAt)],
    })
  },
}
```

---

## `drizzle.config.ts`

```typescript
import { defineConfig } from 'drizzle-kit'

export default defineConfig({
  schema: './src/lib/db/schema.ts',  // single schema file
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
  schemaFilter: ['public'], // required — excludes pgboss schema tables
})
```

---

## How to Use in Server Actions / API Routes

```typescript
// Example: creating a booking in a Server Action
import { DbBookings } from '@/lib/db/bookings'
import { DbNotifications } from '@/lib/db/notifications'

const booking = await DbBookings.create({ ... })
await DbNotifications.logJob({
  bookingId: booking.id,
  jobType: 'send_confirmation_invitee',
  singletonKey: `${booking.id}_send_confirmation_invitee`,
})
```

---

## Table Summary

| Table | Exported from |
|-------|--------------|
| `users`, `sessions`, `accounts`, `verifications` | schema.ts → DbUsers |
| `user_profiles`, `user_branding`, `username_redirects` | schema.ts → DbUsers |
| `event_types`, `event_type_durations`, `cancellation_policies`, `event_type_questions` | schema.ts → DbEventTypes |
| `availability_schedules`, `availability_windows`, `availability_overrides` | schema.ts → DbAvailability |
| `connected_calendars`, `calendar_events_cache` | schema.ts → DbCalendars |
| `video_connections` | schema.ts → DbVideo |
| `bookings`, `booking_answers`, `booking_guests` | schema.ts → DbBookings |
| `notification_preferences`, `workflow_jobs` | schema.ts → DbNotifications |
