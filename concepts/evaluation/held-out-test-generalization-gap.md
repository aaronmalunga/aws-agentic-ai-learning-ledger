# Held-Out Test Generalization Gap

## Why the final test changed the interpretation of the routing benchmark

The routing model-selection workflow selected `C2-N1-C2.0` before the held-out test was opened.

The selected architecture was:

| Component | Choice |
|---|---|
| Text representation | TF-IDF word unigrams |
| Classifier | Linear SVM |
| Regularization | `C = 2.0` |

The candidate was selected using training, validation, preserved failure evidence, and synthetic operational measurements.

The held-out test was not consulted during selection.

---

## Validation versus held-out test

The most important final result was the difference between validation and the untouched test set.

| Metric | Validation | Held-out test |
|---|---:|---:|
| Accuracy | 0.9722 | 0.7500 |
| Macro F1 | 0.9722 | 0.7403 |
| Failures | 1 / 36 | 9 / 36 |
| BUG recall | 1.0000 | 1.0000 |
| PLATFORM recall | 0.9167 | 0.6667 |
| OTHER recall | 1.0000 | 0.5833 |

The validation-to-test macro-F1 gap was approximately:

`0.2319`

This showed that the validation result had overestimated generalization to additional unseen semantic families.

---

## Why this is not a reason to tune against TEST

Once the held-out test was opened, its examples became observed evidence.

Using the nine failures to modify the model and then rerunning the same test would turn the test set into development data.

Therefore Routing v1 was closed without:

- changing the classifier;
- changing `C`;
- adding new n-grams;
- adding rules for observed failures;
- modifying labels;
- adding test records to training;
- rerunning the test after adaptation.

The principle is:

> A held-out test result is evidence, not permission to keep optimizing against the test set.

---

## What the failure pattern revealed

The nine errors were concentrated in several semantic families.

| Family | Failures | Observed issue |
|---|---:|---|
| `other-food-order-request` | 3 | Commerce-like words such as `order` and `buy` overlapped with retailer-support vocabulary |
| `platform-account-deletion` | 2 | Valid support requests fell to OTHER |
| `platform-exchange-policy` | 2 | Exchange language was confused with OTHER and BUG |
| `other-stock-market-question` | 1 | `stock` appeared in an unrelated financial context |
| `other-chess-advice` | 1 | Unrelated advice was classified as PLATFORM |

The result suggests that unigram lexical representations can confuse words that are individually domain-relevant but semantically unrelated in context.

That interpretation is a hypothesis supported by the failure pattern, not a proven causal explanation.

---

## The strongest positive result

All held-out BUG cases were still classified correctly.

| Actual BUG cases | Correct |
|---:|---:|
| 12 | 12 |

BUG recall therefore remained:

`1.0000`

The primary weakness was instead the PLATFORM/OTHER boundary and false activation of support-oriented classes for unrelated requests.

---

## Why the original model-selection decision remains valid

The held-out result does not mean the pre-test selection decision was incorrect.

Before test exposure, the available evidence was:

| Candidate | Validation macro F1 | Failures |
|---|---:|---:|
| B1 deterministic rules | 0.2767 | 22 |
| C1-N1-C2.0 | 0.9444 | 2 |
| C2-N1-C2.0 | **0.9722** | **1** |

`C2-N1-C2.0` was therefore the strongest candidate under the evidence available at selection time.

The held-out test answered a different question:

> How well does that frozen decision generalize to another untouched set of semantic families?

The answer was weaker than validation suggested.

That is precisely why independent test data exists.

---

## What a future Routing v2 would require

Routing v2 may investigate hypotheses such as:

| Hypothesis | Motivation |
|---|---|
| More domain-negative examples | Separate retailer ordering from unrelated food ordering |
| Hierarchical routing | Determine domain relevance before BUG / PLATFORM classification |
| Confidence or abstention | Escalate uncertain classifications safely |
| Alternative lexical features | Test robustness beyond current unigrams |
| Embedding classifier | Introduce stronger semantic representation if justified |
| Larger semantic-family coverage | Improve generalization estimation |

However, Routing v2 must use new independent evaluation evidence.

The existing held-out set can no longer serve as an untouched final test.

---

## Engineering lesson

The most valuable outcome was not the validation score.

It was discovering that:

> A model can look excellent on a carefully protected validation set and still expose meaningful weaknesses on another independently held-out set.

Without the final test, it would have been easy to report `0.9722` macro F1 as if it represented final generalization.

The correct evidence is more nuanced:

| Question | Conclusion |
|---|---|
| Did classical ML beat the deterministic baseline? | Yes |
| Did validation show strong performance? | Yes |
| Did TEST confirm that performance level? | No |
| Did the test reveal useful failure structure? | Yes |
| Should the same TEST now become tuning data? | No |
| Is a future versioned experiment justified? | Potentially |

The discipline of preserving an unfavorable result is more valuable than manufacturing a flattering benchmark.