Testing AI Models on R Code Reasoning
A Small Benchmark of R Semantics, Missing Values, `dplyr`, and Statistical Interpretation
Overview
This report compares several AI coding models on a compact R reasoning task. The code itself is simple: a small grouped data frame, a grouped mean, centering, filtering, and a correlation. However, the task is deliberately designed to test whether a model can mentally execute R code correctly, rather than merely recognize familiar R syntax.
The benchmark exposes several common failure modes:
misunderstanding how `group_by()` affects `mutate()`,
incorrectly assuming `mean()` removes missing values by default,
misunderstanding how `filter()` treats `NA`,
failing to recognize that groups can disappear entirely after filtering,
misunderstanding the default missing-value behavior of `cor()`,
confusing valid R code with syntax errors,
changing the requested statistical estimand instead of implementing it,
and failing to distinguish programming behavior from statistical interpretation.
The tested models were:
Codex
Nemotron 3 Super
GPT-OSS 20B
DeepSeek-Coder
Gemma 4E4B
Qwen2.5-Coder
---
1. Benchmark Prompt
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
Explain exactly what happens when this code is executed.
Identify every problem in the code, including problems that do not generate an explicit R error.
Explain why each problem occurs.
Rewrite the code so that it:
computes the mean of `x` separately within each group while ignoring missing values,
centers `x` around its group mean,
keeps observations with a positive centered value,
reports the number of remaining observations per group,
calculates the correlation between `x` and `y` using all available complete pairs.
State the exact output you expect from the corrected code.
Explain whether calculating the correlation after filtering has a different interpretation from calculating the correlation before filtering.
Do not simply provide corrected code. Explain the behavior of the original code step by step and distinguish between:
syntax errors,
missing-value issues,
statistical/logical issues.
---
2. Why This Is a Useful Benchmark
At first glance, this looks like a basic `dplyr` exercise. In reality, it tests several layers of R knowledge at once.
A weak model can often produce something that sounds like an R explanation while missing the actual execution semantics. For example, it may know that `na.rm = TRUE` exists, but still incorrectly claim that `mean()` ignores `NA` by default.
A strong model needs to reason through the exact state of the data after every pipeline operation.
The task therefore tests both:
language-specific programming knowledge, and
statistical reasoning about the meaning of the result.
---
3. Reference Solution
3.1 Original Code: Exact Execution
The code contains no syntax error.
Because the data are grouped using:
```r
group_by(group)
```
the following calculation is performed separately for groups A and B:
```r
mean_x = mean(x)
```
Group A
For group A:
```r
x = c(1, 2, NA)
```
R's `mean()` uses `na.rm = FALSE` by default:
```r
mean(c(1, 2, NA))
# NA
```
Therefore `mean_x = NA` for every row in group A.
The centered values become:
```r
x - mean_x
```
which gives:
```r
c(NA, NA, NA)
```
because arithmetic involving `NA` produces `NA`.
Group B
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
3.2 Effect of `filter()`
The code then evaluates:
```r
filter(x_centered > 0)
```
For group A:
```r
NA > 0
# NA
```
`dplyr::filter()` keeps rows only where the condition evaluates to `TRUE`. Therefore all rows in group A are removed.
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
Importantly, group A disappears entirely. `summarise()` does not automatically produce an `A, n = 0` row.
3.3 Original Output
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
cannot estimate a Pearson correlation from a single observation. The resulting value is `NA`.
The original code therefore produces essentially:
```text
# A tibble: 1 × 3
  group     n correlation
  <fct> <int>       <dbl>
1 B         1          NA
```
---
4. Corrected Code
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
5. Corrected Output
Group A
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
Only the row with `x = 2, y = 4` survives. Thus `n = 1` and `correlation = NA`.
Group B
The group mean is `5` and the centered values are `-1, 0, 1`. Only `x = 6, y = 12` survives. Thus `n = 1` and `correlation = NA`.
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
6. Statistical Interpretation
Calculating the correlation before and after filtering answers different questions.
Before filtering, the within-group correlation is based on all available complete pairs.
For group A, `(1, 2)` and `(2, 4)` give `r = 1`.
For group B, `(4, 8)` and `(6, 12)` also give `r = 1`.
After filtering on:
```r
x_centered > 0
```
the correlation is conditional on observations having `x` above their group-specific mean. The estimand is therefore different.
The post-filter correlation should not automatically be described as "wrong" or "biased." Whether it is appropriate depends on the research question. It is simply a correlation for a selected subpopulation rather than the full group.
---
7. Scoring Rubric
Criterion	What was tested
R execution tracing	Does the model correctly simulate the code without running it?
`group_by()` semantics	Does it understand grouped `mutate()` and grouped `summarise()`?
`mean()` + `NA`	Does it know that `mean()` does not ignore `NA` by default?
`filter()` semantics	Does it understand `NA > 0` and row removal?
Empty-group behavior	Does it know that a group can disappear completely after filtering?
`cor()` semantics	Does it understand the default missing-value behavior and one-observation result?
Corrected code	Does the proposed correction actually satisfy the prompt?
Exact output	Does it calculate the final result rather than merely describe it?
Statistical interpretation	Does it distinguish the full-sample and filtered correlation correctly?
Task fidelity	Does it solve the requested problem rather than silently changing the estimand?
Scores are approximate and intended as comparative assessments rather than formal psychometric measurements.
---
8. Results
Rank	Model	Score	Overall assessment
1	Codex	9.5/10	Excellent R execution reasoning with only minor overengineering
2	Nemotron 3 Super	6.5/10	Strong understanding, but important mistakes in empty-group and `cor()` semantics
3	GPT-OSS 20B	5/10	Good execution tracing, weaker API semantics and task fidelity
4	DeepSeek-Coder	3/10	Recognized broad issues but did not actually execute the code correctly
5	Gemma 4E4B	2.5/10	Polished explanation but several basic R errors and changed the requested analysis
6	Qwen2.5-Coder	2/10	Major misunderstandings of valid R syntax, grouping, missing values, and object creation
---
9. Model-by-Model Evaluation
9.1 Codex
Score: 9.5/10
Codex produced by far the strongest response.
It correctly identified `mean(c(1, 2, NA))` as `NA` and followed the consequences through the entire pipeline. It correctly concluded that all rows from group A disappear, only `B: x = 6, y = 12` remains, the original result contains only group B, and the original correlation is `NA`.
It also correctly predicted the corrected result: A and B each have `n = 1`, with both correlations remaining `NA`.
A particularly strong observation was that `n()` is not necessarily the same as the effective number of complete observations used by `cor(x, y, use = "complete.obs")`. A retained row may count toward `n()` while being excluded from the correlation if one variable is missing.
Weakness
Its corrected code included defensive checks for minimum sample size and distinct values. This is robust production code but somewhat more complicated than necessary for this benchmark.
Verdict
Codex demonstrated genuine R-specific reasoning rather than merely recognizing syntax patterns.
---
9.2 Nemotron 3 Super
Score: 6.5/10
Nemotron correctly understood the most important missing-value behavior and correctly derived group A's centered values as all `NA` and group B's as `-1, 0, 1`.
However, it then made a major execution error by claiming that the original result would contain:
```text
A   0   NA
B   1   NA
```
This is wrong. After `filter()` removes every A observation, group A no longer appears in the grouped data. `summarise()` does not automatically recreate the empty group.
Nemotron also made smaller errors: it described `NA` in a filter condition as yielding `FALSE` rather than remaining `NA`, misstated the default missing-value behavior of `cor()`, and described the standard deviation of a one-element vector as zero rather than `NA`.
Its corrected code was nevertheless largely valid and its statistical interpretation was generally good.
Verdict
A meaningful step up from weaker models, but still unreliable on subtle execution semantics.
---
9.3 GPT-OSS 20B
Score: 5/10
GPT-OSS 20B did a good job tracing the original pipeline. It correctly recognized grouped computation, `mean(x)` returning `NA` in group A, propagation of `NA` through `x_centered`, disappearance of group A, survival of only the final B observation, and an undefined correlation for that one remaining observation.
However, it incorrectly claimed that `cor()` automatically excludes missing values by default. In R, the default is effectively `use = "everything"`, so missing values generally propagate to `NA`.
It also made an inaccurate claim about `summarise()` retaining grouping. With one grouping variable and standard modern `dplyr` behavior, the final grouping level is typically dropped.
The largest conceptual problem was that it changed the requested analysis. Instead of calculating the correlation per group after filtering, it proposed an overall correlation on the remaining data and repeated that value for each group.
Verdict
Good mental execution of the original code, but weaker knowledge of API details and a tendency to redesign the analysis instead of following the prompt.
---
9.4 DeepSeek-Coder
Score: 3/10
DeepSeek-Coder identified some relevant themes: missing values, centering, filtering, and correlation. However, it failed to reason through the actual execution.
Most importantly, it stated that the code computes the mean of `x` within each group while ignoring missing values. That is false. The original code uses `mean(x)`, not `mean(x, na.rm = TRUE)`.
This mistake prevents it from understanding the central failure of group A.
The response also failed to derive the actual surviving rows, failed to provide the requested exact output, described valid operations as incorrect without explaining the actual semantics, and provided no real corrected code despite having a "corrected code" section.
Verdict
The response recognized relevant R vocabulary but did not convincingly execute the program.
---
9.5 Gemma 4E4B
Score: 2.5/10
Gemma's answer was polished and detailed, but it contained several fundamental R errors.
The most important was the claim that `mean()` automatically ignores `NA` values. This is false, so it incorrectly calculated the original group A mean as `1.5` instead of `NA`.
Its proposed corrected code also included:
```r
cor(x, y, use = "pairwise.complete.obs", na.rm = TRUE)
```
but `cor()` does not have an `na.rm` argument, so this code would error.
Gemma also proposed `sum(x_centered > 0)` without `na.rm = TRUE`. For group A, `sum(c(FALSE, TRUE, NA))` returns `NA`, not `1`.
Finally, Gemma changed the requested statistical task by moving the correlation calculation before filtering and then argued that this was the "correct statistical approach."
Verdict
A good example of how fluent statistical prose can conceal weak language-specific execution reasoning.
---
9.6 Qwen2.5-Coder
Score: 2/10
Qwen2.5-Coder performed worst overall.
It incorrectly stated that `mean_x = mean(x)` calculates the overall mean across all groups, ignoring the effect of `group_by(group)`.
It also claimed that `filter(x_centered > 0)` was syntactically incorrect because `x_centered` was undefined, even though `x_centered` had been created immediately beforehand in `mutate()`.
The model further claimed that `result` was never defined, despite the code clearly assigning `result <- df %>% ...`.
It also repeated the false claim that `mean()` excludes missing values by default.
Surprisingly, its rewritten code was mostly valid: `mean(x, na.rm = TRUE)` and `cor(x, y, use = "complete.obs")` were both appropriate corrections. However, it failed to provide the exact requested output.
Verdict
The model showed some ability to produce plausible corrected R code, but its explanation of the original program was unreliable at a very basic level.
---
10. Error Matrix
Model	`mean()` NA default	Grouped `mutate()`	A disappears after filter	`cor()` NA default	Valid syntax recognition	Exact output
Codex	✅	✅	✅	✅	✅	✅
Nemotron 3 Super	✅	✅	❌	❌ / mixed	✅	❌ original
GPT-OSS 20B	✅	✅	✅	❌	✅	Mostly
DeepSeek-Coder	❌	Partial	❌	Weak	Mostly	❌
Gemma 4E4B	❌	✅	❌	❌	✅	❌
Qwen2.5-Coder	❌	❌	❌	❌	❌	❌
---
11. Most Diagnostic Test Cases
Test 1: Missing values in `mean()`
```r
mean(c(1, 2, NA))
```
Correct result:
```text
NA
```
Models that answered `1.5` revealed a basic misunderstanding of R's missing-value defaults.
Test 2: Grouped mutation
After `group_by(group)`, the expression `mean_x = mean(x)` is evaluated separately within each group. A model that interprets this as the global mean does not understand `dplyr` grouping semantics.
Test 3: `filter()` with missing predicates
```r
NA > 0
```
returns `NA`, not `FALSE`. However, `filter()` retains rows only where the condition is `TRUE`, so rows with `NA` predicates are removed.
Test 4: Empty groups after filtering
If every row belonging to A is removed, the default pipeline does not return `A   0`. The group disappears.
Test 5: Missing values in `cor()`
The default `cor(x, y)` does not automatically mean `use = "complete.obs"` or `use = "pairwise.complete.obs"`. Complete-pair handling must be explicitly requested.
Test 6: One-observation correlation
A single pair such as `cor(6, 12)` does not produce a meaningful Pearson correlation. The result is `NA`.
---
12. Broader Findings
12.1 Fluent explanations are not the same as correct execution
Gemma is the clearest example. Its answer was detailed, organized, and statistically sophisticated in tone. Nevertheless, it failed on several basic R semantics.
Presentation quality should not be used as a proxy for code-reasoning quality.
12.2 Some models can repair code they cannot correctly explain
Qwen2.5-Coder is an interesting example. Its explanation of the original program was very weak, but parts of its corrected code were reasonable.
This indicates that code generation and code execution reasoning are related but separable abilities.
12.3 Models often try to redesign the analysis
Gemma and GPT-OSS 20B both moved toward calculating correlation before filtering or across a different sample.
This may reflect useful statistical instincts, but it is problematic in a benchmark that asks for a specific sequence of operations.
A strong model should distinguish:
what the requested code does,
how to implement the requested correction,
whether an alternative analysis might be preferable.
It should not silently replace step 2 with step 3.
12.4 Exact output prediction is highly diagnostic
Models can often talk convincingly about code while avoiding the hardest commitment: stating the exact result.
Requiring exact output forces the model to resolve grouping, missing values, filtering, disappearance of groups, sample size, and statistical function behavior.
This makes exact-output questions unusually effective for code-model evaluation.
---
13. Overall Ranking
```text
1. Codex             9.5 / 10
2. Nemotron 3 Super  6.5 / 10
3. GPT-OSS 20B       5.0 / 10
4. DeepSeek-Coder    3.0 / 10
5. Gemma 4E4B        2.5 / 10
6. Qwen2.5-Coder     2.0 / 10
```
The gap between Codex and the rest was substantial.
Codex was the only tested model that consistently combined correct R semantics, exact execution tracing, correct missing-value behavior, correct grouping behavior, valid corrected code, exact expected output, and careful statistical interpretation.
Nemotron 3 Super was the closest competitor but still failed an important `dplyr` edge case.
---
14. Conclusion
This small benchmark demonstrates that apparently simple R code can expose large differences in AI coding-model quality.
The main challenge was not writing R syntax. Nearly every model could produce plausible-looking R code.
The harder task was to answer:
> What exactly will R do?
That required precise knowledge of language defaults, `dplyr` grouping behavior, missing-value propagation, filtering semantics, and statistical functions.
The results suggest that future code-model benchmarks should include more tasks where models must:
mentally execute short programs,
predict exact output,
distinguish warnings from errors,
reason about missing values,
identify package-specific semantics,
and separate implementation correctness from methodological advice.
These tasks are compact, cheap to evaluate, and surprisingly effective at distinguishing models that merely generate plausible code from models that actually reason about it.
---
15. Reproducibility Note
This comparison is based on individual model responses to the same conceptual benchmark prompt. Scores are qualitative and were assigned by manually checking the responses against expected R and `dplyr` behavior.
The benchmark is not intended as a comprehensive measure of each model's overall coding ability. It evaluates a narrow but important capability:
> **precise reasoning about R code without execution.**
For a stronger benchmark, the same methodology could be expanded to include:
factor levels,
grouped summaries,
joins,
recycling rules,
`NA` vs `NaN`,
`ifelse()` vs `if_else()`,
factor coercion,
`model.matrix()`,
formula interfaces,
environments and scoping,
and statistical-model output interpretation.
