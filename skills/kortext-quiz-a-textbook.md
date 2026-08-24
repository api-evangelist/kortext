---
name: kortext-quiz-a-textbook
description: >-
  Generate a quiz from a piece of Kortext content, walk a learner through the questions, record
  their answers, and read back the resulting confidence score and knowledge-item coverage.
api: Kortext Labs AI Study Tools API
base_url: https://api-demo.labs.kortext.com
spec: openapi/kortext-labs-api-openapi.json
generated: '2026-08-23'
method: generated
source: >-
  Grounded in openapi/kortext-labs-api-openapi.json. Every operationId below was read from the
  spec; none is invented.
operations:
  - tutor_create_session_tutor_v1_quiz_session_post
  - tutor_create_session_question_tutor_v1_quiz_session_question_post
  - tutor_create_session_answer_attempt_tutor_v1_quiz_session_answer_post
  - tutor_create_session_answer_attempt_tutor_v1_quiz_session_answer__attempt_id__patch
  - tutor_get_session_detail_tutor_v1_quiz_session__session_id__detail_get
  - tutor_get_sessions_grouped_by_content_id_tutor_v1_quiz_sessions_get
  - get_confidence_score_tutor_v1_content__content_id__confidence_score_get
  - get_knowledge_items_for_content_tutor_v1_content__content_id__knowledge_items_get
  - has_process_content_parts_tutor_v1_content__content_id__has_content_parts_get
  - index_content_tutor_v1_content__content_id__index_post
  - get_task_status_tutor_v1_content_content_status__task_id__get
---

# Quiz a Kortext textbook

## Before you start

- **Access is not self-serve.** Kortext runs no developer program and issues no public API keys.
  You need a bearer JWT from an institutional Kortext account, obtained from
  `https://app.kortext.com/account/token`. Do not attempt this flow without one.
- **The published contract is on a non-production host.** `https://api-demo.labs.kortext.com` is
  Kortext's labs demo environment. Treat anything you do there as disposable.
- **You must already hold a `contentId`.** This API exposes no operation to list a user's shelf.
  The identifier comes from the Kortext reader, not from this contract.
- Every operation below is secured with `HTTPBearer` (`Authorization: Bearer <jwt>`). A missing or
  invalid token returns **403** with `{"detail":"Not authenticated"}` — note that this API uses
  403 where 401 is conventional.

## 1. Confirm the content is indexed

```
GET /tutor/v1/content/{content_id}/has-content-parts
```
`has_process_content_parts_tutor_v1_content__content_id__has_content_parts_get`

If it reports no parts, index it first:

```
POST /tutor/v1/content/{content_id}/index
```
`index_content_tutor_v1_content__content_id__index_post`

Indexing is asynchronous. Poll:

```
GET /tutor/v1/content/content/status/{task_id}
```
`get_task_status_tutor_v1_content_content_status__task_id__get`

until `status` settles. `ContentProcessingStatusDTO` carries `processedParts` and `totalParts` —
use them for progress, not for a completion test.

> **Reversibility.** Indexing is undone with
> `POST /tutor/v1/content/{content_id}/delete-index`
> (`delete_index_tutor_v1_content__content_id__delete_index_post`). Kortext states **no window** for
> this. Do not tell a user how long they have to undo it.

## 2. Create the quiz session

```
POST /tutor/v1/quiz/session
```
`tutor_create_session_tutor_v1_quiz_session_post`
Body: `CreateQuizSessionDto` — `contentId`, `pageId`, `pageNumber`, `contentType`.

Two alternative entry points exist when the source is not shelf content:
- `POST /tutor/v1/quiz/session/url` (`CreateQuizSessionFromUrlDto`: `contentId`, `url`, `title`,
  `numQuestions`) — returns **400 "Invalid URL or scraping failed"** if the URL cannot be read.
- `POST /tutor/v1/quiz/session/text` (`CreateQuizSessionFromTextDto`: `contentId`, `text`, `title`,
  `numQuestions`).

> **There is no idempotency key.** This contract declares zero header parameters. If you retry a
> creation because of a timeout you will create a **second session**, and there is no delete
> operation for a quiz session. Confirm before retrying, or reconcile with
> `GET /tutor/v1/quiz/sessions` first.

## 3. Generate questions

```
POST /tutor/v1/quiz/session/question
```
`tutor_create_session_question_tutor_v1_quiz_session_question_post`
Body: `CreateQuizSessionQuestionDto` — `quizSessionId`.

Returns `PublicQuizSessionQuestionDto`: the question, its `contentPart` and its `knowledgeItem`,
with `answers` as `PublicQuizAnswerDto[]`. The **public** projection deliberately omits
`isCorrect` — never try to read the answer key from this response, and never present a "correct"
flag you have not received from step 4.

Generated questions cannot be deleted. Generate one at a time as the learner progresses.

## 4. Record answers

```
POST /tutor/v1/quiz/session/answer
```
`tutor_create_session_answer_attempt_tutor_v1_quiz_session_answer_post`
Body: `SetQuizAnswerAttemptDto` — `sessionId`, `questionId`, `answerId`.
Returns `QuizAttemptDto` including `isCorrect`.

To correct a mis-recorded selection:

```
PATCH /tutor/v1/quiz/session/answer/{attempt_id}
```
`tutor_create_session_answer_attempt_tutor_v1_quiz_session_answer__attempt_id__patch`
Body: `UpdateQuizAnswerAttemptDto` — `answerId`.

> Amending an attempt is the reversal path here. Whether it recomputes the confidence score is
> **not stated in the contract** — re-read the score in step 5 rather than assuming.

## 5. Read the outcome

```
GET /tutor/v1/quiz/session/{session_id}/detail
GET /tutor/v1/quiz/sessions
GET /tutor/v1/content/{content_id}/confidence-score
GET /tutor/v1/content/{content_id}/knowledge-items
GET /tutor/v1/content/{content_id}/quiz-sessions
```

`ContentQuizStatsDto` is the fullest view: `confidenceScore`, `totalQuestionsAttempted`,
`totalCorrectAnswers`, `knowledgeItemsWithCorrectAnswers` against `totalKnowledgeItems`, and
`totalTime`.

## Errors

| Status | Meaning | Parseable? |
|---|---|---|
| 400 | Invalid request | No schema declared |
| 403 | Not authenticated / unauthorized access | No schema declared |
| 404 | Quiz session, question, answer or content not found | No schema declared |
| 422 | Validation error — `{"detail":[{"loc":[...],"msg":"...","type":"..."}]}` | **Yes** — `HTTPValidationError` |
| 500 | Internal server error | No schema declared |

Only **422** is machine-parseable. This API is not RFC 9457 and returns no correlation identifier,
so a 500 gives you nothing to quote to support. See `errors/kortext-problem-types.yml`.

## Rate limits

None published, and none observed in response headers. This flow drives LLM generation on every
question — back off conservatively on 5xx. See `rate-limits/kortext-rate-limits.yml`.

## Note on the duplicated surface

Every `/tutor/v1/content/*` operation in this skill also exists verbatim under `/chat/v1/content/*`
with a differently-suffixed operationId. They are the same 17 behaviours mounted twice. Pick one
prefix and stay on it.
