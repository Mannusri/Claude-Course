# EP20: When AI Needs a Human

## Lesson Goal

This final lesson explains when an AI system should be trusted to continue autonomously and when it should route work to a human reviewer.

The main exam topics are:

- Overall accuracy versus category-specific accuracy
- High-volume and high-stakes failure clusters
- Stratified sampling
- Field-level confidence scores
- Confidence calibration with labeled validation data
- Contradictory-field routing
- Low-confidence human review
- Provenance and claim-source mappings
- Source conflict handling
- Publication and collection dates
- Synthesis-agent governance risk

## Why Overall Accuracy Can Mislead

An AI extraction system may report 97% overall accuracy and still be unsafe for a critical document category.

Example:

```text
90% standard invoices: 99% accurate
10% handwritten receipts: 60% accurate
Overall metric: appears very strong
Critical category: fails frequently
```

High-volume categories dominate the average. Low-volume categories may contain most of the high-stakes failures.

### Exam Rule

Do not trust a single aggregate accuracy number. Find where errors cluster by document type, language, source, workflow, or risk category.

## Systematic Failure Versus Random Noise

If the system consistently fails on handwritten receipts, legal contracts, or a particular language, that is a systematic category failure rather than random noise.

A category can have:

- Low volume
- High business impact
- Persistent error patterns
- A need for dedicated human review

The correct response is to measure and route that category explicitly, not to dismiss it because the overall average is high.

## Stratified Sampling

Stratified sampling divides incoming data into logical groups, called strata, and reviews a representative sample from each group.

Possible strata include:

- Standard PDFs
- Scanned documents
- Handwritten receipts
- Legal contracts
- Different languages
- Different business units
- Different source systems
- High-value versus low-value transactions

### Workflow

```text
1. Categorize incoming records.
2. Sample from every category.
3. Review samples with human-labeled ground truth.
4. Measure accuracy per category.
5. Route weak or high-stakes categories for action.
```

### Why It Works

Stratified sampling finds category-specific failure rates without requiring humans to review every record.

A low-volume category still receives a dedicated sample. It cannot disappear inside the overall average.

### Exam Rule

When an overall metric looks strong but a particular document type is suspected of failing, use stratified sampling and report accuracy by category.

## Accuracy by Category

Track metrics such as:

```text
standard_pdf_accuracy
handwritten_receipt_accuracy
legal_contract_accuracy
multilingual_accuracy
high_value_transaction_accuracy
```

Do not report only:

```text
overall_accuracy
```

A category-specific metric supports a better routing decision than a global average.

## Field-Level Confidence

Confidence should be attached to individual extracted fields, not only to the document as a whole.

Example:

```json
{
  "invoice_number": {
    "value": "INV-2048",
    "confidence": 0.97,
    "source_region": "top-right header"
  },
  "total_amount": {
    "value": 4250,
    "confidence": 0.91,
    "source_region": "bottom table row 12"
  },
  "vendor_address": {
    "value": "...",
    "confidence": 0.63,
    "source_region": "handwritten address block"
  }
}
```

The document may be mostly correct while one field is unreliable.

### Why Document-Level Confidence Is Weak

A document-level score can hide a low-confidence address, date, amount, or identifier inside an otherwise accurate extraction.

Field-level confidence enables targeted review:

- Human reviews the uncertain field.
- Reliable fields continue automatically.
- The entire document does not need to be reprocessed manually.

## Confidence Calibration

Raw model confidence is not automatically trustworthy. Calibrate confidence against a labeled validation set where the correct answer is known.

### Calibration Workflow

1. Create a representative labeled validation set.
2. Run the extraction system on it.
3. Compare reported confidence with actual correctness.
4. Measure false-accept and false-reject rates by field and category.
5. Select thresholds based on business risk.
6. Recalibrate as data and model behavior change.

A reported confidence of `0.85` is meaningful only if validation shows how often fields at that level are actually correct.

### Exam Rule

Do not use raw self-reported confidence as an authorization or escalation policy. Calibrate field-level scores with labeled ground-truth data.

## Human Review Routing

Route a record or field to human review when:

- A field's calibrated confidence is below its threshold.
- Fields contradict each other.
- A high-stakes category has known systematic failures.
- Required provenance is missing.
- A business rule fails.
- The document contains an unsupported or ambiguous interpretation.

Do not silently discard the uncertain field and do not force it through downstream systems.

## Contradictory Fields

A document can produce individually valid fields that are inconsistent together.

Examples:

- Total amount does not equal the sum of line items.
- Due date precedes invoice date.
- Extracted birth date is in the future.
- Currency conflicts with the account or transaction.
- Customer ID does not match the order owner.
- A contract's effective date conflicts with its stated term.

Contradictions should be flagged for validation or human review.

## Anti-Patterns for Low-Confidence Fields

### Silent Discard

Dropping a low-confidence field creates an invisible gap. Downstream systems may assume the information was never needed or may operate on incomplete data without knowing it.

### Force Through

Passing a low-confidence or contradictory value downstream contaminates later decisions and may turn an uncertain interpretation into an apparently trusted fact.

### Correct Routing

```text
Low confidence or contradiction
    -> Preserve the value and source context
    -> Mark the field for review
    -> Route to a human or coordinator
    -> Update the record after resolution
```

## Provenance

Provenance means preserving where a claim or extracted value came from.

Every important claim should retain:

- Source ID
- Source URL or document name
- Page or section
- Publication date
- Collection date
- Extraction location
- Confidence
- Method or agent that produced it

### Claim-Source Mapping

```json
{
  "claim_id": "C001",
  "text": "Adoption increased by 47% year over year.",
  "source_id": "S001",
  "source": "mckinsey-ai-adoption-2024.pdf",
  "section": "Enterprise adoption trends",
  "publication_date": "2024-05-10",
  "collection_date": "2026-09-06",
  "confidence": 0.91
}
```

The mapping must survive every agent boundary:

```text
Search agent -> Analysis agent -> Synthesis agent -> Report agent -> Human reviewer
```

## Provenance Responsibilities by Agent

### Search Agent

Passes source URL or document ID, publication date, credibility metadata, and the initial claim.

### Analysis Agent

Preserves page numbers, sections, quotations, extraction location, and source identifiers.

### Synthesis Agent

Combines claims without dropping or mixing source tags. This is a high-risk point for provenance loss.

### Report Agent

Includes citations and source mappings in the final output so a human can verify every important claim.

### Exam Rule

The synthesis stage is especially risky because multiple claims and sources are combined. Require source fields in both the synthesis prompt and output schema.

## Conflicting Sources

When two sources disagree, do not:

- Pick the highest value
- Pick the lowest value
- Average the values
- Silently discard one source
- Present one value as fact without qualification

Correct workflow:

1. Preserve both claims.
2. Preserve each source mapping.
3. Mark the conflict as unresolved.
4. Include methodology and dates.
5. Let the coordinator investigate.
6. Escalate to a human when the conflict cannot be resolved safely.

Example:

```json
{
  "conflict_id": "CONF-2024-MARKET",
  "topic": "2024 market size",
  "claims": [
    {"value": "12 billion", "source_id": "S001"},
    {"value": "18 billion", "source_id": "S002"}
  ],
  "resolution_state": "unresolved",
  "next_action": "Human review of source methodologies"
}
```

## Temporal Data

Facts have a time dimension. A value published in 2019 is not automatically a current 2024 or 2026 value.

Every important record should distinguish:

- `publication_date`: When the source originally published or measured the fact
- `collection_date`: When the agent retrieved the fact

Example:

```json
{
  "claim": "Average processing time was 12 days.",
  "publication_date": "2019-06-01",
  "collection_date": "2026-09-06",
  "temporal_status": "historical"
}
```

### Why Both Dates Matter

- Publication date describes the age of the underlying evidence.
- Collection date describes when the pipeline observed it.
- Downstream users can judge freshness.
- Conflicting or stale data can be routed for review.

### Exam Rule

Require publication and collection dates in structured outputs. Do not try to solve temporal ambiguity with retries or vague prompt wording.

## Human-in-the-Loop Decision Framework

```text
Is the field confidence below its calibrated threshold?
    Yes -> Human review.
    No  -> Continue.

Are fields contradictory or business rules violated?
    Yes -> Human review or coordinator investigation.
    No  -> Continue.

Is the category known to have systematic failures?
    Yes -> Stratified monitoring and targeted review.
    No  -> Continue.

Is provenance missing or a source conflict unresolved?
    Yes -> Escalate for verification.
    No  -> Continue autonomously.
```

## Exam Scenarios

### Scenario 1: High Overall Accuracy

**Question:** An extraction system has 97% overall accuracy but only 60% accuracy on handwritten receipts. What is the correct response?

**Answer:** Use stratified sampling and category-specific metrics, then route handwritten receipts to human review until the category is reliable.

### Scenario 2: Sampling a Low-Volume Category

**Question:** Handwritten receipts are only 5% of the input volume. Should they be excluded from validation sampling?

**Answer:** No. Each logical category needs a representative sample, especially when it is low-volume but high-stakes.

### Scenario 3: Field-Level Uncertainty

**Question:** An invoice number and amount are high confidence, but the vendor address is low confidence. What should happen?

**Answer:** Route the uncertain field or record for targeted human review rather than rejecting all fields or passing the address through silently.

### Scenario 4: Confidence Calibration

**Question:** The model reports 0.85 confidence, but no labeled validation set exists. Can the threshold be trusted?

**Answer:** No. Calibrate confidence against labeled ground-truth data before using it for routing decisions.

### Scenario 5: Contradictory Fields

**Question:** The line-item sum does not equal the extracted total. What should happen?

**Answer:** Flag the contradiction and route it for validation or human review. Do not silently choose one value.

### Scenario 6: Provenance Loss

**Question:** A final report contains a 47% statistic, but no source or page reference. What is the fix?

**Answer:** Preserve claim-source mappings through every agent boundary and require source metadata in the output schema.

### Scenario 7: Conflicting Sources

**Question:** Two sources report different market sizes. Should the system average them?

**Answer:** No. Preserve both attributed claims, mark the conflict unresolved, and escalate for coordinator or human review.

### Scenario 8: Historical Fact

**Question:** A research claim was published in 2019 but retrieved today. What metadata is required?

**Answer:** Include both publication date and collection date so downstream users can assess temporal validity.

## Exam Anti-Patterns

### 1. Trusting Aggregate Accuracy

High-volume easy cases can hide severe category-specific failures.

### 2. Sampling Only the Largest Category

Low-volume high-stakes categories require their own samples.

### 3. Document-Level Confidence Only

A document can be mostly correct while one critical field is wrong. Use field-level confidence.

### 4. Raw Confidence as Truth

Calibrate scores with labeled validation data before routing.

### 5. Discarding Low-Confidence Fields

Silent gaps contaminate downstream systems. Preserve and route them for review.

### 6. Forcing Uncertain Values Through

Do not pass hallucinated or contradictory values into downstream decisions.

### 7. Dropping Provenance During Synthesis

Every claim must retain source metadata across agent boundaries.

### 8. Averaging Conflicting Sources

Averaging disagreement creates a value no source actually reported.

### 9. Omitting Temporal Metadata

Without publication and collection dates, old facts may be mistaken for current facts.

## Exam Checklist

- Overall accuracy can hide category-specific failures.
- Use stratified sampling across document or risk categories.
- Track accuracy by category, not only globally.
- Confidence belongs at the field level.
- Calibrate confidence with labeled validation sets.
- Route low-confidence or contradictory fields to human review.
- Do not discard uncertain fields silently.
- Do not force uncertain values into downstream systems.
- Preserve claim-source mappings through every agent boundary.
- Treat the synthesis agent as a high-risk provenance-loss point.
- Annotate both sides of source conflicts and escalate unresolved conflicts.
- Include publication and collection dates in structured outputs.
- Human review is a reliability mechanism, not an admission of system failure.

## Final Summary

A reliable AI system does not optimize only for average accuracy. It identifies where failures cluster, measures each category, routes uncertain fields to humans, preserves provenance, and represents temporal context explicitly.

For the exam, remember: **use stratified sampling for category failures, field-level calibrated confidence for routing, claim-source mappings for provenance, publication and collection dates for temporal validity, and human review for low-confidence, contradictory, or unresolved high-stakes cases.**
