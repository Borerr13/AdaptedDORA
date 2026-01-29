# SecureDORA: A Performance Framework for Security Engineering

## Executive Summary

The DORA (DevOps Research and Assessment) metrics transformed how organizations measure software delivery performance by proving that speed and stability aren't tradeoffs. Elite teams achieve both. Security measurement remains stuck in the pre-DORA era: activity-based vanity metrics that don't correlate with actual security outcomes or organizational performance.

This paper proposes SecureDORA, a framework for measuring security engineering effectiveness using outcome-oriented metrics that parallel DORA's approach. The core thesis: **velocity and security aren't tradeoffs**. Organizations with mature security practices deploy faster, not slower, because they eliminate the friction of manual review, false positive triage, and reactive incident response.

Crucially, SecureDORA introduces the **Security Error Budget**, the enforcement mechanism that makes metrics actionable. Elite teams earn autonomy; struggling teams get automatic guardrails. The metric becomes the gatekeeper, not the security team.

---

## The Problem with Current Security Metrics

### Activity Metrics Don't Measure Outcomes

Most organizations track security through activity-based metrics. These metrics measure **motion**, not **outcomes**. A team can score perfectly on all of them while shipping vulnerable code to production daily.

| Motion Metric | What It Measures | Why It's Meaningless |
|---------------|------------------|----------------------|
| **"Vulnerabilities found"** | Scanner activity | Is finding more good (thorough scanning) or bad (poor code)? The number tells you nothing. |
| **"Days since last pentest"** | Calendar | When you last looked says nothing about what you found or how fast you found it. |
| **"Total CVEs patched"** | Remediation activity | Patching 100 CVEs means nothing if you took 6 months or introduced 200 new ones. |
| **"Compliance audit frequency"** | Process adherence | Passing audits quarterly doesn't mean you're secure. It means you filled out forms. |
| **"Scan coverage percentage"** | Tool deployment | 100% coverage with a noisy scanner just means 100% of your code generates false positives. |

**SecureDORA replaces motion with outcomes:**

| Outcome Metric | What It Actually Tells You |
|----------------|---------------------------|
| **Time to Detect** | How fast do you find vulnerabilities? (Elite = instantly, at commit time) |
| **Time to Remediate** | How fast do you fix them once found? |
| **Remediation Quality** | Do your fixes work, or do they break things? |
| **Introduction Rate** | Are you creating or allowing vulnerabilities into your code in the first place? |

Motion metrics let you look busy. Outcome metrics tell you if you're actually secure.

### The False Tradeoff Assumption

Security is treated as inherent friction. The implicit model:

```
More Security = Slower Delivery
Faster Delivery = Less Security
```

This creates organizational dysfunction:
- Security teams become gatekeepers who slow releases
- Engineering teams view security as an obstacle to route around
- Leadership faces an impossible choice between velocity and safety

DORA proved this tradeoff was false for delivery. Elite performers had both speed and stability. The same is true for security, but we lack the measurement framework to prove it.

---

## The SecureDORA Framework

### Core Metrics

SecureDORA defines four primary outcome metrics based on what matters most in security:

1. **Detecting vulnerabilities** - whether design flaws, code vulnerabilities, or CVEs
2. **Speed in responding** - fixing any vulnerability type quickly
3. **Not causing regressions** - security fixes shouldn't break functionality
4. **Not letting vulnerabilities in** - prevention at the source

| Security Priority | SecureDORA Metric | DORA Equivalent |
|-------------------|-------------------|-----------------|
| Detect vulnerabilities | **Time to Detect** | *(none - security specific)* |
| Speed to fix | **Time to Remediate** | Lead Time + Time to Restore |
| No regressions from fixes | **Remediation Quality** | Change Failure Rate (for fixes) |
| Don't let vulns in | **Introduction Rate** | Change Failure Rate (for features) |

**Why no Deployment Frequency equivalent?**

In DORA, high deployment frequency is good: elite teams ship value multiple times per day. But you don't *want* to ship security fixes multiple times per day. That would mean you're constantly introducing and patching vulnerabilities. The goal is to **not need** frequent security deployments because you're not introducing vulnerabilities in the first place.

---

#### 1. Time to Detect

**What it measures**: How long from when a vulnerability is introduced until it's detected.

**Why this is security-specific**: DORA doesn't need a detection metric because service failures are immediately visible. The site goes down, users complain. Vulnerabilities can hide for years. Security uniquely needs to measure detection capability.

**The elite aspiration**: Time to Detect ≈ 0. The vulnerability is caught at commit time, before it ever merges. It never exists in main. It never reaches production. The developer fixes it in the same context where they wrote it.

This is what verification-based detection at commit time enables.

**Performance tiers**:

| Tier | Time to Detect | What It Means |
|------|----------------|---------------|
| **Elite** | ≈ 0 (commit time) | Caught before merge. Never exists in main. Developer fixes immediately. |
| **High** | < 1 day | Caught in CI/CD post-merge. Brief exposure in main. |
| **Medium** | Days to weeks | Caught in periodic scans. Lives in production. |
| **Low** | Weeks to months | Found by pentests, bug bounties, or incidents. Extended exposure. |

**Calculation**:
```
Time to Detect = Detection Timestamp - Introduction Timestamp
```

For code-level vulnerabilities, Introduction Timestamp is the commit SHA.

**Dependencies require two clocks**:
- **Exposure clock**: When you first deployed the vulnerable version. This is your actual risk exposure period.
- **Remediation clock**: When the CVE was published. This is when you could reasonably have known.

For Time to Detect, use the **remediation clock** (CVE publish date) because that's when detection became possible. For Current Exposure severity weighting, consider the **exposure clock** (total time at risk, even if unknowable).

---

#### 2. Time to Remediate

**What it measures**: How long from when a vulnerability is detected until the fix is deployed to production.

**Why we combine Lead Time and Time to Restore**: Whether it's a routine scan finding or an active incident, a vulnerability is a vulnerability. Fix it fast. The headline metric is unified, but the constraints are different, so we segment by trigger type.

**Trigger types** (for diagnosis and appropriate expectations):

| Trigger Type | Examples | Characteristics |
|--------------|----------|-----------------|
| **Incident-triggered** | Active exploitation, public disclosure, bug bounty, external researcher | External pressure, active risk, tighter SLA |
| **Scan-triggered** | CI detection, periodic scan, commit-time finding | Internal discovery, no external pressure, backlog dynamics |

**Performance tiers by trigger type**:

*Incident-Triggered Remediation* (active risk demands faster response):

| Tier | Time to Remediate | What It Means |
|------|-------------------|---------------|
| **Elite** | < 4 hours | Hotfix deployed same business day |
| **High** | < 24 hours | Fixed within one day |
| **Medium** | 1-3 days | Fixed within a few days |
| **Low** | > 3 days | Unacceptable for active incidents |

*Scan-Triggered Remediation* (internal findings, standard process):

| Tier | Time to Remediate | What It Means |
|------|-------------------|---------------|
| **Elite** | < 1 day | Fixed before next deployment cycle |
| **High** | 1-7 days | Fixed within a week |
| **Medium** | 1-4 weeks | Fixed within a month |
| **Low** | > 1 month | Backlog rot |

**What segmentation reveals**: A team Elite on scan findings but Medium on incidents has good hygiene but poor incident response (on-call understaffed? hotfix pipeline broken?). The reverse, great firefighting but rotting backlog, points to prioritization or ownership problems.

**Decomposition** (for diagnosing bottlenecks within either type):

| Phase | Name | Measures | Constraint Type |
|-------|------|----------|-----------------|
| **Phase 1** | Detection-to-Fix-Started | Time until someone begins working on it | Prioritization problem |
| **Phase 2** | Fix-Started-to-Merged | Time to develop and merge the fix | Engineering throughput |
| **Phase 3** | Merged-to-Deployed | Time from merge to production | Release process |

If Time to Remediate is high, decomposition tells you where to focus.

**Calculation**:
```
Time to Remediate = Production Deployment Timestamp - Detection Timestamp
```

---

#### 3. Remediation Quality

**What it measures**: The percentage of security fixes that cause regressions, break functionality, or introduce new issues.

**Why this matters**: Speed without quality is dangerous. A rushed security fix that breaks the payment system or causes data loss is worse than the vulnerability it patched. Security teams sometimes create pressure to "ship the fix NOW" without adequate testing.

This metric ensures we don't sacrifice system stability for security speed.

**Failure classification** (not all rollbacks are equal):

| Outcome | Counts as Failure? | Rationale |
|---------|-------------------|-----------|
| Rollback due to service outage | **Yes** | The fix broke production |
| Rollback due to functional regression | **Yes** | The fix broke features |
| Rollback due to security ineffectiveness | **Yes** | The fix didn't actually fix it |
| Canary/staged rollout catches problem | **Yes** | Problem that flawed code got that far (but Time to Remediate captures fast recovery) |
| Follow-up for scope expansion or hardening | **No** | Defense-in-depth is good practice, not regression |

**What counts as a remediation failure** (true harms only):
- Security fix causes service outage
- Security fix breaks existing functionality
- Security fix introduces new vulnerabilities
- Security fix is rolled back (including in canary/staged rollout)
- Security fix doesn't actually remediate the vulnerability

**What does NOT count as failure**:
- Follow-up commit that adds defense-in-depth (e.g., fix injection, then add CSP)
- Planned iterative hardening after initial fix
- Scope expansion ("while we're here, let's also fix this related path")

**Performance tiers**:

| Tier | Remediation Quality | What It Means |
|------|---------------------|---------------|
| **Elite** | > 99% success | Virtually no fix-related regressions |
| **High** | 95-99% success | Rare regressions, quickly caught |
| **Medium** | 90-95% success | Occasional regressions |
| **Low** | < 90% success | Frequent fix-related problems |

**Calculation**:
```
Remediation Quality = (Successful Security Fixes / Total Security Fixes) × 100

Where "Successful" = no rollback, no outage, no functional regression, fix actually works

Note: Follow-up hardening commits don't count against quality. Only true harms count.
```

---

#### 4. Introduction Rate

**What it measures**: The percentage of deployments that introduce new vulnerabilities.

**Why this is the prevention metric**: The other three metrics are about finding and fixing problems. Introduction Rate is about not creating them in the first place. This is the ultimate measure of secure development practices.

**What counts as "introducing" a vulnerability**:
- Deployment adds code with a vulnerability pattern (injection, XSS, etc.)
- Deployment adds/upgrades a dependency with known vulnerabilities
- Deployment introduces a configuration vulnerability
- Deployment removes or weakens existing security controls

**Causal breakdown** (for diagnosing root causes):

| Introduction Source | Example | Intervention |
|---------------------|---------|--------------|
| **New code** | Developer writes SQL injection | Training, code review patterns, IDE tooling |
| **Dependency** | Library upgrade introduces CVE | Dependency policy, automated upgrade scanning |
| **Configuration** | IaC exposes port to public | IaC policy checks, config templates |
| **Control removal** | PR deletes input validation | Required reviewers for security-sensitive paths |

When Introduction Rate is high, the breakdown tells you where to invest. "30% from dependencies, 50% from new code, 20% from config" points to very different interventions than "90% from new code."

**Performance tiers**:

| Tier | Introduction Rate | What It Means |
|------|-------------------|---------------|
| **Elite** | < 1% | Virtually no deployments introduce vulnerabilities |
| **High** | 1-5% | Rare introductions, strong prevention |
| **Medium** | 5-15% | Regular introductions, typical for most organizations |
| **Low** | > 15% | Frequent introductions, weak prevention |

**Calculation**:
```
Introduction Rate = (Deployments Introducing Vulnerabilities / Total Deployments) × 100
```

---

### The Security Error Budget

Google's SRE model didn't just give us metrics. It gave us Error Budgets. If you burn your budget, feature work stops. SecureDORA adopts this mechanism.

**The Concept**: Every team has a Security Error Budget based on their tier targets. Exceeding the budget triggers automatic consequences.

**Why It Matters**: The Error Budget replaces "Security Team as Gatekeeper" with "Metric as Gatekeeper."
- Elite teams earn autonomy
- Struggling teams get automatic guardrails
- Engineers fight to *attain* elite status because it means freedom

**Example Error Budget Policy**:

| Metric | Elite Threshold | Budget Exceeded |
|--------|-----------------|-----------------|
| Time to Detect | ≈ 0 (commit time) | > 7 days |
| Time to Remediate | < 1 day (scan-triggered) | > 14 days |
| Remediation Quality | > 99% | < 90% |
| Introduction Rate | < 1% | > 5% |

*Note: Error Budget uses scan-triggered thresholds for Time to Remediate since that's the steady-state measure. Incident response performance is tracked separately for operational readiness but doesn't split the Error Budget.*

**Consequences of Budget Status**:

| Budget Status | Consequences |
|---------------|--------------|
| **Elite (all metrics in Elite range)** | Automatic bypass of manual security review. Self-service deployments. Security team consults, doesn't gate. |
| **Within Budget** | Standard review process. No additional friction. |
| **Budget Exceeded** | Automatic gates activate. No deployments without security sign-off. Remediation becomes top priority. |
| **Severely Exceeded** | Feature work stops. All engineering effort redirected to remediation. |

**The Radical Idea**: Elite status buys you fewer controls. Governance is dynamic and earned, not universal and permanent. This inverts how most enterprises think about security, where controls are applied uniformly regardless of track record.

**The Two-Sided Contract**: If security wants to go faster, it must reduce friction. If engineering wants autonomy, it must reduce escapes. Neither side can win alone. Both have skin in the game.

**The Virtuous Cycle**: Teams want Elite status because it removes friction. The way to get Elite status is to write secure code and fix issues fast. Security improves because engineers are intrinsically motivated, not because they're being policed.

---

### Extended Metrics

Beyond the four core metrics, SecureDORA defines additional measures for deeper insight:

#### Current Exposure

**Definition**: Count and severity of vulnerabilities currently in production, regardless of when introduced.

**Why it matters**: The core metrics are about flow (introducing, detecting, fixing). Current Exposure is about state (what's our risk right now?). A team might have great flow metrics but still have legacy vulnerabilities from before they improved.

**Calculation**:
```
Current Exposure = Count of confirmed vulnerabilities in production

Weighted:
Weighted Exposure = (Critical × 10) + (High × 5) + (Medium × 2) + (Low × 1)
```

#### Security Debt

**Definition**: Current Exposure weighted by age, measuring how long vulnerabilities have been sitting unaddressed.

**Why it matters**: A Critical vulnerability discovered yesterday is concerning. A Critical vulnerability that's been sitting for 18 months is a leadership failure. Security Debt captures compounding risk over time. The longer issues sit, the more likely they are to be exploited, and the more they signal organizational dysfunction.

**Calculation**:
```
Security Debt = Σ(Severity Weight × Age in Days)

Where Age = Today - Detection Date
```

**Why this is powerful for leadership**: Current Exposure is a snapshot. Security Debt is a trend indicator. Rising Security Debt means you're accumulating risk faster than you're retiring it. It's the security equivalent of technical debt, and just as psychologically compelling in executive conversations.

#### Signal-to-Noise Ratio

**Definition**: True positive findings divided by total findings.

**Why it matters**: A scanner producing 1,000 findings with 50 true vulnerabilities wastes 95% of triage effort.

**Target**: Elite organizations achieve >90% signal ratio.

#### Return on Friction (RoF)

**Definition**: Vulnerabilities prevented per unit of security friction invested.

**Calculation**:
```
RoF = Vulnerabilities Prevented Pre-Production / Security Friction Hours
```

**Why it matters**: Exposes inefficient security processes. "We spent 1,000 hours to catch 2 low-risk bugs" vs. "We spent 10 hours to catch 50 critical bugs."

#### Security Friction Index

**Definition**: Engineering hours spent on security activities that don't directly improve the product.

**Components**:
- Time triaging scan results
- Time waiting for security review/approval
- Time researching how to fix findings
- Time re-fixing rejected remediation attempts

**Target**: Elite organizations approach zero friction.

---

### Capabilities (Predictors of Elite Metrics)

DORA identifies capabilities (trunk-based development, automated testing) that *predict* good metrics. SecureDORA similarly identifies security capabilities:

#### Verification Coverage

**Definition**: Percentage of critical sinks with verified guard conditions.

**Note**: This is a *capability*, not a core metric. High Verification Coverage predicts low Introduction Rate and fast Time to Detect, but it's an internal state, not an outcome.

**What "verified" means**: Static analysis confirms the guard executes on all paths from source to sink (dominance in the control flow graph), not merely that it exists somewhere.

#### Shift-Left Ratio

**Definition**: Percentage of vulnerabilities detected before merge vs. after.

**Why it matters**: Pre-merge detection (high Shift-Left Ratio) drives Time to Detect toward zero.

#### Automated Fix Rate

**Definition**: Percentage of detected vulnerabilities that receive auto-generated fix suggestions.

**Why it matters**: Findings with fixes accelerate Time to Remediate.

---

## Limits and Failure Modes

Any measurement framework can be gamed or misapplied. SecureDORA is no exception.

### Goodhart's Law

> "When a measure becomes a target, it ceases to be a good measure."

**Common gaming patterns**:

| Gaming Behavior | Metric Affected | How It Distorts |
|-----------------|-----------------|-----------------|
| **Severity reclassification** | Introduction Rate | Downgrading findings to avoid counting |
| **Delayed detection logging** | Time to Detect | Not recording detection until fix is ready |
| **Rushing fixes without testing** | Remediation Quality | Fast Time to Remediate but broken fixes |
| **Suppressing findings** | All metrics | Marking true positives as false positives |

**Mitigations**:
- Track metrics for insight, not punishment
- Audit categorization decisions periodically
- Use the Error Budget for enforcement, not individual metrics
- Complement with actual outcomes (incidents, breaches, external findings)

### Verification-Based Detection: Capabilities and Constraints

**What verification can confirm**:
- Data flows from identified sources to identified sinks
- Presence of sanitization/validation on all paths (dominance)
- Parameterization of database queries
- Known vulnerable code patterns

**What verification cannot fully confirm** (modeling gaps):

| Gap | Example | Impact |
|-----|---------|--------|
| **Framework behavior** | ORM magic methods, template engines | Sink may not be recognized |
| **Reflection / dynamic dispatch** | `getattr(obj, user_input)()` | Data flow is opaque |
| **Deserialization** | Arbitrary object instantiation | Object graph is runtime-dependent |
| **External defenses** | WAF rules, auth proxies | Defense exists but isn't in analyzed code |

**Honest framing**: Verification-based detection limits false positives to modeling errors, whereas pattern matching produces false positives by design. This is a dramatic improvement, not a guarantee of perfection.

---

## The Security Error Budget in Practice

### Policy Template

```
SECURITY ERROR BUDGET POLICY v1.0

Measurement Period: Rolling 90 days
Review Cadence: Weekly automated, Monthly human review

ELITE STATUS REQUIREMENTS (all must be met):
- Time to Detect: ≈ 0 (commit-time detection)
- Time to Remediate: < 1 day (scan-triggered); incident response tracked separately
- Remediation Quality: > 99%
- Introduction Rate: < 1%

ELITE STATUS BENEFITS:
- No mandatory security review for standard deployments
- Self-service security scanning (results informational, not blocking)
- Security team available for consultation, not approval

BUDGET EXCEEDED TRIGGERS:
- Time to Detect > 7 days → Detection tooling review mandated
- Time to Remediate > 14 days → Remediation sprint required
- Remediation Quality < 90% → Fix process review required
- Introduction Rate > 5% → Manual review for all deployments

APPEALS PROCESS:
- Disputes on finding validity: Security team adjudicates within 48 hours
- Emergency deployments during freeze: VP-level approval required

SUPPRESSION AUDIT:
- All suppressed/false-positive findings logged with justification and approver
- Monthly random audit of 10% of suppressions by security team
- Patterns in suppression reasons trigger rule/tooling review
- Suppression-to-confirmed-vulnerability ratio tracked as health metric
```

### Transition Strategy

**Quarter 1: Measure**
- Instrument all metrics
- Establish baselines
- No enforcement

**Quarter 2: Targets**
- Set achievable targets (Medium tier)
- Begin tracking publicly
- No enforcement yet

**Quarter 3: Soft Enforcement**
- Budget exceeded triggers warnings, not blocks
- Elite benefits activated for qualifying teams

**Quarter 4: Full Enforcement**
- Budget exceeded triggers real consequences
- Error Budget becomes the gatekeeper

---

## The Elite Performance Model

### What Elite Looks Like

In an elite organization:

1. **Time to Detect ≈ 0**. Vulnerabilities caught at commit time, before merge. Developers fix in context.

2. **Time to Remediate < 1 day**. When something slips through, it's fixed within 24 hours.

3. **Remediation Quality > 99%**. Security fixes don't break things. Proper testing, proper review.

4. **Introduction Rate < 1%**. Secure coding is the norm. Vulnerabilities are exceptional, not routine.

5. **No security review gate** because the team has earned trust through metrics.

6. **DORA metrics improve** because security friction approaches zero.

### The Maturity Model

| Level | Characteristics | Profile |
|-------|-----------------|---------|
| **Low** | Periodic scans, reactive fixes, security as blocker | TTD: months, TTR: weeks, Quality: <90%, Intro: >15% |
| **Medium** | CI scanning, regular fixes, some automation | TTD: days, TTR: weeks, Quality: 90-95%, Intro: 5-15% |
| **High** | Continuous scanning, fast fixes, security as enabler | TTD: hours, TTR: days, Quality: 95-99%, Intro: 1-5% |
| **Elite** | Commit-time detection, same-day fixes, earned autonomy | TTD: ≈0, TTR: <1 day, Quality: >99%, Intro: <1% |

---

## Implementing SecureDORA

### Phase 1: Instrument (Weeks 1-4)

1. **Deploy measurement infrastructure**: Track introduction time, detection time, remediation time, fix outcomes.

2. **Establish baselines**: Calculate current metrics across all four dimensions.

3. **Identify constraint**: Which metric is worst? That's your focus.

### Phase 2: Target (Weeks 5-12)

1. **Set achievable goals**: Target Medium tier initially.

2. **Publish dashboards**: Make metrics visible to teams.

3. **Identify quick wins**: Replace noisy scanners, automate obvious fixes.

### Phase 3: Enforce (Weeks 13-24)

1. **Activate Error Budget**: Start with soft enforcement.

2. **Grant Elite benefits**: Teams meeting thresholds get autonomy.

3. **Coach struggling teams**: Focus on their specific constraint.

### Phase 4: Optimize (Ongoing)

1. **Drive Time to Detect toward zero**: Shift detection to commit time.

2. **Automate remediation**: Generate and apply fixes automatically.

3. **Raise the bar**: Tighten thresholds as organization improves.

---

## Organizational Implications

### Security Team Evolution

| Maturity | Security Team Role |
|----------|-------------------|
| Low | Gatekeepers who approve/reject |
| Medium | Reviewers who consult on findings |
| High | Enablers who improve tooling |
| Elite | Architects who design security in; Error Budget administrators |

### Metric Ownership

| Metric | Primary Owner | Supporting Owner |
|--------|---------------|------------------|
| Time to Detect | Security | Tooling/Platform |
| Time to Remediate | Engineering | Security |
| Remediation Quality | Engineering | QA |
| Introduction Rate | Engineering | Security |

### Executive Communication

SecureDORA provides vocabulary for leadership:

- "Time to Detect is now under 1 hour, so we catch vulnerabilities before they merge"
- "Introduction Rate dropped from 12% to 2%, which means we're shipping cleaner code"
- "Three teams earned Elite status and no longer need security gates"
- "Remediation Quality is 98%, meaning our security fixes don't break things"

---

## Conclusion

The security industry has operated without rigorous performance measurement while DevOps transformed through DORA. SecureDORA brings the same discipline to security.

Four metrics capture what matters:
1. **Time to Detect** - Find vulnerabilities fast (ideally at commit time)
2. **Time to Remediate** - Fix them quickly once found
3. **Remediation Quality** - Don't break things with the fix
4. **Introduction Rate** - Don't introduce vulnerabilities in the first place

The Security Error Budget is the enforcement mechanism that makes this real. Elite teams earn autonomy. Struggling teams get guardrails. The metric becomes the gatekeeper, not the security team.

The question for every organization: where are you on the SecureDORA maturity model, and what would it take to reach Elite?

---

## Appendix: Metric Calculation Reference

### Time to Detect
```
Time to Detect = Detection Timestamp - Introduction Timestamp

Introduction Timestamp:
- Code vulnerabilities: Commit SHA where pattern first appears
- Dependencies: CVE publish date (remediation clock - when detection became possible)
- Design flaws: First deployment of affected feature

Note: For dependencies, track "exposure clock" (first vulnerable deployment) separately
for risk assessment, but use "remediation clock" (CVE publish) for Time to Detect
since you can't detect what hasn't been disclosed.
```

### Time to Remediate
```
Time to Remediate = Production Deployment Timestamp - Detection Timestamp

Segment by trigger type (different thresholds):
- Incident-triggered: Elite < 4h, High < 24h, Medium 1-3d, Low > 3d
- Scan-triggered: Elite < 1d, High 1-7d, Medium 1-4w, Low > 1mo

Decomposition:
- Phase 1 (Prioritization) = Work Started - Detection
- Phase 2 (Development) = Fix Merged - Work Started
- Phase 3 (Deployment) = Production Deploy - Fix Merged
```

### Remediation Quality
```
Remediation Quality = (Successful Fixes / Total Fixes) × 100

Successful = No rollback, no outage, no functional regression, fix actually works

Counts as failure: outage, functional regression, security ineffectiveness, canary rollback
Does NOT count: follow-up hardening, scope expansion, iterative defense-in-depth
```

### Introduction Rate
```
Introduction Rate = (Deployments Introducing Vulnerabilities / Total Deployments) × 100

Causal Breakdown (for root cause analysis):
- New Code %: Vulnerabilities from developer-written code
- Dependency %: Vulnerabilities from added/upgraded dependencies
- Configuration %: Vulnerabilities from IaC, config files, environment
- Control Removal %: Vulnerabilities from weakened/deleted defenses
```

### Current Exposure
```
Current Exposure = Count of vulnerabilities in production

Weighted Exposure = (Critical × 10) + (High × 5) + (Medium × 2) + (Low × 1)
```

### Signal-to-Noise Ratio
```
Signal Ratio = True Positives / Total Findings
```

### Return on Friction
```
RoF = Vulnerabilities Prevented / Security Friction Hours
```
