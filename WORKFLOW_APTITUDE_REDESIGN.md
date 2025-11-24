# HRFY MCQ Platform Workflow

## 1. Content Operations
1. **Intake**  
   - Recruiter or SME defines assessment goals (test type, job profile, skill coverage, difficulty mix).  
   - Create or reuse a `test_template` skeleton in the admin console.
2. **Question Authoring**  
   - Author selects taxonomy tags, uploads stem text, embeds media, defines options, explanation, and metadata (time-to-answer, language, difficulty).  
   - Editor enforces validation (exactly one correct option, accessibility metadata, media size limits).
3. **Review & Publish**  
   - Reviewer checks pedagogy, compliance, branding.  
   - On approval, question transitions to `published` and becomes eligible for template use.  
   - Any edits create a new version; prior test instances keep the historical snapshot.

## 2. Template Management
1. Admin configures template attributes: duration, total questions, sections, scoring policy, anti-repetition thresholds, reporting visibility, and proctoring rules.  
2. Each section defines one or more taxonomy filters (e.g., `skill_tag in {DSA, SQL}` + `difficulty in {Medium, Hard}`).  
3. Templates can be cloned per recruiter organization or job requisition to tweak quotas while sharing base configuration.

## 3. Test Generation
1. Recruiter triggers **Assessment Run** (invite) or candidate self-initiates **Practice Run**.  
2. Delivery Orchestrator requests section-wise question pools from Question Service using template filters and repetition constraints:
   ```
   SELECT q.*
   FROM questions q
   JOIN question_taxonomy qt ON qt.question_id = q.id
   WHERE q.status = 'published'
     AND matches_section_filters(qt, section)
     AND NOT EXISTS (
         SELECT 1 FROM question_exposures qe
         WHERE qe.question_id = q.id
           AND qe.candidate_id = :candidate_id
           AND qe.used_at > NOW() - INTERVAL 'cooldown_days'
     )
   ORDER BY diversity_weight(q, section)
   LIMIT section.question_count;
   ```
3. Selected questions + template snapshot compose a `test_instance` saved with seed metadata for deterministic recreation.

## 4. Test Delivery
1. Candidate authenticates → receives secure test session token.  
2. Runner loads `test_instance` questions via paginated API; media fetched using signed URLs with expiration aligned to session duration.  
3. Timer enforcement handled client-side with server reconciliation; optional auto-submit when time expires.  
4. User responses streamed back in real-time to maintain progress restore capability.

## 5. Anti-Repetition Enforcement During Delivery
- For every question served, Delivery Orchestrator immediately inserts a `question_exposures` record with `outcome='viewed'`.  
- On answer submission, record is updated with `result`.  
- If orchestrator cannot find enough eligible questions, it surfaces a template-level alert so content ops can expand the pool or relax filters—no LLM fallback.

## 6. Scoring & Reporting
1. Upon submission, Scoring Service applies section-specific policies (negative marking, psychometric scaling, adaptive weights).  
2. Computed results stored in `test_results`; breakdown tables capture per-skill / per-topic performance.  
3. Explanation visibility and topic breakdown toggled per template settings before sharing with candidate or recruiter dashboards.  
4. Aggregated telemetry emitted to analytics warehouse for cohort comparisons and content performance metrics.

## 7. Feedback & Continuous Improvement
- Recruiters flag problematic questions directly from reports; flags push items back to draft.  
- Analytics highlight overused or underperforming questions, prompting SMEs to refresh content or adjust filters.  
- Scheduled taxonomy reviews ensure new job profiles/exams/skills are added without schema migrations.

This workflow decouples content lifecycle from delivery, giving HRFY the flexibility needed for future test types and enterprise clients.

