# Testing AI Models on R Code Reasoning

## A Small Benchmark of R Semantics, Missing Values, `dplyr`, and Statistical Interpretation

### Overview

This report compares several AI coding models on a compact R reasoning task. The code itself is simple: a small grouped data frame, a grouped mean, centering, filtering, and a correlation. However, the task is deliberately designed to test whether a model can **mentally execute R code correctly**, rather than merely recognize familiar R syntax.

The benchmark exposes several common failure modes:

- misunderstanding how `group_by()` affects `mutate()`,
- incorrectly assuming `mean()` removes missing values by default,
- misunderstanding how `filter()` treats `NA`,
- failing to recognize that groups can disappear entirely after filtering,
- misunderstanding the default missing-value behavior of `cor()`,
- confusing valid R code with syntax errors,
- changing the requested statistical estimand instead of implementing it,
- and failing to distinguish programming behavior from statistical interpretation.

The tested models were:

1. Codex
2. Nemotron-3-Super:cloud
3. Gemma4:latest (9.6 GB)
4. Nemotron-3.5-Lightning:latest (25 GB)
5. GPT-OSS 20B (13 GB)
6. Qwen3-Coder:30B (18 GB)
7. DeepSeek-Coder (779 MB)
8. Gemma 4E4B (8 GB)
9. Qwen2.5-Coder (4.7 GB)
10. Mistral:latest (4.4 GB)

---

# 1. Benchmark Prompt

The following prompt was given to the models.

```r
library(dplyr)

df <- data.frame(
  group = factor(c("A", "A", "A", "B", "B", "B")),
  x = c(1, 2, NA, 4, 5, 6),
  y = c(2, 4, 6, 8, NA, 12)
)

result <- df %>%
  group_by(group) %>%
  mutate(
    mean_x = mean(x),
    x_centered = x - mean_x
  ) %>%
  filter(x_centered > 0) %>%
  summarise(
    n = n(),
    correlation = cor(x, y)
  )

print(result)
```

Without running the code:

1. Explain exactly what happens when this code is executed.
2. Identify every problem in the code, including problems that do not generate an explicit R error.
3. Explain why each problem occurs.
4. Rewrite the code so that it:
   - computes the mean of `x` separately within each group while ignoring missing values,
   - centers `x` around its group mean,
   - keeps observations with a positive centered value,
   - reports the number of remaining observations per group,
   - calculates the correlation between `x` and `y` using all available complete pairs.
5. State the exact output you expect from the corrected code.
6. Explain whether calculating the correlation after filtering has a different interpretation from calculating the correlation before filtering.

Do not simply provide corrected code. Explain the behavior of the original code step by step and distinguish between:

- syntax errors,
- missing-value issues,
- statistical/logical issues.

---

# 2. Why This Is a Useful Benchmark

At first glance, this looks like a basic `dplyr` exercise. In reality, it tests several layers of R knowledge at once.

A weak model can often produce something that *sounds* like an R explanation while missing the actual execution semantics. For example, it may know that `na.rm = TRUE` exists, but still incorrectly claim that `mean()` ignores `NA` by default.

A strong model needs to reason through the exact state of the data after every pipeline operation.

The task therefore tests both:

- **language-specific programming knowledge**, and
- **statistical reasoning about the meaning of the result**.

---

# 3. Reference Solution

## 3.1 Original Code: Exact Execution

The code contains no syntax error.

Because the data are grouped using:

```r
group_by(group)
```

the following calculation is performed separately for groups A and B:

```r
mean_x = mean(x)
```

### Group A

For group A:

```r
x = c(1, 2, NA)
```

R's `mean()` uses `na.rm = FALSE` by default:

```r
mean(c(1, 2, NA))
# NA
```

Therefore:

```r
mean_x = NA
```

for every row in group A.

The centered values become:

```r
x - mean_x
```

which gives:

```r
c(NA, NA, NA)
```

because arithmetic involving `NA` produces `NA`.

### Group B

For group B:

```r
x = c(4, 5, 6)
```

so:

```r
mean_x = 5
```

and:

```r
x_centered = c(-1, 0, 1)
```

---

## 3.2 Effect of `filter()`

The code then evaluates:

```r
filter(x_centered > 0)
```

For group A:

```r
NA > 0
# NA
```

`dplyr::filter()` keeps rows only where the condition evaluates to `TRUE`.

Therefore all rows in group A are removed.

For group B:

```text
x = 4  -> x_centered = -1 -> removed
x = 5  -> x_centered =  0 -> removed
x = 6  -> x_centered =  1 -> retained
```

Only one row remains:

```text
group = B
x = 6
y = 12
```

Importantly, **group A disappears entirely**. `summarise()` does not automatically produce an `A, n = 0` row.

---

## 3.3 Original Output

The remaining group is B with one observation.

Therefore:

```r
n()
# 1
```

and:

```r
cor(6, 12)
```

cannot estimate a Pearson correlation from a single observation.

The resulting value is `NA`.

The original code therefore produces essentially:

```text
# A tibble: 1 × 3
  group     n correlation
  <fct> <int>       <dbl>
1 B         1          NA
```

---

# 4. Corrected Code

A direct correction that follows the requested order of operations is:

```r
library(dplyr)

result <- df %>%
  group_by(group) %>%
  mutate(
    mean_x = mean(x, na.rm = TRUE),
    x_centered = x - mean_x
  ) %>%
  filter(x_centered > 0) %>%
  summarise(
    n = n(),
    correlation = cor(x, y, use = "complete.obs"),
    .groups = "drop"
  )

print(result)
```

---

# 5. Corrected Output

### Group A

Ignoring the missing value:

```r
mean(c(1, 2, NA), na.rm = TRUE)
# 1.5
```

Therefore:

```text
x            1      2     NA
x_centered  -0.5    0.5   NA
```

Only the row with:

```text
x = 2
y = 4
```

survives.

Thus:

```text
n = 1
correlation = NA
```

### Group B

The group mean is:

```text
5
```

and the centered values are:

```text
-1, 0, 1
```

Only:

```text
x = 6
y = 12
```

survives.

Thus:

```text
n = 1
correlation = NA
```

The corrected result is:

```text
# A tibble: 2 × 3
  group     n correlation
  <fct> <int>       <dbl>
1 A         1          NA
2 B         1          NA
```

The `NA` correlations are not an error in the corrected code. They occur because only one complete observation remains in each group.

---

# 6. Statistical Interpretation

Calculating the correlation before and after filtering answers different questions.

Before filtering, the within-group correlation is based on all available complete pairs.

For group A:

```text
(1, 2)
(2, 4)
```

which gives:

```text
r = 1
```

For group B:

```text
(4, 8)
(6, 12)
```

which also gives:

```text
r = 1
```

After filtering on:

```r
x_centered > 0
```

the correlation is conditional on observations having `x` above their group-specific mean.

The estimand is therefore different.

The post-filter correlation should not automatically be described as "wrong" or "biased." Whether it is appropriate depends on the research question. It is simply a correlation for a selected subpopulation rather than the full group.

---

# 7. Scoring Rubric

The models were evaluated on the following dimensions.

| Criterion | What was tested |
|---|---|
| R execution tracing | Does the model correctly simulate the code without running it? |
| `group_by()` semantics | Does it understand grouped `mutate()` and grouped `summarise()`? |
| `mean()` + `NA` | Does it know that `mean()` does not ignore `NA` by default? |
| `filter()` semantics | Does it understand `NA > 0` and row removal? |
| Empty-group behavior | Does it know that a group can disappear completely after filtering? |
| `cor()` semantics | Does it understand the default missing-value behavior and one-observation result? |
| Corrected code | Does the proposed correction actually satisfy the prompt? |
| Exact output | Does it calculate the final result rather than merely describe it? |
| Statistical interpretation | Does it distinguish the full-sample and filtered correlation correctly? |
| Task fidelity | Does it solve the requested problem rather than silently changing the estimand? |

Scores are approximate and intended as comparative assessments rather than formal psychometric measurements.

---

# 8. Results

| Rank | Model | Score | Overall assessment |
|---:|---|---:|---|
| 1 | **Codex** | **9.5/10** | Excellent R execution reasoning with only minor overengineering |
| 2 | **Nemotron-3-Super:cloud** | **6.5/10** | Strong understanding, but important mistakes in empty-group and `cor()` semantics |
| 3 | **Gemma4:latest (9.6 GB)** | **6.0/10** | Good core reasoning and corrected output, but misses the disappearing-group edge case and misstates `cor()` behavior |
| 4 | **Nemotron-3.5-Lightning:latest (25 GB)** | **6.0/10** | Strong row-level tracing and valid correction, but incorrect empty-group handling and contradictory exact output |
| 5 | **GPT-OSS 20B (13 GB)** | **5.0/10** | Good execution tracing, weaker API semantics and task fidelity |
| 6 | **Qwen3-Coder:30B (18 GB)** | **3.5/10** | Corrected code is mostly good, but the original execution is misunderstood at a fundamental level |
| 7 | **DeepSeek-Coder (779 MB)** | **3.0/10** | Recognized broad issues but did not actually execute the code correctly |
| 8 | **Gemma 4E4B (8 GB)** | **2.5/10** | Polished explanation but several basic R errors and changed the requested analysis |
| 9 | **Qwen2.5-Coder (4.7 GB)** | **2.0/10** | Major misunderstandings of valid R syntax, grouping, missing values, and object creation |
| 10 | **Mistral:latest (4.4 GB)** | **1.5/10** | Fails core missing-value semantics, changes the task, and proposes broken replacement code |

---

# 9. Model-by-Model Evaluation

## 9.1 Codex

### Score: 9.5/10

Codex produced by far the strongest response.

It correctly identified:

```r
mean(c(1, 2, NA))
# NA
```

and followed the consequences through the entire pipeline.

It correctly concluded that:

- all rows from group A disappear,
- only `B: x = 6, y = 12` remains,
- the original result contains only group B,
- the original correlation is `NA`,
- the corrected result contains A and B with `n = 1`,
- and both corrected correlations remain `NA`.

It also explicitly distinguished the default `cor()` behavior from the one-observation problem. In the original pipeline, the final `NA` is not caused by a missing `y`, because the only surviving row is complete.

Another particularly strong observation was that:

```r
n()
```

is not necessarily the same as the effective number of complete observations used by:

```r
cor(x, y, use = "complete.obs")
```

A retained row can count toward `n()` while still being excluded from the correlation if one of the variables is missing.

### Weakness

The corrected code included defensive checks for minimum sample size and distinct values. This is robust production code but somewhat more complicated than necessary for the benchmark.

### Verdict

Codex demonstrated genuine R-specific reasoning rather than merely recognizing syntax patterns.

---

## 9.2 Nemotron-3-Super:cloud

### Score: 6.5/10

Nemotron-3-Super:cloud correctly understood the most important missing-value behavior:

```r
mean(c(1, 2, NA))
# NA
```

It also correctly derived:

```text
Group A: x_centered = NA, NA, NA
Group B: x_centered = -1, 0, 1
```

and correctly recognized that only the final B row survives.

However, it then made a major execution error by claiming that the original result would contain:

```text
A   0   NA
B   1   NA
```

This is wrong.

After `filter()` removes every A observation, group A no longer appears in the grouped data. `summarise()` does not automatically recreate the empty group.

The correct original output contains only B.

Nemotron-3-Super:cloud also made smaller errors:

- it described `NA` in a filter condition as yielding `FALSE` rather than remaining `NA`,
- it misstated the default missing-value behavior of `cor()`,
- and it described the standard deviation of a one-element vector as zero rather than `NA`.

Its corrected code was nevertheless largely valid and its statistical interpretation was generally good.

### Verdict

A meaningful step up from weaker models, but still unreliable on subtle execution semantics.

---

## 9.3 Gemma4:latest (9.6 GB)

### Score: 6.0/10

Gemma4:latest correctly understood that `group_by(group)` makes `mean(x)` group-specific and that `mean()` does not remove `NA` by default.

It also correctly worked out the corrected group means, centered values, surviving observations, and the final corrected result with `n = 1` and `correlation = NA` for both A and B.

The main execution error was the prediction that the original result would still contain both groups.

In the original pipeline, every A row is removed and group A disappears completely. `summarise()` therefore returns only the surviving B group.

Gemma4:latest also incorrectly suggested that `cor()` automatically ignores incomplete pairs.

Base R defaults to:

```r
use = "everything"
```

so complete-pair handling must be requested explicitly.

In this particular pipeline, however, the final original `NA` is caused by the single surviving observation rather than by a missing `y`.

The corrected code was mostly good. Using `pairwise.complete.obs` is acceptable for this two-variable case, although `complete.obs` matches the benchmark wording more directly.

An additional `is.finite()` condition was unnecessary and slightly changed the specification.

### Verdict

Good R knowledge and a strong correction, but not precise enough on the benchmark's empty-group and `cor()` traps.

---

## 9.4 Nemotron-3.5-Lightning:latest (25 GB)

### Score: 6.0/10

Nemotron-3.5-Lightning:latest traced the grouped means and centered values well.

It correctly obtained:

```text
Group A:
mean_x = NA
x_centered = NA, NA, NA
```

and:

```text
Group B:
mean_x = 5
x_centered = -1, 0, 1
```

It also correctly recognized that only the B row with:

```text
x = 6
y = 12
```

survives the original filter.

Its main error came immediately afterward: it reported group A as an empty `n = 0` row in the final summary.

That is not how the default grouped pipeline behaves. Once every A row has been filtered out, A is absent from the data passed to `summarise()`.

It also incorrectly stated that group B still contained the `y = NA` row after filtering.

That row has:

```text
x = 5
x_centered = 0
y = NA
```

and is already removed because:

```r
0 > 0
# FALSE
```

The original B correlation is therefore `NA` because only the single complete pair `(6, 12)` remains.

The corrected code itself was valid and the response eventually self-corrected the corrected A count from 0 to 1. However, it left the earlier contradictory output in place, so the exact-output portion remained unreliable.

Its statistical interpretation was generally sound, although describing the post-filter correlation as automatically "biased" was too strong.

Filtering changes the target subpopulation and estimand; whether that is appropriate depends on the research question.

### Verdict

Strong core mechanics and a good repair, but an important `dplyr` empty-group mistake prevents a higher score.

---

## 9.5 GPT-OSS 20B (13 GB)

### Score: 5.0/10

GPT-OSS 20B did a good job tracing the original pipeline.

It correctly recognized:

- grouped computation,
- `mean(x)` returning `NA` in group A,
- propagation of `NA` through `x_centered`,
- disappearance of group A,
- survival of only the final B observation,
- and an undefined correlation for that one remaining observation.

This is an important strength because several other models failed much earlier.

However, it incorrectly claimed that `cor()` automatically excludes missing values by default.

In R, the default is effectively:

```r
use = "everything"
```

so missing values generally propagate to `NA`.

It also made an inaccurate claim about `summarise()` retaining grouping. With one grouping variable and standard modern `dplyr` behavior, the final grouping level is typically dropped.

The largest conceptual problem was that it changed the requested analysis.

Instead of calculating the correlation per group after filtering, it proposed an overall correlation on the remaining data and repeated that value for each group.

That is a different estimand.

### Verdict

Good mental execution of the original code, but weaker knowledge of API details and a tendency to redesign the analysis instead of following the prompt.

---

## 9.6 Qwen3-Coder:30B (18 GB)

### Score: 3.5/10

Qwen3-Coder:30B's corrected pipeline was considerably better than its explanation of the original code.

In the correction, it correctly used:

```r
mean(x, na.rm = TRUE)
```

centered within groups, filtered positive centered values, and arrived at one surviving observation in each group with an undefined correlation.

The original execution, however, was fundamentally misunderstood.

It repeatedly claimed that:

```r
mean(x)
```

inside grouped `mutate()` computes a global mean and even calculated a global value while silently ignoring the `NA`.

Both statements are wrong.

Grouping makes the mean group-specific, and `mean()` does not remove missing values unless:

```r
na.rm = TRUE
```

is supplied.

It therefore failed to derive the actual original output.

It also described:

```r
use = "complete.obs"
```

as the default for `cor()`, which is incorrect.

A later answer about interpretation correctly distinguished correlation before filtering from correlation conditional on:

```r
x_centered > 0
```

but that did not repair the core execution mistakes.

### Verdict

Able to construct a plausible repair, but unreliable when asked what the original R code actually does.

---

## 9.7 DeepSeek-Coder (779 MB)

### Score: 3.0/10

DeepSeek-Coder identified some relevant themes:

- missing values,
- centering,
- filtering,
- and correlation.

However, it failed to reason through the actual execution.

Most importantly, it stated that the original code computes the mean of `x` within each group while ignoring missing values.

That is false.

The original code uses:

```r
mean(x)
```

not:

```r
mean(x, na.rm = TRUE)
```

This mistake prevents it from understanding the central failure of group A.

The response also:

- failed to derive the actual surviving rows,
- failed to provide the requested exact output,
- described valid operations as incorrect without explaining the actual semantics,
- and provided no real corrected code despite having a "corrected code" section.

### Verdict

The response recognized relevant R vocabulary but did not convincingly execute the program.

---

## 9.8 Gemma 4E4B (8 GB)

### Score: 2.5/10

Gemma 4E4B's answer was polished and detailed, but it contained several fundamental R errors.

The most important was:

> `mean()` automatically ignores `NA` values.

This is false.

It therefore incorrectly calculated the original group A mean as:

```text
1.5
```

instead of:

```text
NA
```

Its proposed corrected code also included:

```r
cor(x, y, use = "pairwise.complete.obs", na.rm = TRUE)
```

but `cor()` does not have an `na.rm` argument.

This code would error.

Gemma also proposed:

```r
sum(x_centered > 0)
```

without:

```r
na.rm = TRUE
```

For group A:

```r
sum(c(FALSE, TRUE, NA))
# NA
```

not `1`.

Finally, Gemma changed the requested statistical task by moving the correlation calculation before filtering and then argued that this was the "correct statistical approach."

That may be a reasonable alternative analysis, but it does not answer the benchmark as written.

### Verdict

A good example of how fluent statistical prose can conceal weak language-specific execution reasoning.

---

## 9.9 Qwen2.5-Coder (4.7 GB)

### Score: 2.0/10

Qwen2.5-Coder incorrectly stated that:

```r
mean_x = mean(x)
```

calculates the overall mean across all groups.

This ignores the effect of:

```r
group_by(group)
```

It also claimed that:

```r
filter(x_centered > 0)
```

was syntactically incorrect because `x_centered` was undefined.

But `x_centered` had been created immediately beforehand in `mutate()`.

The model further claimed that:

```r
result
```

was never defined, despite the code clearly assigning:

```r
result <- df %>% ...
```

It also repeated the false claim that `mean()` excludes missing values by default.

Surprisingly, its rewritten code was mostly valid:

```r
mean(x, na.rm = TRUE)
```

and:

```r
cor(x, y, use = "complete.obs")
```

were both appropriate corrections.

However, it failed to provide the exact requested output.

### Verdict

The model showed some ability to produce plausible corrected R code, but its explanation of the original program was unreliable at a very basic level.

---

## 9.10 Mistral:latest (4.4 GB)

### Score: 1.5/10

Mistral:latest failed both of the benchmark's central missing-value defaults.

It claimed that `mean()` ignores missing values and suggested that `cor()` automatically drops incomplete pairs.

Both are false under the defaults used in the original script.

It also never traced the original program to its exact surviving rows or output.

Instead of preserving the task, it changed the centering from a group-specific mean to an overall mean, added an unrelated filter on non-missing `y`, and altered the meaning of the requested count.

The proposed replacement code introduced additional problems:

- the computed group means were never used,
- `length()` was applied to a data frame where `nrow()` was needed,
- an undefined `covar_pairs()` function was called,
- and the final `rbind()` combined incompatible structures.

### Verdict

The weakest response in the current benchmark: core R semantics, task fidelity, and replacement-code correctness all failed.

---

## 9.11 Qwen3.6 and Qwen3.6:35B

### Score: Not scored

Both Qwen3.6 variants failed to produce a substantive final response to the benchmark prompt.

Instead, the returned output consisted only of reasoning or thought-process text, without a completed answer addressing the requested execution trace, problems, corrected code, exact output, and statistical interpretation.

Because there was no final benchmark response to evaluate, these models were not assigned a numerical correctness score.

This is qualitatively different from an incorrect answer. The other tested models attempted to answer the benchmark and could therefore be evaluated against the reference solution. In these two cases, benchmark completion itself failed.

### Verdict

Not scored because no substantive final benchmark answer was produced.

---

# 10. Error Matrix

| Model | `mean()` NA default | Grouped `mutate()` | A disappears after filter | `cor()` NA default | Valid syntax recognition | Exact output |
|---|---|---|---|---|---|---|
| **Codex** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Nemotron-3-Super:cloud** | ✅ | ✅ | ❌ | ❌ / mixed | ✅ | ❌ original |
| **Gemma4:latest (9.6 GB)** | ✅ | ✅ | ❌ | ❌ / mixed | ✅ | ❌ original |
| **Nemotron-3.5-Lightning:latest (25 GB)** | ✅ | ✅ | ❌ | ✅ / mixed | ✅ | ❌ / contradictory |
| **GPT-OSS 20B (13 GB)** | ✅ | ✅ | ✅ | ❌ | ✅ | Mostly |
| **Qwen3-Coder:30B (18 GB)** | ❌ | ❌ | ❌ | ❌ / mixed | ✅ | ❌ original |
| **DeepSeek-Coder (779 MB)** | ❌ | Partial | ❌ | Weak | Mostly | ❌ |
| **Gemma 4E4B (8 GB)** | ❌ | ✅ | ❌ | ❌ | ✅ | ❌ |
| **Qwen2.5-Coder (4.7 GB)** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Mistral:latest (4.4 GB)** | ❌ | Partial | Partial | ❌ | ❌ corrected code | ❌ |

---

# 11. Most Diagnostic Test Cases

Several tiny expressions turned out to be especially useful discriminators.

## Test 1: Missing values in `mean()`

```r
mean(c(1, 2, NA))
```

Correct result:

```text
NA
```

Models that answered `1.5` revealed a basic misunderstanding of R's missing-value defaults.

---

## Test 2: Grouped mutation

After:

```r
group_by(group)
```

the expression:

```r
mean_x = mean(x)
```

is evaluated separately within each group.

A model that interprets this as the global mean does not understand `dplyr` grouping semantics.

---

## Test 3: `filter()` with missing predicates

```r
NA > 0
```

returns:

```text
NA
```

not `FALSE`.

However:

```r
filter(...)
```

retains rows only where the condition is `TRUE`, so rows with `NA` predicates are removed.

This distinction tests precision.

---

## Test 4: Empty groups after filtering

If every row belonging to A is removed, the default pipeline does not return:

```text
A   0
```

The group disappears.

This was one of the most useful traps in the benchmark.

---

## Test 5: Missing values in `cor()`

The default:

```r
cor(x, y)
```

does not automatically mean:

```r
use = "complete.obs"
```

or:

```r
use = "pairwise.complete.obs"
```

Complete-pair handling must be explicitly requested.

---

## Test 6: One-observation correlation

A single pair:

```r
cor(6, 12)
```

does not produce a meaningful correlation.

The correct result is:

```text
NA
```

---

# 12. Broader Findings

## 12.1 Fluent explanations are not the same as correct execution

Gemma 4E4B is a particularly clear example.

Its answer was detailed, organized, and statistically sophisticated in tone. Nevertheless, it failed on several basic R semantics.

This suggests that presentation quality should not be used as a proxy for code-reasoning quality.

---

## 12.2 Some models can repair code they cannot correctly explain

Qwen2.5-Coder and Qwen3-Coder:30B are interesting examples.

Their explanations of the original program were weak relative to parts of their corrected code, which were more plausible.

This indicates that code generation and code execution reasoning are related but separable abilities.

---

## 12.3 Models often try to redesign the analysis

Gemma 4E4B and GPT-OSS 20B both moved toward calculating correlation before filtering or across a different sample, while Mistral:latest changed the centering target itself.

This may reflect useful statistical instincts, but it is problematic in a benchmark that asks for a specific sequence of operations.

A strong model should distinguish:

1. what the requested code does,
2. how to implement the requested correction,
3. whether an alternative analysis might be preferable.

It should not silently replace step 2 with step 3.

---

## 12.4 Exact output prediction is highly diagnostic

Models can often talk convincingly about code while avoiding the hardest commitment: stating the exact result.

Requiring the exact output forces the model to resolve all of the following:

- grouping,
- missing values,
- filtering,
- disappearance of groups,
- sample size,
- and statistical function behavior.

This makes exact-output questions unusually effective for code-model evaluation.

---

## 12.5 Model size does not guarantee stronger reasoning

The addition of local model sizes makes it possible to compare reasoning quality with the approximate storage footprint of the tested models.

The benchmark does not show a simple relationship between model size and performance.

For example, Nemotron-3.5-Lightning:latest had an approximate Ollama size of 25 GB and scored 6.0/10, while the much smaller Gemma4:latest at approximately 9.6 GB received the same score.

Likewise, model size alone cannot explain the ranking among the lower-scoring models.

These comparisons should be treated cautiously because Ollama file size is affected by factors such as quantization and architecture. File size is therefore not equivalent to parameter count, computational complexity, or runtime memory use.

Nevertheless, including the approximate local model size provides useful practical context for evaluating the trade-off between local resource requirements and observed reasoning quality.

---

# 13. Overall Ranking

```text
1. Codex                                   9.5 / 10
2. Nemotron-3-Super:cloud                  6.5 / 10
3. Gemma4:latest (9.6 GB)                  6.0 / 10
4. Nemotron-3.5-Lightning:latest (25 GB)   6.0 / 10
5. GPT-OSS 20B (13 GB)                     5.0 / 10
6. Qwen3-Coder:30B (18 GB)                 3.5 / 10
7. DeepSeek-Coder (779 MB)                 3.0 / 10
8. Gemma 4E4B (8 GB)                       2.5 / 10
9. Qwen2.5-Coder (4.7 GB)                  2.0 / 10
10. Mistral:latest (4.4 GB)                1.5 / 10
```

The gap between Codex and the rest remains substantial.

Codex was the only tested model that consistently combined:

- correct R semantics,
- exact execution tracing,
- correct missing-value behavior,
- correct grouping behavior,
- valid corrected code,
- exact expected output,
- and careful statistical interpretation.

Nemotron-3-Super:cloud was the closest competitor but still failed an important `dplyr` edge case.

Gemma4:latest and Nemotron-3.5-Lightning:latest formed the next tier: both showed solid R knowledge but missed the disappearing-group behavior that the benchmark was designed to expose.

---

# 14. Conclusion

This small benchmark demonstrates that apparently simple R code can expose large differences in AI coding-model quality.

The main challenge was not writing R syntax. Nearly every model could produce plausible-looking R code.

The harder task was to answer:

> What exactly will R do?

That required precise knowledge of language defaults, `dplyr` grouping behavior, missing-value propagation, filtering semantics, and statistical functions.

The results suggest that future code-model benchmarks should include more tasks where models must:

- mentally execute short programs,
- predict exact output,
- distinguish warnings from errors,
- reason about missing values,
- identify package-specific semantics,
- and separate implementation correctness from methodological advice.

These tasks are compact, cheap to evaluate, and surprisingly effective at distinguishing models that merely generate plausible code from models that actually reason about it.

The comparison also suggests that larger local model files do not automatically imply stronger reasoning performance. This makes resource requirements an additional dimension worth considering when evaluating models for local use.

---

# 15. Reproducibility and Test Environment

This comparison is based on individual model responses to the same conceptual benchmark prompt.

Scores are qualitative and were assigned by manually checking the responses against the expected behavior of R, base statistical functions, and `dplyr`.

## 15.1 Hardware and Operating Systems

The tests were conducted on two separate laptops:

| System | Operating System | System RAM |
|---|---|---:|
| Laptop 1 | Windows 11 | 32 GB |
| Laptop 2 | Debian Linux | 64 GB |

The use of two machines made it possible to run models with different local resource requirements.

The benchmark scores themselves are based on response correctness rather than execution speed or hardware performance.

Hardware differences are therefore relevant primarily for whether a model could be hosted and executed locally, rather than for determining its score.

---

## 15.2 Ollama and Odysseus Setup

The locally hosted models were downloaded and managed using **Ollama**.

The models were then run through **Odysseus**, which served as the interface used to submit the benchmark prompt and collect the model responses.

The general local testing workflow was therefore:

```text
Benchmark prompt
      ↓
   Odysseus
      ↓
    Ollama
      ↓
Local language model
      ↓
 Model response
      ↓
Manual evaluation against
the R reference solution
```

Cloud-hosted models are explicitly identified in their model names where applicable.

For example:

```text
Nemotron-3-Super:cloud
```

was cloud-hosted and should therefore not be interpreted as having the same local hardware requirements as the models downloaded and executed locally.

---

## 15.3 Memory Disabled During Testing

An important part of the experimental setup was that **the memory option in Odysseus was disabled during the benchmark**.

This was done to keep the individual model tests isolated from one another.

In particular, previous benchmark responses were not intentionally retained through Odysseus memory and supplied as contextual information to models tested afterward.

This reduced the risk of **cross-model response contamination**.

Without this precaution, a model tested later could potentially receive information derived from an earlier model's response through the interface's memory system. That would make the benchmark less useful as a comparison of independently generated answers.

Each model was therefore tested with the same benchmark prompt while the Odysseus memory function was disabled.

The intended evaluation procedure was:

```text
Identical benchmark prompt
          ↓
       Odysseus
   (memory disabled)
          ↓
        Ollama
          ↓
 Individual language model
          ↓
    Independent response
          ↓
 Manual evaluation against
  the R reference solution
```

It is important to distinguish this from model training.

Disabling Odysseus memory does not imply that the underlying model weights would otherwise be updated or that a model would literally learn from another response during ordinary inference.

Rather, disabling memory was intended to ensure that **previous benchmark answers were not automatically supplied as additional context to subsequent model sessions**.

This makes the comparison closer to an independent-response design.

---

## 15.4 Approximate Local Model Sizes

For the locally tested models where installation size was recorded:

| Model | Approximate Ollama Size |
|---|---:|
| Nemotron-3.5-Lightning:latest | 25 GB |
| Qwen3-Coder:30B | 18 GB |
| GPT-OSS 20B | 13 GB |
| Gemma4:latest | 9.6 GB |
| Gemma 4E4B | 8 GB |
| Qwen2.5-Coder | 4.7 GB |
| Mistral:latest | 4.4 GB |
| DeepSeek-Coder | 779 MB |

These values represent the approximate local model sizes observed for the Ollama installations used during the benchmark.

They should not be interpreted as direct measures of parameter count or model complexity.

Model storage size can depend on factors including:

- quantization,
- numerical precision,
- model architecture,
- packaging,
- and the particular Ollama model variant used.

The values are nevertheless useful as a practical measure of the amount of local storage required by the tested versions.

---

## 15.5 What Was and Was Not Measured

The benchmark primarily evaluates **answer correctness**.

It does not currently provide systematic measurements of:

- inference speed,
- tokens per second,
- time to first token,
- CPU utilization,
- GPU utilization,
- RAM consumption,
- VRAM consumption,
- energy consumption,
- or response latency.

The hardware information and approximate model sizes should therefore be interpreted as contextual information rather than performance benchmarks.

A 25 GB model receiving a particular score should not, for example, automatically be considered less efficient than a 9.6 GB model with the same score without also measuring inference speed, memory consumption, hardware acceleration, and other runtime characteristics.

---

## 15.6 Scope of the Benchmark

The benchmark is not intended as a comprehensive measure of each model's overall coding ability.

It evaluates a narrow but important capability:

> **Precise reasoning about R code without execution.**

The task focuses specifically on whether a model can correctly reason about:

- R execution semantics,
- grouped `dplyr` operations,
- missing-value propagation,
- filtering,
- summary statistics,
- exact outputs,
- and statistical interpretation.

Performance on this benchmark should therefore not be interpreted as a general ranking of the models for all programming, statistical, or language-model tasks.

For a stronger benchmark, the same methodology could be expanded to include:

- factor levels,
- grouped summaries,
- joins,
- recycling rules,
- `NA` vs. `NaN`,
- `ifelse()` vs. `if_else()`,
- factor coercion,
- `model.matrix()`,
- formula interfaces,
- environments and scoping,
- statistical-model output interpretation,
- inference time,
- memory consumption,
- and response quality relative to model size.

A larger benchmark containing multiple independent R tasks would also reduce the influence of any one unusual language or package-specific edge case and provide a more reliable comparison of model reasoning ability.
