---
name: kortext-study-artifacts-from-content
description: >-
  Turn an indexed piece of Kortext content into study artifacts — flashcards, mnemonics, an idea
  compass, a reading plan, a Mermaid visualisation, an extracted-wisdom summary, or a two-voice AI
  podcast — and manage what persists afterwards.
api: Kortext Labs AI Study Tools API
base_url: https://api-demo.labs.kortext.com
spec: openapi/kortext-labs-api-openapi.json
generated: '2026-08-23'
method: generated
source: >-
  Grounded in openapi/kortext-labs-api-openapi.json. Every operationId below was read from the
  spec; none is invented.
operations:
  - index_content_tutor_v1_content__content_id__index_post
  - delete_index_tutor_v1_content__content_id__delete_index_post
  - get_my_indexed_files_tutor_v1_content_my_indexed_files_get
  - get_all_task_statuses_tutor_v1_content_tasks_get
  - generate_flashcards_tutor_v1_content_to_flashcards_post
  - generate_mnemonics_tutor_v1_content_mnemonics_post
  - generate_idea_compass_tutor_v1_content_idea_compass_post
  - generate_reading_plan_tutor_v1_content_reading_plan_post
  - visualise_content_tutor_v1_content_visualise_post
  - extract_wisdom_tutor_v1_content__content_id__wisdom_get
  - get_wisdom_ids_for_user_tutor_v1_content_wisdom_ids_get
  - get_wisdom_by_id_tutor_v1_content_wisdom__wisdom_id__get
  - create_podcast_podcast_v1_content__content_id__podcast_post
  - list_podcasts_podcast_v1_podcasts_get
  - get_podcast_podcast_v1_podcasts__podcast_id__get
  - delete_podcast_podcast_v1_podcasts__podcast_id__delete
---

# Generate study artifacts from Kortext content

## Before you start

Same access rules as every Kortext flow: an institutional bearer JWT from
`https://app.kortext.com/account/token`, a `contentId` you already hold from the Kortext reader,
and a published contract that lives only on the labs demo host. There is no self-serve signup.

## 1. Make sure the content is indexed

```
GET  /tutor/v1/content/my-indexed-files        get_my_indexed_files_tutor_v1_content_my_indexed_files_get
POST /tutor/v1/content/{content_id}/index      index_content_tutor_v1_content__content_id__index_post
GET  /tutor/v1/content/tasks                   get_all_task_statuses_tutor_v1_content_tasks_get
```

`get_my_indexed_files_*` returns `IndexedFileDto[]` (`contentId`, `format`, `title`). Neither this
operation nor `/tutor/v1/content/tasks` is paginated — they return unbounded arrays. Budget for
that on a heavy account.

## 2. Ephemeral generators — nothing persists

These five return their artifact in the response body. Nothing is stored, so nothing needs undoing.
Re-running is safe and is the correct way to recover from a failure.

```
POST /tutor/v1/content/to-flashcards   generate_flashcards_tutor_v1_content_to_flashcards_post
POST /tutor/v1/content/mnemonics       generate_mnemonics_tutor_v1_content_mnemonics_post
POST /tutor/v1/content/idea-compass    generate_idea_compass_tutor_v1_content_idea_compass_post
POST /tutor/v1/content/reading-plan    generate_reading_plan_tutor_v1_content_reading_plan_post
POST /tutor/v1/content/visualise       visualise_content_tutor_v1_content_visualise_post
```

`visualise` takes `VisualisationRequest` (`content_id`, `type`, `prompt`) and returns
`VisualisationResponse` — a single `mermaidMarkdown` string. An unsupported `type` returns
**400 "Invalid diagram type"**; the contract does not enumerate the valid types, so probe rather
than guess, and surface the 400 to the user rather than retrying blind.

Note the snake_case break: `visualise` uses `content_id`, everything else uses `contentId`.

## 3. Wisdom — persists, and is listable but not deletable

```
GET /tutor/v1/content/{content_id}/wisdom   extract_wisdom_tutor_v1_content__content_id__wisdom_get
GET /tutor/v1/content/wisdom-ids            get_wisdom_ids_for_user_tutor_v1_content_wisdom_ids_get
GET /tutor/v1/content/wisdom/{wisdom_id}    get_wisdom_by_id_tutor_v1_content_wisdom__wisdom_id__get
```

`extract_wisdom_*` is a GET that creates a stored `WisdomDto` (`id`, `contentId`, `contentTitle`,
`wisdom`, `createdAt`). **There is no delete operation for wisdom.** Tell the user that before you
generate it on their behalf — it is a one-way write dressed as a read.

## 4. Podcasts — expensive, persistent, and the one thing you can delete

```
POST   /podcast/v1/content/{content_id}/podcast   create_podcast_podcast_v1_content__content_id__podcast_post
GET    /podcast/v1/podcasts                       list_podcasts_podcast_v1_podcasts_get
GET    /podcast/v1/podcasts/{podcast_id}          get_podcast_podcast_v1_podcasts__podcast_id__get
DELETE /podcast/v1/podcasts/{podcast_id}          delete_podcast_podcast_v1_podcasts__podcast_id__delete
```

Creation body is `multipart/form-data`: `title`, `voice1`, `voice2`. The response is
`PodcastResponseDto` — `id`, `title`, `script`, `audioBlobUrl`, `status`, `createdAt`,
`podcastMetadata`. Treat `status` as the completion signal; the contract does not describe its
values, so do not branch on a value you have not observed.

`GET /podcast/v1/podcasts` is the **only paginated operation in the whole API**: `page` (default 1)
and `pageSize` (default 10) as query parameters, returning `PaginatedPodcastResponse` with `items`,
`total`, `page`, `pageSize`, `totalPages`.

`DELETE` returns **204**, and **500 "Internal server error during deletion"** on failure. Whether
the deletion is soft or hard, and whether `audioBlobUrl` is purged, is **not stated**.

> **Reversibility.** Podcast creation is the only expensive write with a real reversal path.
> Kortext publishes **no window** for it. Say "you can delete it" — never "you can delete it
> within N days".
>
> **No idempotency.** Podcast synthesis is the costliest operation in the API and there is no
> request key. A timeout retry generates a second podcast. Reconcile with
> `GET /podcast/v1/podcasts` before retrying.

## 5. Cleaning up

```
POST /tutor/v1/content/{content_id}/delete-index   delete_index_tutor_v1_content__content_id__delete_index_post
```

Removes the index built in step 1. Note it is a POST, not a DELETE. No window is stated for
recovery, and the contract does not say whether wisdom, quizzes or podcasts derived from that index
survive it — do not promise the user either way.

## Errors and limits

422 is the only parseable error (`HTTPValidationError`); 400/403/404/500 carry a description and
no schema. No rate limits are published and none appear in response headers, while every operation
here drives LLM or audio generation. See `errors/kortext-problem-types.yml`,
`rate-limits/kortext-rate-limits.yml` and `conventions/kortext-conventions.yml`.
