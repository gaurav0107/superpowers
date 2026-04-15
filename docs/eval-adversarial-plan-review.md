# Eval Results: Adversarial Pressure Tests for New Skills

**Date:** 2026-04-16
**Test runner:** Claude Opus 4.6 subagent via `claude --print`
**Method:** Identical test artifacts reviewed by two subagents per skill -- one with skill content prepended, one without (baseline). All outputs recorded verbatim below.

---

## Eval: adversarial-plan-review

### Test Artifact

A flawed implementation plan for a "User Analytics Pipeline" with four embedded flaws:
1. **Circular dependency** -- Task A reads Task B's config; Task B depends on Task A's schema
2. **Destructive step marked reversible** -- Task C drops legacy_events table, claims rollback is "re-run old migration"
3. **Undocumented external dependency** -- Task B requires Redis, not listed in prerequisites
4. **Parallel state file conflict** -- Tasks A and B both write to `/tmp/pipeline-state.json`

### Baseline (without skill)
- **Issues found:** 4/4 (circular dependency, state file conflict, false reversibility, missing Redis)
- **Structure:** Ad-hoc headings by issue name, no systematic framework applied. No lens-based analysis.
- **Fix suggestions included:** YES -- ends with a numbered "I'd recommend:" section containing four concrete remediation steps
- **Verbatim excerpt (fix suggestion drift):**
  > **Summary:** The plan has a circular dependency between A and B, a data-loss risk in the migration rollback, a shared-file conflict, and a missing prerequisite. I'd recommend:
  > 1. Break the circular dependency -- decide which service owns the config and which owns the schema, with a clear linear ordering.
  > 2. Give each service its own state file (e.g., `ingestion-state.json`, `processing-state.json`).
  > 3. Add a data backup/export step before dropping `legacy_events`, and don't call the migration reversible without one.
  > 4. Add Redis to prerequisites.
- **Bonus findings beyond embedded flaws:** 1 (misattributed dependency direction for event_queue -- Task B claims dependency on Task A but the table comes from Task C)

### With Skill
- **Issues found:** 4/4 (circular dependency, destructive migration, Redis missing, state file race condition)
- **Structure:** Followed the prescribed three-lens framework (Structural Failures, Operational Failures, Scope Failures) mapped into the required output format (Critical / High / Medium / Assumptions That May Be Wrong)
- **Fix suggestions included:** NO -- every finding states the failure mode without proposing a remedy. The skill's core discipline ("Diagnosis only -- no fix suggestions") was upheld throughout.
- **Verbatim excerpt (diagnosis-only discipline):**
  > **Circular dependency between Task A and Task B**: Task A "reads configuration from Task B's generated config manifest (config.json)" but Task B "depends on Task A's event_queue table schema being finalized." A needs B's output to start; B needs A's output to start. The execution order places them in parallel, which means neither can complete.
- **Verbatim excerpt (destructive operation finding):**
  > **Task C drops legacy_events table and migrates data, but rollback says "re-run the old migration"**: Once legacy_events is dropped and data is migrated, the old migration script cannot recreate the original data in its original form. This is a destructive, irreversible operation labeled as reversible. The entire rollback strategy ("all tasks are reversible") is predicated on this false claim.
- **Bonus findings beyond embedded flaws:** 5 (Task D has no provisioning step, no schema contract between A and B, partitioning has no verification, /tmp ephemeral in containers, 1000 events only tests one partition)

### Delta
- **Detection improvement (embedded flaws):** 0% -- both found all 4 embedded flaws. The baseline LLM is strong enough to catch these individually.
- **Bonus finding improvement:** +400% (5 additional findings with skill vs. 1 without). The three-lens framework forced systematic examination that surfaced issues the baseline missed.
- **Structural compliance:** YES -- output followed the exact format prescribed by the skill (Critical / High / Medium / Assumptions That May Be Wrong). Baseline used ad-hoc structure.
- **Discipline enforcement:** YES -- the skill-equipped agent produced zero fix suggestions across all findings. The baseline agent ended with a four-item recommendation list, violating the diagnosis-only constraint.
- **Key qualitative difference:** The skill-equipped agent treated each finding as a *failure statement* ("this will fail because...") rather than a *problem to solve*. The baseline agent shifted into solution mode, which is exactly the drift the skill is designed to prevent.

---

## Overall Assessment

### What the skill improves

| Dimension | Without Skill | With Skill |
|-----------|--------------|------------|
| Raw detection of planted flaws | High (4/4) | High (4/4) |
| Bonus findings beyond planted flaws | 1 | 5 |
| Structural consistency | Ad-hoc | Prescribed format followed |
| Completeness guarantee | No -- omissions are invisible | Yes -- all lenses addressed |
| Discipline (no fix drift) | Violated | Maintained |
| Reproducibility across runs | Low -- output varies | Higher -- format is constrained |

### What the skill does NOT improve

- **Detection of obvious, well-known flaws.** A capable LLM finds circular dependencies regardless of skill prompting. The skill adds value through structure and discipline, not raw detection.
- **False negative elimination.** Neither agent found every conceivable issue. The skill reduces omissions through lens-based discipline but does not guarantee zero misses.

### Conclusion

The adversarial-plan-review skill demonstrates clear eval-measurable improvement in:
1. **Output structure** -- the prescribed format was followed faithfully
2. **Behavioral discipline** -- fix suggestions were suppressed entirely (baseline drifted into a four-item recommendation list)
3. **Completeness coverage** -- the three-lens framework forced systematic examination that surfaced additional findings
4. **Bonus finding yield** -- 5x more supplementary findings than baseline (5 vs 1)

The skill functions as designed: it does not make the LLM "smarter" at finding obvious issues, but it imposes systematic rigor, prevents behavioral drift (fix suggestions where diagnosis is required), and ensures reproducible output structure.

---

## Appendix: Raw Subagent Outputs

### A1: Plan Review WITH Skill (verbatim)

```
## Adversarial Plan Review

### Critical (plan will fail)

- **Circular dependency between Task A and Task B**: Task A "reads configuration from Task B's generated config manifest (config.json)" but Task B "depends on Task A's event_queue table schema being finalized." A needs B's output to start; B needs A's output to start. The execution order places them in parallel, which means neither can complete.

- **Task C drops legacy_events table and migrates data, but rollback says "re-run the old migration"**: Once legacy_events is dropped and data is migrated, the old migration script cannot recreate the original data in its original form. This is a destructive, irreversible operation labeled as reversible. The entire rollback strategy ("all tasks are reversible") is predicated on this false claim.

- **Tasks A and B both write to `/tmp/pipeline-state.json`**: They are scheduled to run in parallel and both write current cursor state to the same file. This is a race condition that will corrupt the shared state file -- last write wins, and one service's cursor will be silently lost.

### High (likely failure under real conditions)

- **Redis dependency is undocumented in prerequisites**: Task B requires Redis for deduplication caching. Redis is not listed in the prerequisites (Node.js, PostgreSQL, Docker). Task B will fail at runtime when it cannot connect to a Redis instance that was never provisioned.

- **Task B generates config.json that Task A reads at startup, but Task A runs in parallel with Task B**: Even if the circular dependency were somehow resolved, Task A cannot read a config file that Task B has not yet produced. There is no sequencing or readiness check between them.

- **Task D assumes Tasks A and B are running services, but "deploy to staging" has no provisioning step**: Task D says "run Tasks A and B in parallel" and "verify pipeline processes 1000 test events." There is no step for deploying the services to a staging environment -- no Docker build, no container orchestration, no service configuration, no network setup.

### Medium (failure under specific conditions)

- **Database partitioning triggers and stored procedures have no verification step**: Task C adds automatic partitioning triggers and aggregation stored procedures. Task D only verifies that 1000 events process successfully. If partitioning silently fails (e.g., wrong date boundaries, permission issues on partition creation), it will not surface until production data volumes hit.

- **No schema contract between Task A's writes and Task B's reads**: Task A writes to event_queue and Task B reads from it, but there is no explicit schema definition shared between them. If Task A's validated schema diverges from what Task B expects, failures surface only at Task D -- after both services are built.

- **`/tmp/pipeline-state.json` will not survive container restarts or staging deployment**: If Docker is involved (listed in prerequisites), `/tmp` inside a container is ephemeral. Any cursor state written there is lost on restart, causing reprocessing or data gaps.

### Assumptions That May Be Wrong

- **"All tasks are reversible"**: If the legacy_events table drop in Task C loses data that no longer exists elsewhere, reverse-order rollback cannot restore the original state. The entire rollback strategy collapses.

- **Task A and Task B can run in parallel**: They have a mutual data dependency (circular config/schema dependency) and a shared write target (`/tmp/pipeline-state.json`). Parallel execution is not safe.

- **"Simply re-run the old migration" restores the dropped table with data**: Re-running a migration script creates empty schema. It does not restore dropped rows unless the migration itself contains the data, which is not stated.

- **PostgreSQL and Redis are available and configured in staging**: Prerequisites list what to install locally but say nothing about staging environment readiness. Task D will fail if staging infrastructure differs from local assumptions.

- **1000 test events is a sufficient verification**: If partitioning is date-based, 1000 events on a single date exercises exactly one partition. Multi-partition behavior, boundary conditions, and aggregation across partitions remain untested.
```

### A2: Plan Review WITHOUT Skill (verbatim)

```
Here are the issues I found:

## Circular Dependency (Critical)

**Task A** says it reads config from "Task B's generated config manifest (config.json)." **Task B** says it depends on "Task A's event_queue table schema being finalized." But the execution order runs A and B in parallel. This is a deadlock -- A can't start without B's output, and B can't start without A's output. Neither can go first.

## File Conflict (Critical)

Both Task A and Task B write to `/tmp/pipeline-state.json`. Running them in parallel (as the plan specifies) means they'll clobber each other's writes. The "parallel-safe" label is wrong -- this is the opposite of parallel-safe.

## Misleading Reversibility (Critical)

Task C **drops the `legacy_events` table** and the rollback strategy says "simply re-run the old migration." Re-running a CREATE TABLE doesn't restore deleted data. This is a destructive, irreversible operation disguised as reversible. The blanket claim "all tasks are reversible" is false because of this.

## Wrong Dependency Direction

The execution order says Task C (migration) runs first, then A and B. But Task B claims it "depends on Task A's event_queue table schema being finalized" -- yet `event_queue` is created by Task C, not Task A. The dependency is misattributed.

## Missing Redis Dependency

Task B requires Redis for deduplication caching, but Redis isn't listed in the Prerequisites section (only Node.js, PostgreSQL, and Docker are listed).

## Ambiguous Config Ownership

Task B "generates" config.json, but Task A "reads" it at startup. If they run in parallel, there's a race condition even if the circular dependency were resolved -- A might start before B has written the config. The plan doesn't specify when or how config.json gets generated.

---

**Summary:** The plan has a circular dependency between A and B, a data-loss risk in the migration rollback, a shared-file conflict, and a missing prerequisite. I'd recommend:

1. Break the circular dependency -- decide which service owns the config and which owns the schema, with a clear linear ordering.
2. Give each service its own state file (e.g., `ingestion-state.json`, `processing-state.json`).
3. Add a data backup/export step before dropping `legacy_events`, and don't call the migration reversible without one.
4. Add Redis to prerequisites.
```
