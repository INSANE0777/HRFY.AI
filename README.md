# HRFY MCQ Platform Blueprint

## 1. Objective
Transform the Aptitude vertical into a future-proof MCQ engine that powers aptitude, psychometric, technical, and recruiter-specific assessments without relying on AI fallback questions.

## 2. Guiding Principles
- **Single Question Platform**: one canonical question service with schema flexible enough for any MCQ use case.
- **Structured Taxonomy First**: every question tagged across multiple independent dimensions (test type, job profile, exam type, skill, cognitive trait, difficulty band).
- **Deterministic Question Set**: remove on-the-fly LLM generation; rely on curated and pre-approved content pipelines.
- **Observability & Compliance**: every question exposure is audit logged to drive repetition control and analytics.
- **Composable Configurations**: tests defined via templates that bundle constraints, scoring rules, content mix, and delivery behavior.

## 3. High-Level Architecture
| Layer | Responsibilities | Notes |
| --- | --- | --- |
| Content Authoring UI | Recruiters / SMEs upload questions, attach media, manage taxonomy. | Supports draft → review → publish workflow. |
| Question Service | Stores questions, media references, version history, repetition tracking. | Backed by relational DB + object storage for media. |
| Test Template Service | Defines test types, sections, scoring, timing, visibility rules. | Templates can be cloned per recruiter/job. |
| Delivery Orchestrator | Generates concrete test instances based on template constraints, honoring anti-repetition logic. | Stateless service pulling from Question Service. |
| Candidate Test Runner | Web/mobile client rendering multimedia MCQs, timing, proctoring hooks. | Uses signed URLs for media. |
| Reporting & Analytics | Computes raw scores, normalized metrics, topic/skill breakdowns. | Controls explanation visibility per creator config. |

Event bus (e.g., Kafka) carries question usage, completion, and scoring events to analytics warehouse.

## 4. Data Model Highlights
```
Questions
├─ id (UUID)
├─ stem (rich text / HTML / Markdown)
├─ media_assets: [{type, url, alt_text}]
├─ options: [
     { id, text, media_assets[], is_correct, rationale_reference }
  ]
├─ explanation (rich text + media)
├─ metadata:
     difficulty_level (enum)
     language
     time_to_answer_sec (int)
├─ taxonomy_tags: [
     test_type_id,
     job_profile_id,
     exam_id,
     skill_tag_ids[],
     competency_id,
     cognitive_trait_id
  ]
├─ version (int), status {draft,review,published,retired}
├─ last_used_candidates: materialized view from exposures table
```

**Taxonomy tables** (TestTypes, JobProfiles, Exams, SkillTags, Competencies) allow many-to-many tagging via join tables. This supports new verticals without schema changes.

**Question exposures table**
```
question_exposures (
  id UUID,
  question_id,
  candidate_id,
  test_instance_id,
  used_at,
  outcome {unused, viewed, answered},
  result {correct, incorrect, skipped},
  recruiter_org_id
)
```
Drive anti-repetition logic using sliding windows scoped by recruiter/test purpose rather than hardcoded lists.

## 5. Anti-Repetition Strategy
1. Maintain `question_exposures` with indexed lookups on `(candidate_id, question_id)` and `(question_id, recruiter_org_id)`.
2. Allow test templates to define repetition policy:
   - `cooldown_days_per_candidate`
   - `max_exposures_per_candidate`
   - `org_rotation_depth` (e.g., question not reused until X other candidates in same org have seen it)
3. Delivery Orchestrator fetches eligible questions via SQL filters + weighted randomization to ensure diversity across taxonomy dimensions.
4. Provide dashboard signals (question heatmaps) for content managers to refresh overused items.

## 6. Configuration Model
`test_templates` table fields:
- `name`, `test_type_id`, `duration_minutes`, `total_questions`
- `sections`: JSON array describing topic/skill quotas, adaptive rules, and order constraints.
- `scoring_policy`: per option negative marking, partial credit, or psychometric scaling.
- `visibility_rules`: toggles for explanations, topic breakdown, percentile benchmarks.
- `proctoring_settings`: camera/mic requirements, question locks, attempt limits.
- `eligibility_filters`: job profile, experience band, exam cohort, etc.

Templates instantiate `test_instances` with snapshot copies to ensure historical integrity.

## 7. Multimedia Question Support
- Store media files (images, audio, video) in object storage (S3/Azure Blob) with CDN delivery.
- Questions/options reference media via signed URLs and include `alt_text` for accessibility.
- Frontend renders using unified component capable of Markdown + media attachments.
- For technical assessments, allow code snippets with syntax highlighting.

## 8. Content Lifecycle & Governance
1. Author uploads question (draft).
2. SME reviewer approves → status `review`.
3. Content QA publishes to `published` state.
4. Version bump when edits occur; historical versions preserved for audit.
5. Retire questions tied to outdated exam patterns; tests referencing retired items re-hydrate via template rules.

## 9. Migration Path
1. Backfill taxonomy tables using current `question_type` and manual tagging for job/exam dimensions.
2. Migrate existing `last_used_for_candidates` data into `question_exposures`.
3. Build conversion jobs to wrap old tests into template format.
4. Incrementally enable multimedia upload while keeping legacy text-only editor available until parity reached.

## 10. Roadmap Snapshot
- **Phase 1**: Schema migration, template service, exposure tracking, remove LLM fallback.
- **Phase 2**: Multimedia rendering, recruiter custom workflows, proctoring hooks.
- **Phase 3**: Advanced analytics, psychometric scoring libraries, marketplace for shared question banks.

This blueprint sets a foundation for a modular, extensible MCQ platform ready for future verticals and enterprise requirements.

