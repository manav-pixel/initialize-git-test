# Chat Assistant: Workbench vs Dashboard - Free-Form Questions

## Overview

The Chat Assistant handles free-form questions differently depending on whether it's invoked from the **Model Validation Workbench** (active validation workflow) or the **Dynamic Dashboard** (completed validation review). This document details those differences.

---

## 1. Stage Context

### Workbench: Stage-Specific

The Workbench has a **specific workflow stage** that determines how questions are answered.

```typescript
// Stage is SPECIFIC to workflow step
stage = "DATA_INTAKE_QA"  // User is running QA checks

User asks: "Is data quality adequate?"
  ↓
Orchestrator knows: stage = DATA_INTAKE_QA
  ↓
Routes to: DATA_QUALITY category
  ↓
Loads artifacts: [QAReport, ValidationScope, DataQualityMetrics]
  ↓
Response is QA-focused: "Based on QA results, data quality is..."
```

**Stage Mapping** (Workbench steps):
```
Step 1 (Create Data)           → undefined (generic)
Step 2 (QA)                    → DATA_INTAKE_QA
Step 3 (Select Metrics)        → undefined (generic)
Step 4 (Run Validation)        → RUN_VALIDATION
Step 5 (Review Results)        → REVIEW_RESULTS
Step 6 (Risk Assessment)       → RISK_QUANTIFICATION
Step 7 (Qualitative)           → FINDING_MANAGEMENT
Step 8 (Quantitative)          → FINDING_MANAGEMENT
Step 9 (Executive Summary)     → PUBLISH_SUMMARY
```

### Dashboard: Undefined Stage

The Dashboard shows a **completed, published validation** with no specific workflow stage.

```typescript
// Stage is UNDEFINED (post-hoc review)
stage = undefined  // Completed validation

User asks: "Is data quality adequate?"
  ↓
Orchestrator knows: NO stage context
  ↓
Must infer from question alone (embeddings)
  ↓
Loads ALL available artifacts
  ↓
Response is more historical: "Looking at the completed validation..."
```

---

## 2. Artifact Availability Strategy

### Workbench: On-Demand Loading

Artifacts are loaded **on-demand based on the current stage**.

```
On-Demand Loading Flow:
┌─────────────────────────────────────────────────────┐
│ User asks free-form question                        │
├─────────────────────────────────────────────────────┤
│ Determine current stage: DATA_INTAKE_QA             │
├─────────────────────────────────────────────────────┤
│ Stage-specific artifacts to load:                   │
│ ├─ QAReport (primary)                              │
│ ├─ ValidationScope (context)                       │
│ ├─ DataQualityMetrics (reference)                  │
│ └─ ValidationConfig (reference)                    │
├─────────────────────────────────────────────────────┤
│ Load only these artifacts (~1-2 seconds)            │
├─────────────────────────────────────────────────────┤
│ Answer question with stage-relevant context        │
└─────────────────────────────────────────────────────┘
```

**Advantages**:
- ✓ Fast artifact loading (fewer artifacts)
- ✓ Focused, relevant responses
- ✓ Limited to what's available at this stage
- ✓ Natural workflow guidance

**Disadvantages**:
- ✗ Can't access artifacts from later steps
- ✗ Incomplete context if step not completed
- ✗ May not have all data for comprehensive answer

### Dashboard: Pre-Loaded Strategy

**All artifacts are pre-loaded** when the dashboard initializes.

```
Pre-Loaded Strategy:
┌──────────────────────────────────────────────────────┐
│ Dashboard loads validation                          │
├──────────────────────────────────────────────────────┤
│ Immediately preload ALL artifacts:                  │
│ ├─ ValidationScope                                  │
│ ├─ ValidationMetrics                                │
│ ├─ PerformanceMetricsTrend                          │
│ ├─ TierClassification                               │
│ ├─ RiskAssessment                                   │
│ ├─ QAReport                                         │
│ ├─ HistoricalComparison                             │
│ ├─ PortfolioTrends                                  │
│ ├─ CharacteristicsData                              │
│ ├─ ModelValidationRAG                               │
│ └─ ... (15+ artifacts total)                        │
├──────────────────────────────────────────────────────┤
│ All data cached and ready (~3-5 seconds initially)  │
├──────────────────────────────────────────────────────┤
│ User asks question anytime                          │
│   → Instant artifact resolution                     │
│   → Complete context available                      │
└──────────────────────────────────────────────────────┘
```

**Advantages**:
- ✓ Complete context always available
- ✓ Instant artifact access (no loading delay)
- ✓ Can reference any aspect of validation
- ✓ Comprehensive, informed responses

**Disadvantages**:
- ✗ Higher initial load time (all artifacts)
- ✗ Uses more memory/bandwidth
- ✗ Artifacts may be outdated if validation runs again

---

## 3. Orchestrator Classification Behavior

### Workbench: Stage-Biased Classification

The Orchestrator **uses stage information** to guide classification.

```
Classification Decision Tree:
┌────────────────────────────────────────────────────┐
│ Question: "What does this validation tell us?"    │
│ Stage: DATA_INTAKE_QA                              │
├────────────────────────────────────────────────────┤
│ Embeddings Classification:                         │
│ - Compute embedding                                │
│ - Search similarity: 0.75 (MEDIUM)                 │
│ - Top candidates: [DATA_QUALITY, GENERIC]          │
├────────────────────────────────────────────────────┤
│ Confidence check: 0.75 < 0.85?                    │
│ YES → Need LLM confirmation                        │
├────────────────────────────────────────────────────┤
│ Call LLM with context:                             │
│ "Stage is DATA_INTAKE_QA. User asks: ..."          │
│ "Route to most relevant category."                 │
├────────────────────────────────────────────────────┤
│ LLM Response:                                      │
│ "Category: DATA_QUALITY (focused on QA stage)"     │
│ "Confidence: 0.89"                                 │
├────────────────────────────────────────────────────┤
│ Result: Route to DataQualityAgent                  │
│ → Response emphasizes QA aspects                   │
└────────────────────────────────────────────────────┘
```

**Classification Output**:
```typescript
{
  category: "DATA_QUALITY",
  confidence: 0.89,
  stage_context: "DATA_INTAKE_QA",
  reasoning: "During QA phase, question is about data validation"
}
```

### Dashboard: Pure Embeddings/LLM Classification

The Orchestrator **has no stage information** and relies purely on embeddings and LLM.

```
Classification Decision Tree:
┌────────────────────────────────────────────────────┐
│ Question: "What does this validation tell us?"    │
│ Stage: undefined (completed validation)            │
├────────────────────────────────────────────────────┤
│ Embeddings Classification:                         │
│ - Compute embedding                                │
│ - Search similarity: 0.75 (MEDIUM)                 │
│ - Top candidates: [GENERIC, PERFORMANCE, QUALITATIVE]│
├────────────────────────────────────────────────────┤
│ Confidence check: 0.75 < 0.85?                    │
│ YES → Need LLM confirmation                        │
├────────────────────────────────────────────────────┤
│ Call LLM with minimal context:                     │
│ "User asks: ..."                                   │
│ "Validation is FINALIZED. Route category."         │
├────────────────────────────────────────────────────┤
│ LLM Response:                                      │
│ "Category: GENERIC (broad summary question)"       │
│ "Confidence: 0.82"                                 │
├────────────────────────────────────────────────────┤
│ Result: Route to GenericAgent                      │
│ → Response comprehensive, all-encompassing         │
└────────────────────────────────────────────────────┘
```

**Classification Output**:
```typescript
{
  category: "GENERIC",
  confidence: 0.82,
  stage_context: undefined,
  reasoning: "Question requires comprehensive validation summary"
}
```

---

## 4. Graceful Degradation Paths

### Workbench: Acknowledges Workflow Progress

When artifacts are missing, degradation acknowledges the **workflow stage**.

```
Scenario: User asks "Show me the risk tier classification"
         While still at Step 2 (QA step)

  ↓
Load artifacts for RISK_QUANTIFICATION:
  ├─ TierClassification ✗ (not available yet)
  ├─ ValidationMetrics ✓
  └─ RiskAssessment ✗ (not available yet)

  ↓
Critical artifact missing (TierClassification)

  ↓
Look for stage-specific fallback:
  - Can we answer with DATA_INTAKE_QA artifacts?
  - No alternative playbook matches

  ↓
Use Policy-Based Fallback (POLICY_INTERPRETATION):
  "The risk tier classification is determined during the
   Risk Assessment step (Step 6). To proceed:

   1. Complete QA checks (current step)
   2. Run validation (Step 4)
   3. Review results (Step 5)
   4. Then conduct risk assessment

   For now, policy requires [standard risk requirements]."

Result: User understands what's needed to progress
```

### Dashboard: Historical/Comparative Fallback

When artifacts are missing (rare, since all pre-loaded), degradation uses **historical context**.

```
Scenario: User asks "Show me the risk tier classification"
         Some critical data missing from this validation

  ↓
Load artifacts:
  ├─ TierClassification ✗ (failed to compute)
  ├─ HistoricalComparison ✓
  └─ ValidationMetrics ✓

  ↓
Critical artifact missing but have historical data

  ↓
Use Comparative Fallback:
  "This validation's risk tier could not be computed.
   However, comparing to prior validations:

   - Prior 3 validations: Tier 2 (Moderate)
   - Recent trend: Metrics improving
   - Policy recommends: Tier 1-2 based on metrics

   [Provides best-effort guidance from similar cases]"

Result: User gets useful analysis even with missing data
```

---

## 5. Question Type Distribution

### Workbench: Diverse Question Types

Users ask a variety of question types, with **stage-specific dominance**.

```
Question Distribution in Workbench:

┌─ Stage-Specific Contextual Questions ─────────── 60% ┐
│                                                       │
│ User is at Step 2 (QA):                              │
│   "Is data quality adequate?"                        │
│   "What failed QA checks?"                           │
│   "Should we proceed to validation?"                 │
│                                                       │
│ User is at Step 5 (Review Results):                  │
│   "What are the key findings?"                       │
│   "Is the model performing as expected?"             │
│   "What metrics concern you most?"                   │
│                                                       │
└───────────────────────────────────────────────────────┘

┌─ How-To & Policy Questions ────────────────────── 25% ┐
│                                                       │
│   "What does Gini coefficient mean?"                 │
│   "How do I document findings?"                      │
│   "What's required by ECB regulations?"              │
│   "How should I interpret PSI?"                      │
│                                                       │
└───────────────────────────────────────────────────────┘

┌─ Workflow Navigation Questions ────────────────── 15% ┐
│                                                       │
│   "What's the next step?"                            │
│   "Do I need to flag this issue?"                    │
│   "Can I skip to executive summary?"                 │
│   "Is this validation complete?"                     │
│                                                       │
└───────────────────────────────────────────────────────┘

✓ Free-form questions are EXPECTED and ENCOURAGED
✓ Stage context makes responses highly relevant
✓ Users leverage assistant to progress workflow
```

### Dashboard: Cue-Dominated with Some Free-Form

Most interactions are **cue-driven**, with free-form as secondary.

```
Question Distribution in Dashboard:

┌─ Contextual Cue-Based Questions ───────────────── 70% ┐
│                                                        │
│ User hovers over chart → Cue bubble appears:           │
│   Click: "Why is Gini dropping?"                      │
│   → Shows trend analysis                              │
│                                                        │
│ User clicks metric → Contextual question:             │
│   "Is this risk level acceptable?"                    │
│   → Compares to thresholds & prior validations       │
│                                                        │
│ User sees finding → Related cue:                      │
│   "What should we do about this finding?"             │
│   → Links to remediation guidance                     │
│                                                        │
└────────────────────────────────────────────────────────┘

┌─ Free-Form Validation Questions ────────────────── 20% ┐
│                                                        │
│   "Summarize the key findings"                        │
│   "Generate an audit checklist"                       │
│   "What changed since last validation?"               │
│   "Is this model fit for production?"                 │
│                                                        │
└────────────────────────────────────────────────────────┘

┌─ Policy/Regulatory Questions ──────────────────── 10% ┐
│                                                        │
│   "What does ECB require for this?"                   │
│   "Is this a material model change?"                  │
│   "What's our policy on degraded models?"             │
│                                                        │
└────────────────────────────────────────────────────────┘

⚠ Free-form questions are SECONDARY (cues are primary)
⚠ Most interactions initiated by suggested cues
⚠ Free-form used for deeper analysis
```

---

## 6. Real Example: Same Question, Different Context

### Question: "Generate an audit checklist for this model"

```
═══════════════════════════════════════════════════════════════════

                           WORKBENCH
                        (Step 8 - Quantitative)

Stage: FINDING_MANAGEMENT
Workflow State: Actively validating model

  ┌─ Classification ─────────────────────────────────────┐
  │ Question: "Generate an audit checklist for this model"│
  │                                                       │
  │ Embeddings: similarity = 0.89 (HIGH)                │
  │ Predicted category: AUDIT_REQUIREMENTS              │
  │ Skip LLM (confidence high)                           │
  │ Confidence: 0.92                                     │
  └─────────────────────────────────────────────────────┘

  ┌─ Artifact Loading ───────────────────────────────────┐
  │ Load stage-specific artifacts:                       │
  │ ├─ TierClassification ✓ (Tier-2)                    │
  │ ├─ ValidationMetrics ✓ (4 key metrics)             │
  │ ├─ RiskAssessment ✓ (3 findings)                    │
  │ ├─ MRMPolicy ✓ (audit requirements)                │
  │ ├─ PriorFindings ✓ (1 prior issue)                 │
  │ └─ ExecutiveSummary ✗ (not generated yet)          │
  │                                                       │
  │ Total load time: ~1.2 seconds                        │
  └─────────────────────────────────────────────────────┘

  ┌─ Response Generation ────────────────────────────────┐
  │ Agent: FindingManagementAgent                        │
  │ System Prompt: "You are a risk assessment expert..."│
  │                                                       │
  │ Generated Checklist:                                 │
  │                                                       │
  │ AUDIT CHECKLIST - Credit Card PD Model v1.2         │
  │ Tier: 2 (Moderate Risk)                             │
  │                                                       │
  │ DATA VALIDATION:                                     │
  │ ☐ QA checks completed: PASS                         │
  │   (9/9 checks passed per QA report)                 │
  │ ☐ File integrity: PASS                              │
  │ ☐ Sample size adequate (50K+ records)              │
  │                                                       │
  │ PERFORMANCE VALIDATION:                             │
  │ ☐ Gini coefficient ≥ 0.30: PASS (0.42)            │
  │ ☐ Stability metrics: PASS                           │
  │ ☐ Back-testing: PASS (3/3 periods)                 │
  │                                                       │
  │ GOVERNANCE:                                         │
  │ ☐ MRM Policy compliance (Policy Section 4.2)       │
  │ ☐ Regulatory requirements (Tier-2 scope)           │
  │ ☐ Review by Risk Committee: PENDING                │
  │ ☐ Prior finding addressed: YES (from Q3 validation) │
  │                                                       │
  │ BASIS: MODEL_DATA                                    │
  │ SOURCES: [QAReport, ValidationMetrics, MRMPolicy]  │
  └─────────────────────────────────────────────────────┘

Result: STEP-FOCUSED, ACTIONABLE CHECKLIST
        - Focuses on immediate validation audit needs
        - References current findings (1 prior issue resolved)
        - Aligned with Tier-2 requirements
        - User can act on items immediately

═══════════════════════════════════════════════════════════════════

                           DASHBOARD
                    (Completed Validation Review)

Stage: undefined
Workflow State: Published validation (completed)

  ┌─ Classification ─────────────────────────────────────┐
  │ Question: "Generate an audit checklist for this model"│
  │                                                       │
  │ Embeddings: similarity = 0.71 (MEDIUM)              │
  │ Need LLM confirmation                               │
  │ Call LLM with question + validation context         │
  │                                                       │
  │ LLM Response:                                        │
  │   Category: AUDIT_REQUIREMENTS                      │
  │   Confidence: 0.89                                   │
  │ Skip refinement                                      │
  └─────────────────────────────────────────────────────┘

  ┌─ Artifact Loading ───────────────────────────────────┐
  │ All artifacts pre-loaded at dashboard init:          │
  │ ├─ ValidationScope ✓                                │
  │ ├─ ValidationMetrics ✓                              │
  │ ├─ PerformanceMetricsTrend ✓ (4 validations)      │
  │ ├─ TierClassification ✓ (Tier-2)                   │
  │ ├─ RiskAssessment ✓                                │
  │ ├─ HistoricalComparison ✓ (4 prior validations)   │
  │ ├─ MRMPolicy ✓                                     │
  │ ├─ ExecutiveSummary ✓                              │
  │ ├─ KeyModelFindings ✓                              │
  │ └─ [15+ more artifacts...]                         │
  │                                                       │
  │ Total: All available instantly                       │
  └─────────────────────────────────────────────────────┘

  ┌─ Response Generation ────────────────────────────────┐
  │ Agent: FindingManagementAgent                        │
  │ System Prompt: "You are a model governance expert..."│
  │                                                       │
  │ Generated Checklist:                                 │
  │                                                       │
  │ AUDIT CHECKLIST - Credit Card PD Model v1.2         │
  │ Tier: 2 (Moderate Risk) | Status: FINALIZED         │
  │                                                       │
  │ HISTORICAL PERFORMANCE:                             │
  │ ☐ Consistency across validations: PASS             │
  │   - Gini trend: 0.40 → 0.42 (stable)               │
  │   - PSI trend: 0.08 → 0.12 (drifting, monitor)    │
  │ ☐ Prior findings: 3 resolved, 0 outstanding         │
  │                                                       │
  │ CURRENT VALIDATION (Feb 2026):                      │
  │ ☐ Data quality: EXCELLENT (9/9 QA checks pass)    │
  │ ☐ Model performance: GOOD (Gini 0.42, PSI 0.12)   │
  │ ☐ Risk tier: APPROPRIATE (Tier-2)                  │
  │                                                       │
  │ REGULATORY COMPLIANCE:                              │
  │ ☐ ECB validation frequency: Met (annual)            │
  │ ☐ OCC reporting: Current                            │
  │ ☐ RBI requirements: Compliant                       │
  │ ☐ Material change assessment: No (drift acceptable) │
  │                                                       │
  │ GOVERNANCE & SIGN-OFF:                              │
  │ ☐ Model owner review: APPROVED                      │
  │ ☐ Risk committee sign-off: APPROVED                 │
  │ ☐ Compliance verification: PASSED                   │
  │ ☐ Documentation complete: YES                       │
  │                                                       │
  │ BASIS: MIXED (MODEL_DATA + REGULATORY)              │
  │ SOURCES: [All validation artifacts, Policy docs]    │
  └─────────────────────────────────────────────────────┘

Result: COMPREHENSIVE, HISTORICAL CHECKLIST
        - Includes trend analysis (4 validations)
        - Shows regulatory alignment
        - Demonstrates compliance & approvals
        - Audit-ready documentation
        - Can be included in compliance report

═══════════════════════════════════════════════════════════════════

COMPARISON:

Aspect                  Workbench           Dashboard
─────────────────────────────────────────────────────────
Focus                   Immediate audit     Comprehensive review
Time Scope              Current validation  Trend across validations
Sources Referenced      Current only        Historical + current
Governance Status       In progress         Completed/Approved
Regulatory Details      Tier-specific       Full compliance matrix
Use Case                Planning QA review  Audit documentation
Latency                 1.2s (fast)         Instant (pre-loaded)
Context                 Step-focused        Holistic
```

---

## 7. Code-Level Differences

### Workbench Implementation

```typescript
// src/components-2.0/model-validation-workbench.tsx

<ChatAssistant
  user={user || { id: 'unknown', roles: [userRole || 'analyst'] }}
  validationId={(completedValidationId || validationId) ?? undefined}
  stage={dashboardStage}              // ← SPECIFIC STAGE
  onClose={() => setShowCopilotPanel(false)}
  qaStatus={qaStatus}                 // ← QA tracking
  onQACompleted={() => handleQACompleted()}  // ← Cache invalidation
/>

// Stage determination:
const dashboardStage = (() => {
  const stageMap = {
    2: AureonStage.DATA_INTAKE_QA,
    4: AureonStage.RUN_VALIDATION,
    5: AureonStage.REVIEW_RESULTS,
    6: AureonStage.RISK_QUANTIFICATION,
    7: AureonStage.FINDING_MANAGEMENT,
    8: AureonStage.FINDING_MANAGEMENT,
    9: AureonStage.PUBLISH_SUMMARY,
  }
  return stageMap[viewingStep]
})()
```

### Dashboard Implementation

```typescript
// src/components/dynamic-dashboard/dynamic-dashboard.tsx

// ChatAssistant NOT directly embedded in main render
// Instead, opened via button/cue interaction

// When user opens chat:
<ChatAssistant
  user={user}
  validationId={validationId}
  stage={undefined}                   // ← NO STAGE
  onClose={() => setShowChat(false)}
/>

// Pre-load all artifacts immediately:
useEffect(() => {
  preloadArtifacts(validationId, stage=null)  // null = load all
}, [validationId])
```

---

## 8. Comparison Summary Table

| Aspect | Workbench | Dashboard |
|--------|-----------|-----------|
| **Stage Context** | Specific (DATA_INTAKE_QA, etc.) | Undefined |
| **Artifact Strategy** | On-demand per stage | All pre-loaded |
| **Loading Latency** | 1-2 seconds per question | Instant |
| **Initialization Cost** | Low (only current stage) | High (all artifacts) |
| **Free-Form Frequency** | 60% of interactions | 20% of interactions |
| **Primary Interaction** | Free-form questions | Contextual cues |
| **Question Type** | Progressive, step-focused | Retrospective, comprehensive |
| **Degradation Path** | Workflow-aware | Historical-aware |
| **Response Scope** | Current step + policies | All validation data + trends |
| **Classification** | Stage-biased | Pure embeddings/LLM |
| **Use Case** | Active validation workflow | Completed validation review |
| **Regulatory Focus** | Tier-specific requirements | Full compliance matrix |
| **Historical Context** | Limited (current validation) | Full (all validations) |
| **User Goal** | "How do I proceed?" | "What did we learn?" |

---

## 9. When to Use Each

### Use Workbench Chat When:

✓ User is actively running validation steps
✓ Need step-specific guidance (QA-focused, validation-focused, etc.)
✓ Want progressive, scaffolded assistance
✓ Limited context available (steps not yet complete)
✓ Asking "What should I do now?" or "Can I proceed?"
✓ Seeking workflow navigation help

**Example Questions**:
- "Is this data quality adequate for validation?"
- "What do these metrics tell us?"
- "Should we flag this as a finding?"
- "What's the next step?"

### Use Dashboard Chat When:

✓ Reviewing a completed, published validation
✓ Need comprehensive analysis across full validation
✓ Want historical/trend perspective
✓ All validation data available for analysis
✓ Asking "What did we learn?" or "Is this compliant?"
✓ Seeking documentation/audit checklist generation

**Example Questions**:
- "Generate an audit checklist"
- "Summarize key findings"
- "Compare this to prior validations"
- "What changed since last quarter?"

---

## 10. Technical Impact

### Performance Implications

```
Workbench (On-Demand):
├─ Initial load: <500ms
├─ Per question: 1-2s (artifact loading)
├─ Memory usage: Low (~10-50MB)
├─ Total session: ~30s (6 questions)

Dashboard (Pre-Loaded):
├─ Initial load: 3-5s (all artifacts)
├─ Per question: Instant (<500ms)
├─ Memory usage: High (~100-200MB)
├─ Total session: ~35s (10 questions) - but each is instant
```

### Bandwidth Implications

```
Workbench:
├─ Initial: Minimal
├─ Per question: 50-200KB artifacts
├─ Total: 300KB - 1.2MB (6 questions)

Dashboard:
├─ Initial: 1-2MB (all artifacts at once)
├─ Per question: 0KB (cached)
├─ Total: 1-2MB (regardless of questions)
```

---

## 11. Error Handling Differences

### Workbench Error Scenarios

```
Missing Artifact (common):
→ "We need to complete [step X] first"
→ Helpful workflow guidance
→ Suggests next action

Example:
User at Step 2 asks about risk tier
→ "Risk tier is determined in Step 6 (Risk Assessment).
   First complete: Step 4 (Run Validation)
                   Step 5 (Review Results)"
```

### Dashboard Error Scenarios

```
Missing Artifact (rare, since all pre-loaded):
→ "This analysis requires historical data"
→ Provide best-effort with available data
→ Reference policy as fallback

Example:
Historical comparison data missing
→ "Comparing to policy standards:
   - Your metrics exceed policy minimums
   - Recommend continuation with monitoring"
```

---

## 12. Key Takeaways

1. **Stage Matters**: Workbench leverages workflow stage to provide focused guidance; Dashboard has no stage context and must be comprehensive.

2. **Timing Strategy**: Workbench loads artifacts on-demand (fast, focused); Dashboard pre-loads everything (slower initial, fast responses).

3. **Question Distribution**: Workbench favors free-form questions (60%); Dashboard favors cue-driven interactions (70%).

4. **Response Character**: Workbench provides progressive guidance ("What now?"); Dashboard provides retrospective analysis ("What happened?").

5. **Degradation Philosophy**: Workbench acknowledges workflow progress; Dashboard provides comparative/historical context.

6. **User Mental Model**:
   - Workbench users think: "I'm building this validation step-by-step, help me proceed"
   - Dashboard users think: "This validation is done, help me understand & document it"

---

## Conclusion

While both Workbench and Dashboard use the same Chat Assistant component, they leverage it very differently:

- **Workbench** is a **progressive, step-guided assistant** for active validators
- **Dashboard** is a **comprehensive, retrospective analyst** for validation reviewers

These differences emerge from their fundamental purposes and are optimized through stage context, artifact strategy, and question handling. Understanding these differences helps explain why the same question produces different responses depending on context.

