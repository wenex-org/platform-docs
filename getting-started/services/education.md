# Education

**Port:** REST `:3160` · gRPC `:5160`

E-learning domain. A deliberately **thin** service: it owns only the two concepts that do not already exist on the platform — the **course curriculum** (course → chapters → lessons) and the **enrollment** (the link between a learner and a course, with progress and payment). Everything else is reused by `ObjectId` reference to existing services.

## Reused platform modules (no logic duplicated)

| Education need | Existing module | Linked via |
| --- | --- | --- |
| Video / material / cover files | `special/files` | `thumbnail`, `lessons[].video` |
| Live session (webinar) | `general/events` | `course.event` |
| Comments / rating / Q&A | `general/comments` | by `course` id |
| Support tickets | `content/tickets` | by `course` id |
| Mentoring / group chat | `conjoint/channels` + `members` | by `course` id |
| Payment / order / refund | `financial/invoices` + `transactions` | `enrollment.invoice` |
| Learning progress events | `general/activities` | by `enrollment` id |
| Instructor / student identity | `identity/users` + `profiles` | `course.instructor`, `enrollment.student` |
| Pricing model (template) | `career/services` | `Course` mirrors its shape |

## Collections

| Collection | Path | Purpose |
| --- | --- | --- |
| Courses | `/education/courses` | Course definition + embedded curriculum (chapters → lessons → assessment + quiz questions) |
| Enrollments | `/education/enrollments` | Learner ↔ course link with progress, final grade, and payment reference |
| Submissions | `/education/submissions` | Gradebook: per-attempt quiz/exercise/exam record with score, feedback, files |

> **Grading model.** The course curriculum is the shared blueprint (same for every learner), so a graded lesson (`type` `QUIZ`/`EXERCISE`/`EXAM`) only carries its **definition** — an embedded `assessment` (max marks, pass mark, weight, due date, attempts, and `questions[]` for quizzes). A learner's actual grade is **per-(student, lesson, attempt)** and lives in `education/submissions`; a lightweight roll-up sits on `enrollment.lesson_progress[]` and `enrollment.final_grade`.

## `education/courses`

A sellable course, modeled on `career/services` (same pricing/rating shape) plus an embedded curriculum.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `type` | ✅ | `CourseType` | `VIDEO`, `LIVE`, `HYBRID` |
| `name` | ✅ | string | Course title |
| `status` | ✅ | `Status` | `ACTIVE`, `INACTIVE` |
| `instructor` | ✅ | MongoId | `identity/users` ID of the instructor |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `state` | `State` | `PENDING` (default), `APPROVED`, `REJECTED`, `VERIFIED`, `UNKNOWN` |
| `level` | `CourseLevel` | `BEGINNER`, `INTERMEDIATE`, `ADVANCED` |
| `language` | string | Primary language tag |
| `event` | MongoId | Live session — raw `general/events` ID |
| `categories` | string[] | Category tags |
| `currency` | MongoId | `financial/currencies` ID |
| `price` | number | List price |
| `profit` | number | Profit amount |
| `discount` | number | Discount amount |
| `rate` | number | Learning rate (platform-bounded) |
| `votes` | number | Number of ratings |
| `rating` | number | Aggregate rating |
| `thumbnail` | MongoId | Cover image — `special/files` ID |
| `chapters` | `Chapter[]` | Embedded curriculum (see below) |

### Embedded `Chapter`

| Field | Type | Description |
| --- | --- | --- |
| `title` | string | Chapter title |
| `order` | number | Display order |
| `lessons` | `Lesson[]` | Embedded lessons |

### Embedded `Lesson`

| Field | Type | Description |
| --- | --- | --- |
| `title` | string | Lesson title |
| `type` | `LessonType` | `VIDEO`, `ARTICLE`, `QUIZ`, `EXERCISE`, `EXAM`, `LIVE` |
| `order` | number | Display order |
| `content` | string | Article body or external URL |
| `topics` | string[] | Covered topics |
| `is_preview` | boolean | Free preview lesson |
| `video` | MongoId | `special/files` ID of the video |
| `duration` | number | Length in seconds |
| `assessment` | `Assessment` | Present for `QUIZ`/`EXERCISE`/`EXAM` lessons |

### Embedded `Assessment` (on graded lessons)

| Field | Type | Description |
| --- | --- | --- |
| `max` | number | Total marks |
| `pass_mark` | number | Minimum passing marks |
| `weight` | number | Contribution to `enrollment.final_grade` |
| `attempts_allowed` | number | Max attempts |
| `due_date` | Date | Deadline |
| `questions` | `Question[]` | For `QUIZ`/`EXAM` — `text`, `type` (`MCQ`/`TRUE_FALSE`/`MULTI_SELECT`/`SHORT`/`ESSAY`), `options[]`, `answers[]`, `marks` |

> File fields (`thumbnail`, `lessons[].video`) and `event` are raw MongoIds into `special/files` / `general/events` — resolve them with a separate read against those services.

## `education/enrollments`

A learner's enrollment in a course.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `course` | ✅ | MongoId | `education/courses` ID |
| `student` | ✅ | MongoId | `identity/users` ID |
| `status` | ✅ | `EnrollmentStatus` | `ACTIVE`, `SUSPENDED`, `COMPLETED`, `CANCELLED` |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `method` | `EnrollmentMethod` | `MANUAL`, `SELF` |
| `progress` | number | Completion percentage (0–100) |
| `ticket_code` | string | Access / coupon code |
| `invoice` | MongoId | `financial/invoices` ID for the purchase |
| `started_at` | Date | When learning started |
| `completed_at` | Date | When the course was completed |

### Population

| Path | Collection |
| --- | --- |
| `course` | `education/courses` |

## `education/submissions`

A single graded attempt by a learner for one graded lesson (quiz / exercise / exam). This is the gradebook — one document per attempt.

### Required Create Fields

| Field | Required | Type | Description |
| --- | :---: | --- | --- |
| `enrollment` | ✅ | MongoId | `education/enrollments` ID |
| `course` | ✅ | MongoId | `education/courses` ID |
| `lesson` | ✅ | string | Embedded course lesson `_id` |
| `student` | ✅ | MongoId | `identity/users` ID (denormalized for reporting) |
| `type` | ✅ | `SubmissionType` | `QUIZ`, `EXERCISE`, `EXAM` |
| `status` | ✅ | `SubmissionStatus` | `DRAFT`, `SUBMITTED`, `GRADING`, `GRADED`, `RETURNED`, `LATE` |

### Optional Fields

| Field | Type | Description |
| --- | --- | --- |
| `attempt` | number | Attempt number (default 1) |
| `answers` | string[] | Quiz answers (aligned to `assessment.questions[]`) |
| `content` | string | Free-text answer (exercise) |
| `attachments` | MongoId[] | Submitted files — `special/files` IDs |
| `score` / `max` / `percentage` | number | Result |
| `passed` | boolean | Pass/fail |
| `feedback` | string | Grader feedback |
| `graded_by` | MongoId | `identity/users` ID of the grader |
| `graded_at` / `submitted_at` | Date | Timestamps |

> Quiz submissions can be auto-scored from `answers` vs the lesson's `assessment.questions[].answers`; exercise/exam submissions are graded manually (`feedback` + `graded_by`). Multiple attempts are separate submission documents.

## Query Tips

- List a learner's courses by filtering `enrollments` on `student` and `status`.
- Course catalog pages filter `courses` by `categories`, `type`, `state: APPROVED`.
- Progress history is event-sourced in `general/activities` — query it by the enrollment id rather than storing a log on the enrollment.
