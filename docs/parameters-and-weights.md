# Parameters and Weights

## 1. Purpose

This document describes the parameters used by the Class Allocation Tool and the weighted criteria used to evaluate candidate allocations.

The system is a multi-criteria decision-support tool. It does not define a single universally correct allocation. Instead, it produces a candidate arrangement according to priorities expressed through configurable weights.

> This document uses public terminology. The original production source code and school data are proprietary.

## 2. Terminology

| Public term | Meaning in the allocation model |
|---|---|
| Class Types | `MAINSTREAM`, `EAL` and `T` / Transition classes |
| Academic Scores | English, French and Mathematics scores |
| Individual Plans | Learning-support or individual-plan indicators, including EBEP-related data |
| School Assistant | Family-based or school-based support categories |
| English Plans | FLESCO-related information |
| `WITH` | Requested students that a student would preferably join |
| `WITHOUT` | Students or classes that should preferably be kept apart |
| Behaviour | Classroom-behaviour score |
| Previous Class | The student's previous or current recorded class |

## 3. User-Configurable Parameters

### 3.1. Cohort name

The user provides a cohort name, such as `CE2` or `CM1`. The name is used to:

- identify the input workbook;
- generate class labels;
- name the output workbook;
- associate the allocation with the displayed academic year.

### 3.2. Number of classes

The user specifies the number of classes to create. The web interface accepts values between 1 and 9.

The allocation engine computes target class sizes using integer division:

\[
N = qK + r, \qquad 0 \leq r < K
\]

where:

- \(N\) is the number of students;
- \(K\) is the number of classes;
- \(q\) is the base class size;
- \(r\) is the number of classes receiving one additional student.

The first \(r\) classes receive \(q+1\) students and the remaining classes receive \(q\) students. This ensures that all classes differ in size by at most one student.

### 3.3. Class Types

Each class can be configured through two interface flags:

- EAL class.
- Transition class.

The resulting internal Class Types are:

| Configuration | Internal Class Type |
|---|---|
| Neither flag selected | `MAINSTREAM` |
| EAL selected | `EAL` |
| Transition selected | `T` |
| EAL and Transition selected in the interface | represented as a Transition-compatible class configuration |

The current backend stores one effective Class Type per class. A Transition class is represented by `T` in the allocation engine.

### 3.4. Number of attempts

The user specifies the maximum number of independent allocation attempts. The web interface accepts values between 1 and 99, with a default of 10.

Each attempt contains:

1. A randomised student ordering.
2. A greedy initial allocation.
3. A local search based on individual moves.
4. A local search based on pairwise swaps.
5. A global score evaluation.

The best valid result across all attempts is retained.

### 3.5. Weight mode

The user can select:

- `Recommended`: the default weight set defined by the application.
- `Custom`: the latest values saved through the weight-configuration page.

Custom weights are persisted to a JSON file and loaded again in future sessions.

## 4. Input Data Parameters

The Excel input is converted into an internal student representation. The relevant parameter groups are described below.

### 4.1. Identity and previous class

- Student name.
- Optional first name.
- Gender.
- Previous class.
- Current class, when available.

Previous-class information is used to calculate the dominance of a previous group inside a new class.

### 4.2. Academic Scores

The following Academic Scores are read and normalised:

- English.
- French.
- Mathematics.

Scores are expected to be between 1 and 5. Missing values are accepted and replaced by the neutral default value 3 when the optimisation engine calculates means and cohort targets.

### 4.3. Behaviour

Behaviour is represented by a score, normally between 1 and 5. The model also separately tracks students with Behaviour scores of 4 and 5 because these categories have dedicated dispersion criteria.

### 4.4. Individual Plans

A student is considered to have an Individual Plan when either:

- the relevant support flag is explicitly positive; or
- the individual-plan field is non-empty.

The criterion is balanced between classes as a proportion of the class population.

### 4.5. School Assistant

The model distinguishes two School Assistant categories:

- Family-based support.
- School-based support.

Family-based support is balanced proportionally. School-based support uses a non-linear cost because the model prefers certain groupings, discourages excessive concentration and separately penalises mixing the two support categories in the same class.

### 4.6. English Plans

English Plans identify students who should preferably be placed in a compatible English-support Class Type.

An English Plan contributes a preference:

- placement in `EAL` or `T` is rewarded relative to `MAINSTREAM`;
- placement in `MAINSTREAM` receives a penalty.

### 4.7. `WITH` preferences

The input supports multiple `WITH` columns. Names are normalised and resolved against the cohort index.

The implementation:

- accepts multiple requested students;
- ignores empty references;
- removes unresolved references;
- removes self-references;
- removes duplicates;
- keeps at most four resolved requests per student.

The optimisation score penalises a student when none of their resolved `WITH` preferences is placed in the same class.

### 4.8. `WITHOUT` constraints

The input supports multiple `WITHOUT` columns. These references are resolved against the cohort index and then symmetrised.

If student A lists student B as `WITHOUT`, the model also treats B as incompatible with A. This prevents one-sided input from producing an inconsistent evaluation.

A pair of incompatible students assigned to the same class receives a penalty.

### 4.9. Forbidden class

A student may contain an explicit forbidden-class reference. Class names are normalised and resolved against the configured class list.

A resolved forbidden class is treated as a hard placement restriction during initial allocation and local-search operations. The final integrity check rejects a solution that violates this rule.

### 4.10. New-student status

The current allocation engine identifies a student as new when a valid arrival date exists and is later than the execution date.

The number of new students is balanced between classes using a squared-deviation cost.

## 5. Recommended Weights

The current recommended configuration is:

| Criterion | Recommended weight | Public interpretation |
|---|---:|---|
| Behaviour | 2.0 | Balance mean Behaviour scores |
| English Academic Score | 0.5 | Balance mean English scores |
| French Academic Score | 0.5 | Balance mean French scores |
| Mathematics Academic Score | 0.5 | Balance mean Mathematics scores |
| Gender mix | 2.0 | Balance the proportion of boys and girls |
| Individual Plans | 4.0 | Balance the proportion of students with Individual Plans |
| New students | 4.0 | Balance new students between classes |
| Family-based School Assistant | 2.5 | Balance family-based support |
| School-based School Assistant | 2.5 | Control school-based support distribution |
| No `WITH` satisfied | 8.0 | Penalise students with no requested student in their class |
| English Plans in compatible Class Types | 2.0 | Prefer compatible English-support placement |
| EAL distribution | 2.0 | Balance EAL students across EAL classes |
| Behaviour level 5 dispersion | 2.0 | Distribute Behaviour level 5 students |
| Behaviour level 4 dispersion | 1.0 | Distribute Behaviour level 4 students |
| Forbidden class violation | 15.0 | Strongly penalise a forbidden placement |
| Class Type incompatibility | 15.0 | Penalise unsuitable Class Type placement |
| `WITHOUT` violation | 12.0 | Penalise incompatible students placed together |
| Strong gender imbalance | 8.0 | Penalise classes below the gender-balance threshold |
| Previous Class dominance | 6.0 | Penalise excessive preservation of a previous group |

The weights are not probabilities and do not represent percentages. They are relative coefficients in the global cost function.

## 6. Cohort Targets

Before allocation scoring, the engine calculates cohort-level targets.

For numerical Academic Scores and Behaviour:

\[
\bar{x}_{cohort} = \frac{1}{N}\sum_{i=1}^{N} x_i
\]

For proportions:

\[
p_{cohort} = \frac{\text{number of students with the attribute}}{N}
\]

Examples include:

- overall Behaviour mean;
- overall English, French and Mathematics means;
- overall gender proportion;
- Individual Plans proportion;
- family-based School Assistant proportion;
- school-based School Assistant counts;
- target number of EAL students per EAL class;
- target number of new students per class;
- target number of Behaviour level 4 and 5 students per class.

## 7. Weight Semantics

A larger weight increases the influence of a criterion in the cost function. It does not guarantee that the criterion will always be satisfied.

For example:

- A high `WITHOUT` weight strongly discourages incompatible students from being placed together.
- A moderate Academic Score weight encourages balance but allows other constraints to take priority.
- A zero weight disables the criterion's contribution to the objective, except where a separate hard validation still applies.

The allocation remains a trade-off between criteria. Some preferences may be mutually incompatible, especially when the cohort is small or heavily constrained.

## 8. Cost Components

The global score is the sum of criterion-specific components:

\[
C = C_{academic} + C_{behaviour} + C_{support} + C_{relationships} + C_{class\ type} + C_{continuity}
\]

The implementation calculates the following detailed components:

- Behaviour balance.
- English Academic Score balance.
- French Academic Score balance.
- Mathematics Academic Score balance.
- Gender-mix balance.
- Individual Plans balance.
- Family-based School Assistant balance.
- School-based School Assistant distribution.
- Mixed School Assistant penalty.
- Unsatisfied `WITH` requests.
- English Plans preference.
- EAL distribution.
- New-student distribution.
- Behaviour level 4 dispersion.
- Behaviour level 5 dispersion.
- Forbidden class penalty.
- Class Type compatibility penalty.
- `WITHOUT` penalty.
- Strong gender imbalance penalty.
- Previous Class dominance penalty.

## 9. Formula Patterns

### 9.1. Mean-based balance

For a class-level mean \(m_k\), cohort target \(m^*\), and weight \(w\):

\[
C_k = w(m_k - m^*)^2
\]

This pattern is used for Behaviour and Academic Scores.

### 9.2. Proportion-based balance

For a class-level proportion \(p_k\), cohort target \(p^*\), and weight \(w\):

\[
C_k = w(p_k - p^*)^2
\]

This pattern is used for gender mix and Individual Plans.

### 9.3. Count-based dispersion

For a class count \(n_k\), target count \(n^*\), and weight \(w\):

\[
C_k = w(n_k - n^*)^2
\]

This pattern is used for EAL students, new students and Behaviour levels 4 and 5.

### 9.4. Relationship penalties

A student with at least one resolved `WITH` request incurs the `No WITH satisfied` penalty when none of the requested students shares the same class.

A `WITHOUT` pair incurs its configured penalty once when both students are assigned to the same class. Symmetric duplicates are de-duplicated during scoring.

### 9.5. Strong gender imbalance

A class is penalised when either gender proportion is below 35%. The penalty increases with the distance below this threshold.

### 9.6. Previous Class dominance

A class is penalised when more than 60% of its students come from the same previous class. The penalty increases with the excess above this threshold.

## 10. Configuration Persistence

Recommended weights are defined in the allocation module. Custom weights are stored as JSON by the web application.

The user interface presents:

- basic criteria first;
- advanced criteria behind an expandable section;
- recommended values for comparison;
- the current value for each criterion;
- sliders ranging from 0 to 20 in increments of 0.5.

The current implementation accepts non-negative values through the interactive configuration range. Custom values are reused in future runs until replaced.

## 11. Interpretation of the Final Score

The score is an internal optimisation objective. A lower score indicates a better result according to the selected weights and the implemented criteria.

Scores should not be compared across different cohorts or weight configurations as if they were universal quality ratings. They are meaningful primarily when comparing candidate allocations produced for the same cohort and configuration.

The result workbook should therefore be reviewed through both:

- the global score and detailed statistics;
- professional judgement from authorised school staff.

## 12. Adaptation to Other Schools

The parameter model is extensible. A deployment for another school may require changes to:

- Excel column mapping;
- score scales;
- Class Type definitions;
- English Plans rules;
- Individual Plans categories;
- School Assistant categories;
- `WITH` and `WITHOUT` input conventions;
- forbidden-class syntax;
- recommended weights;
- output workbook format;
- local safeguarding and access requirements.

New criteria can be implemented by adding:

1. an input field or derived attribute;
2. a normalisation rule;
3. a class-level statistic or constraint evaluation;
4. a configurable weight;
5. a cost component;
6. an output statistic where useful;
7. validation using synthetic or authorised test data.

This makes the application suitable as a configurable foundation rather than a fixed one-school-only rule set.
