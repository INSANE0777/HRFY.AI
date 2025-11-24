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

## 5. Removing AI Fallback & Improving Question Diversity

### 5.1 Eliminating LLM Question Generation

**Problem**: Current system generates questions on-the-fly via LLM when question bank lacks matching content.

**Solution**: Replace with deterministic, curated content pipeline:

1. **Proactive Content Management**
   - Content ops team builds question pools ahead of demand using taxonomy filters
   - Dashboard shows "pool health" metrics: available questions per taxonomy combination
   - Alerts trigger when pool size drops below threshold (e.g., <50 questions for a skill+difficulty combo)

2. **Fail-Safe Behavior (No LLM)**
   - If Delivery Orchestrator cannot find enough eligible questions:
     - **Assessment tests**: Return error to recruiter with specific pool gaps; test creation blocked until content is added
     - **Practice tests**: Gracefully reduce question count or suggest alternative taxonomy filters
   - System never generates questions automatically; all content must be pre-approved

3. **Content Expansion Workflow**
   - When pool shortage detected → Content ops receives prioritized backlog
   - SMEs author questions in bulk using taxonomy templates
   - Questions go through review → publish pipeline before becoming available
   - Historical tracking ensures we can predict demand patterns and scale pools proactively

### 5.2 Enhanced Anti-Repetition System (Beyond 5-Candidate Limit)

**Current Limitation**: `last_used_for_candidates` stores only 5 candidate IDs, oldest removed when limit reached.

**New Approach**: Time-based sliding windows + configurable policies + multi-dimensional tracking

1. **Question Exposures Audit Table**
   ```
   question_exposures (
     id UUID,
     question_id UUID,
     candidate_id UUID,
     test_instance_id UUID,
     recruiter_org_id UUID,  -- NEW: org-level tracking
     used_at TIMESTAMP,
     outcome {viewed, answered},
     result {correct, incorrect, skipped}
   )
   ```
   - **Indexes**: `(candidate_id, question_id, used_at)`, `(question_id, recruiter_org_id, used_at)`
   - Stores complete history (not just last 5), enabling flexible query patterns

2. **Configurable Repetition Policies** (per test template)
   ```json
   {
     "repetition_policy": {
       "cooldown_days_per_candidate": 90,        // Question not shown to same candidate for 90 days
       "max_exposures_per_candidate": 3,         // Max 3 times ever per candidate
       "org_rotation_depth": 20,                 // Question not reused until 20 other candidates in org have seen it
       "global_cooldown_days": 7,                // Question not reused globally for 7 days (optional)
       "enforce_taxonomy_diversity": true        // Prefer questions from underrepresented taxonomy tags
     }
   }
   ```

3. **Query Logic for Question Selection**
   ```sql
   -- Example: Find eligible questions for a candidate
   WITH candidate_exposures AS (
     SELECT question_id, MAX(used_at) as last_used
     FROM question_exposures
     WHERE candidate_id = :candidate_id
     GROUP BY question_id
   ),
   org_exposures AS (
     SELECT question_id, COUNT(DISTINCT candidate_id) as exposure_count
     FROM question_exposures
     WHERE recruiter_org_id = :org_id
       AND used_at > NOW() - INTERVAL '30 days'
     GROUP BY question_id
   )
   SELECT q.*
   FROM questions q
   JOIN question_taxonomy qt ON qt.question_id = q.id
   LEFT JOIN candidate_exposures ce ON ce.question_id = q.id
   LEFT JOIN org_exposures oe ON oe.question_id = q.id
   WHERE q.status = 'published'
     AND matches_section_filters(qt, :section)
     -- Candidate-level checks
     AND (ce.last_used IS NULL 
          OR ce.last_used < NOW() - INTERVAL ':cooldown_days days'
          OR (SELECT COUNT(*) FROM question_exposures 
              WHERE question_id = q.id AND candidate_id = :candidate_id) < :max_exposures)
     -- Org-level rotation
     AND (oe.exposure_count IS NULL OR oe.exposure_count < :org_rotation_depth)
   ORDER BY 
     diversity_weight(q, :section),  -- Prefer questions from underrepresented taxonomy tags
     RANDOM()  -- Randomize within eligible pool
   LIMIT :question_count;
   ```

4. **Benefits Over Old System**
   - **No hard limit**: Can track unlimited candidate history
   - **Time-based**: Questions become eligible again after cooldown period
   - **Org-aware**: Prevents question leakage across recruiter organizations
   - **Configurable**: Each test type can have different repetition rules
   - **Analytics-driven**: Heatmaps show which questions are overused, enabling proactive content refresh

5. **Monitoring & Alerts**
   - Dashboard shows: "Questions approaching repetition limits" (e.g., 80% of candidates in org have seen this)
   - Automated alerts when pool becomes too constrained by repetition rules
   - Content ops can bulk-retire overused questions and add fresh content

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

