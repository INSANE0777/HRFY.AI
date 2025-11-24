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
2. **Pre-flight Validation**: Orchestrator checks pool availability for each section using template's repetition policy. If insufficient, test creation fails with specific gap report (see Section 5.2).
3. **Question Selection**: Delivery Orchestrator requests section-wise question pools from Question Service:
   - Applies taxonomy filters (skill, difficulty, test type, etc.)
   - Enforces candidate-level repetition: cooldown days + max exposures per candidate
   - Enforces org-level rotation: question not reused until X other candidates in org have seen it
   - Applies diversity weighting: prefer questions from underrepresented taxonomy tags
   - Randomizes within eligible pool
4. Selected questions + template snapshot compose a `test_instance` saved with seed metadata for deterministic recreation.
5. **Immediate Exposure Logging**: As questions are assigned, `question_exposures` records created with `outcome='viewed'` to prevent reuse in same session.

## 4. Test Delivery
1. Candidate authenticates → receives secure test session token.  
2. Runner loads `test_instance` questions via paginated API; media fetched using signed URLs with expiration aligned to session duration.  
3. Timer enforcement handled client-side with server reconciliation; optional auto-submit when time expires.  
4. User responses streamed back in real-time to maintain progress restore capability.

## 5. Anti-Repetition Enforcement During Delivery

### 5.1 Exposure Tracking
- For every question served, Delivery Orchestrator immediately inserts a `question_exposures` record with `outcome='viewed'` and `recruiter_org_id`.  
- On answer submission, record is updated with `outcome='answered'` and `result` (correct/incorrect/skipped).  
- All exposures are permanently logged (no 5-candidate limit); queries use time-based windows and configurable policies.

### 5.2 Handling Pool Shortages (No LLM Fallback)
**When eligible questions are insufficient:**

1. **Pre-flight Check** (before test instance creation):
   - Orchestrator queries available pool size for each section
   - If pool < required questions × 1.5 (safety margin), test creation is **blocked**
   - Recruiter receives error: *"Insufficient questions for 'DSA - Hard' section. Available: 12, Required: 20. Please contact content team or adjust filters."*

2. **During Test Generation** (edge case):
   - If pool shrinks between pre-flight and generation (rare race condition):
     - **Assessment tests**: Fail gracefully, notify recruiter, allow retry after content expansion
     - **Practice tests**: Reduce question count to available pool, show message: *"Showing 15 of 20 requested questions due to availability"*

3. **Content Team Workflow**:
   - Pool shortage alerts → Content ops dashboard
   - Priority queue shows: `[High] DSA + Hard: 12 available, 20 needed`
   - SMEs bulk-author questions matching taxonomy
   - Once published, test creation automatically unblocks

**Key Principle**: System never generates questions. All content must be human-reviewed and published before use.

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

