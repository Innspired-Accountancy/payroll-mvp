# Slice a: Correction Eligibility and Chain Management

**Story:** story-08-corrections
**Epic:** epic-02-hmrc-submissions
**Effort:** M
**Dependencies:** None

---

## Goal

Implement correction eligibility checks and submission chain tracking. Determine if a submission can be corrected based on HMRC rules (current tax year vs prior year) and maintain parent-child relationships for audit trail.

---

## Decision Checklist

- [x] All libraries/packages named: date-fns 3.x (date utilities), Drizzle ORM 0.30.x
- [x] SDK methods identified: db.query.hmrcSubmissions.findMany(), date comparisons
- [x] External service endpoints: N/A (internal logic)
- [x] Data contracts defined: CorrectionEligibility, SubmissionChain, PriorYearRules interfaces
- [x] Configuration: PRIOR_YEAR_CORRECTION_START_MONTH=4 (April)
- [x] Error scenarios: Ineligible for correction, chain break, prior year deadline passed
- [x] No "TBD", slash-notation, or placeholder text

---

## Spec References

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 2: System maintains submission chain
- HMRC RTI Correction Guidelines

---

## Files in Scope

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/hmrc/corrections/eligibility.ts` | create | Correction eligibility checker |
| `src/lib/hmrc/corrections/chain.ts` | create | Submission chain manager |
| `src/lib/hmrc/corrections/types.ts` | create | Correction types |

---

## Responsibilities

1. Check if submission is eligible for correction
2. Determine current vs prior tax year correction rules
3. Track submission chain (original → corrections)
4. Validate correction timing (prior year corrections only from 6 April)
5. Generate next version number

---

## Contracts

### CorrectionEligibilityChecker.check()
- **Method:** `async check(submissionId: string): Promise<EligibilityResult>`
- **Input:** `submissionId: string` — UUID of submission to correct
- **Output:** EligibilityResult
  ```typescript
  interface EligibilityResult {
    eligible: boolean;
    reason?: 'not_rejected' | 'tax_year_closed' | 'deadline_passed' | 'already_corrected';
    submissionType: 'fps' | 'eps';
    originalSubmission: {
      id: string;
      correlationId: string;
      taxYear: string;
      taxPeriod: number;
      status: string;
      submittedAt: Date;
      version: number;
    };
    correctionType: 'current_year' | 'prior_year';
    rules: {
      canSubmitImmediately: boolean;
      earliestSubmissionDate?: Date; // For prior year: 6 April following year
      deadlineDate?: Date;
      requiresLateReason: boolean;
    };
    chain: {
      rootSubmissionId: string;
      currentVersion: number;
      nextVersion: number;
      chainLength: number;
    };
  }
  ```

### Tax Year Correction Rules
```typescript
const CORRECTION_RULES = {
  currentYear: {
    // Corrections in current tax year
    eligibleStatuses: ['rejected', 'accepted'], // Can correct even if accepted
    canSubmitImmediately: true,
    requiresLateReason: (submissionDate: Date, correctionDate: Date) => {
      // Late if after 19th of tax month
      const taxMonthEnd = getTaxMonthEnd(submissionDate);
      return correctionDate > addDays(taxMonthEnd, 19);
    }
  },
  priorYear: {
    // Corrections for prior tax years
    eligibleStatuses: ['accepted'], // Only correct accepted submissions
    earliestSubmissionDate: (taxYear: string) => {
      // Can only correct from 6 April following tax year end
      const year = parseInt(taxYear.split('-')[0]) + 1;
      return new Date(year, 3, 6); // 6 April
    },
    deadlineDate: (taxYear: string) => {
      // No specific deadline, but recommend by following 19 April
      const year = parseInt(taxYear.split('-')[0]) + 1;
      return new Date(year, 3, 19); // 19 April
    },
    requiresLateReason: true // Always late for prior year
  }
};
```

### SubmissionChainManager
```typescript
class SubmissionChainManager {
  // Get full chain from root to latest
  async getChain(rootSubmissionId: string): Promise<SubmissionChain>;
  
  // Get next version number
  async getNextVersion(parentSubmissionId: string): Promise<number>;
  
  // Create chain link
  async createLink(
    parentSubmissionId: string,
    childSubmissionId: string
  ): Promise<ChainLink>;
}

interface SubmissionChain {
  rootSubmission: SubmissionSummary;
  corrections: SubmissionSummary[];
  totalVersions: number;
  currentVersion: number;
}
```

---

## Business Rules & Invariants

1. Only rejected or accepted submissions can be corrected
2. Current year corrections can be submitted immediately
3. Prior year corrections only allowed from 6 April following tax year
4. Late reporting reason always required for prior year corrections
5. Submission chain tracks all versions for audit
6. Root submission is never deleted (chain integrity)

---

## Edge Cases

1. **Submission already corrected** — Return chain with latest version
2. **Tax year closed for new submissions** — Block, suggest prior year correction
3. **Multiple corrections in chain** — Linear chain (no branching)
4. **Original submission deleted** — Chain broken, flag for admin
5. **Prior year correction before 6 April** — Block with earliest date message

---

## Tests

### correction-eligibility.test.ts
- Check eligible current year correction
- Check eligible prior year correction (after 6 April)
- Block prior year correction before 6 April
- Get submission chain with multiple corrections
- Calculate next version number

---

## Verification

```bash
npm run typecheck
npm run test src/lib/hmrc/corrections/eligibility.test.ts
npm run lint src/lib/hmrc/corrections/
```

---

## Source Sections

- 02-02-hmrc-submissions-spec.md § User Journeys → Journey 2: System maintains submission chain
- 02-02-hmrc-submissions-spec.md § Non-Functional Requirements → Support prior tax year corrections
