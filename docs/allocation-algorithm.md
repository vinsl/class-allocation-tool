# Allocation Algorithm

## 1. Purpose and Design Principles

The Class Allocation Tool uses a stochastic, weighted, multi-criteria local-search algorithm to generate candidate class allocations.

The algorithm is designed to balance several objectives simultaneously:

- preserve Class Type compatibility;
- respect explicit forbidden-class constraints;
- maintain balanced class sizes;
- balance Academic Scores and Behaviour;
- distribute support-related attributes;
- account for English Plans;
- satisfy as many `WITH` preferences as possible;
- separate `WITHOUT` pairs where possible;
- avoid excessive continuity from the previous class structure;
- provide a reproducible operational workflow for school review.

The algorithm is a decision-support method. It ranks candidate allocations according to configurable priorities; it does not make an autonomous educational decision.

> This document describes the implemented algorithm at a technical level without disclosing proprietary student data or production deployment details.

## 2. Terminology

| Public terminology | Algorithmic meaning |
|---|---|
| Class Types | `MAINSTREAM`, `EAL` and `T` / Transition |
| Academic Scores | English, French and Mathematics scores |
| Individual Plans | Learning-support and individual-plan indicators |
| School Assistant | Family-based and school-based support categories |
| English Plans | FLESCO-related placement preference |
| `WITH` | Soft relational preference for students to share a class |
| `WITHOUT` | Soft relational constraint discouraging students from sharing a class |
| Candidate allocation | One complete assignment of all input students to classes |
| Cost / score | Scalar objective value; lower is better |

## 3. Mathematical Formulation

Let:

- \(S\) be the set of students;
- \(K\) be the set of classes;
- \(x_{s,k} \in \{0,1\}\) indicate whether student \(s\) is assigned to class \(k\);
- \(C(x)\) be the global cost of allocation \(x\).

A valid allocation should satisfy:

\[
\sum_{k \in K} x_{s,k} = 1 \qquad \forall s \in S
\]

Each student is assigned exactly once.

For target class capacities \(cap_k\):

\[
\sum_{s \in S} x_{s,k} = cap_k \qquad \forall k \in K
\]

The implementation computes capacities so that all classes contain either \(q\) or \(q+1\) students.

For explicit forbidden-class constraints, if student \(s\) cannot be assigned to class \(k\):

\[
x_{s,k} = 0
\]

The practical implementation enforces this restriction during initial construction, move operations and swap operations, and verifies it before export.

Class Type compatibility is also checked during initial placement and contributes to the global objective when an unsuitable placement is evaluated.

The optimisation objective is a weighted sum:

\[
C(x) = \sum_{j=1}^{m} w_j P_j(x)
\]

where:

- \(P_j(x)\) is the penalty associated with criterion \(j\);
- \(w_j\) is the configurable weight of criterion \(j\);
- lower values of \(C(x)\) are preferred.

## 4. Input Normalisation and Constraint Resolution

Before optimisation, the input workbook is transformed into typed student objects.

The preprocessing stage:

1. Reads the Excel workbook.
2. Normalises column names.
3. Cleans text values.
4. Converts Academic Scores and Behaviour values to the 1–5 scale.
5. Converts yes/no fields to booleans where possible.
6. Converts date fields to Python dates.
7. Extracts `WITH` and `WITHOUT` references.
8. Builds indexes for student and class name resolution.
9. Resolves references against the current cohort and configured classes.

### 4.1. Name normalisation

Names and class labels are converted to robust matching keys by:

- trimming whitespace;
- converting to lowercase;
- removing accents;
- collapsing repeated spaces.

For multi-token names and labels, the implementation also supports reversed token order. Ambiguous references are not treated as valid unique matches.

### 4.2. `WITH` resolution

For every student, the algorithm:

- resolves each `WITH` entry against the cohort index;
- discards unresolved entries;
- discards self-references;
- removes duplicates;
- retains at most four resolved preferences.

### 4.3. `WITHOUT` resolution

`WITHOUT` entries are resolved using the same approach. The resulting relation is symmetrised:

```text
If A lists B as WITHOUT,
then B is also treated as WITHOUT A.
```

Symmetrisation prevents the score from depending on which student happened to record the relationship.

## 5. Class Type Compatibility

The input category of a student determines the classes that are considered compatible.

| Student category | Compatible destination |
|---|---|
| `MAINSTREAM` | Any Class Type |
| `EAL` | `EAL` only |
| `TRANSITION` | `T` if a `T` class exists; otherwise `EAL` |

The function that checks compatibility is used during candidate placement and local-search operations.

A Class Type mismatch also contributes a soft penalty to the global score. This provides a second layer of protection in addition to the initial placement filter and final integrity checks.

## 6. Cohort Targets

The algorithm first calculates cohort-level targets used by the cost function.

### 6.1. Academic and Behaviour means

For each score attribute \(a\), the target is:

\[
\mu_a = \frac{1}{|S|} \sum_{s \in S} a(s)
\]

Missing values are replaced by the neutral default value 3.

The calculated targets include:

- Behaviour mean;
- English Academic Score mean;
- French Academic Score mean;
- Mathematics Academic Score mean.

### 6.2. Proportion targets

The following cohort proportions are computed:

- proportion of boys;
- proportion of students with Individual Plans;
- proportion of students with family-based School Assistant support;
- proportion of students with school-based School Assistant support where applicable.

### 6.3. Count targets

The algorithm derives target counts for:

- EAL students per EAL class;
- new students per class;
- Behaviour level 4 students per class;
- Behaviour level 5 students per class.

## 7. Capacity Calculation

For \(N\) students and \(K\) classes, the algorithm calculates:

\[
q, r = divmod(N,K)
\]

Class capacities are assigned as follows:

- the first \(r\) classes receive \(q+1\) students;
- all remaining classes receive \(q\) students.

This creates balanced target capacities without requiring an additional capacity parameter.

## 8. Student Placement Order

The initial allocation is greedy, but students are not processed in arbitrary input order.

A difficulty score is used to place more constrained students first. The ordering prioritises:

1. Transition students.
2. EAL students.
3. Students with more combined `WITH` and `WITHOUT` relationships.
4. Students with more `WITHOUT` relationships.
5. Students with more `WITH` preferences.
6. Higher Behaviour scores.

The list is shuffled before sorting to introduce controlled stochastic variation between attempts.

This ordering reduces the risk that constrained students are left with no compatible destination near the end of the initial construction.

## 9. Greedy Initial Solution

For each student in the difficulty order:

1. Identify classes below their target capacity.
2. Remove classes that violate the student's forbidden-class constraint.
3. Evaluate each remaining candidate using a local placement score.
4. Select the candidate with the lowest local score.
5. Add the student to that class.

If no authorised class is available, the attempt fails and the next outer attempt may try a different random ordering.

### 9.1. Local placement score

The local score estimates the effect of adding one student to a candidate class.

It considers:

- deviation of Behaviour mean from the cohort target;
- deviation of Academic Score means from cohort targets;
- gender-proportion deviation;
- Individual Plans concentration;
- family-based School Assistant concentration;
- school-based School Assistant concentration;
- new-student concentration;
- presence of a requested `WITH` student;
- English Plans preference;
- concentration of Behaviour levels 4 and 5;
- Class Type mismatch;
- a small random tie-breaking term.

The local score is a heuristic used only to construct the initial solution. The final candidate is evaluated using the complete global cost function.

## 10. Global Cost Function

The global cost is calculated as a detailed sum of criterion-specific components.

```text
Global cost
├── Behaviour balance
├── Academic Score balance
│   ├── English
│   ├── French
│   └── Mathematics
├── Gender mix
├── Individual Plans
├── School Assistant distribution
│   ├── Family-based support
│   ├── School-based support
│   └── Mixed support penalty
├── WITH satisfaction
├── English Plans preference
├── EAL distribution
├── New-student distribution
├── Behaviour level 4 dispersion
├── Behaviour level 5 dispersion
├── Forbidden-class penalty
├── Class Type penalty
├── WITHOUT penalty
├── Strong gender-imbalance penalty
└── Previous Class dominance penalty
```

### 10.1. Mean-based criteria

For a class-level mean \(m_k\), cohort target \(m^*\), and weight \(w\):

\[
P_k = w(m_k - m^*)^2
\]

This pattern is used for:

- Behaviour;
- English Academic Score;
- French Academic Score;
- Mathematics Academic Score.

### 10.2. Proportion-based criteria

For a class proportion \(p_k\), cohort target \(p^*\), and weight \(w\):

\[
P_k = w(p_k-p^*)^2
\]

This pattern is used for:

- gender mix;
- Individual Plans;
- family-based School Assistant distribution.

### 10.3. Count-based dispersion

For a class count \(n_k\), target count \(n^*\), and weight \(w\):

\[
P_k = w(n_k-n^*)^2
\]

This pattern is used for:

- EAL distribution across EAL classes;
- new students;
- Behaviour level 4;
- Behaviour level 5.

## 11. Specific Cost Components

### 11.1. School-based School Assistant cost

The school-based support cost is non-linear:

| Count in class | Cost behaviour |
|---:|---|
| 0 | 0 |
| 1 | Small positive cost |
| 2 | Negative adjustment, favouring a pair |
| More than 2 | Increasing quadratic cost |

This encodes the operational preference for grouping two school-based support cases while discouraging excessive concentration.

### 11.2. Mixed School Assistant cost

A class containing both family-based and school-based School Assistant cases receives an additional penalty proportional to the product of the two counts.

### 11.3. `WITH` satisfaction

For each student with at least one resolved `WITH` preference:

- no co-located requested student: add the configured `No WITH satisfied` weight;
- at least one co-located requested student: no such penalty.

The model therefore optimises minimum satisfaction rather than requiring every requested relationship to be satisfied.

### 11.4. English Plans

Students flagged through English Plans receive a preference for `EAL` or `T` classes. The cost increases when they are assigned to `MAINSTREAM` and decreases relatively when assigned to a compatible support Class Type during local placement.

### 11.5. EAL distribution

Let \(E
i\) be the number of EAL students in EAL class \(i\), and \(E^*\) the average target. The component is:

\[
P_{EAL} = w_{EAL}\sum_i(E_i-E^*)^2
\]

### 11.6. `WITHOUT` constraints

Resolved `WITHOUT` relationships are symmetrised and de-duplicated. An incompatible pair assigned to the same class receives the configured penalty once.

### 11.7. Forbidden-class penalty

A student placed in their resolved forbidden class receives the configured penalty. In addition, the placement is rejected during initial construction and local search, and the final integrity check raises an error.

### 11.8. Class Type penalty

A student placed in a Class Type incompatible with their category receives the configured Class Type penalty.

### 11.9. Strong gender imbalance

For each non-empty class, the proportion of boys and girls is calculated. If either proportion is below 35%, the cost increases with the distance below the threshold.

### 11.10. Previous Class dominance

For every class, the proportion of students originating from the most represented Previous Class is calculated. If this proportion exceeds 60%, the class receives an increasing penalty.

## 12. Local Search Phase 1: Individual Moves

After the greedy construction, the algorithm attempts to improve the candidate using single-student moves.

For up to 120 improvement iterations:

1. Select a source class and a student.
2. Select a different destination class.
3. Reject the move if the destination is full.
4. Reject the move if the destination violates the forbidden-class constraint.
5. Temporarily remove the student from the source class.
6. Add the student to the destination class.
7. Validate allocation integrity.
8. Recalculate the global cost.
9. Keep the move only if it strictly reduces the current cost.
10. Otherwise restore the previous state.

The search stops early when a full pass produces no improvement.

This is a first-improvement local search rather than an exhaustive global optimisation.

## 13. Local Search Phase 2: Pairwise Swaps

The algorithm then attempts pairwise swaps between classes.

For up to 80 improvement iterations:

1. Select two distinct classes.
2. Select one student from each class.
3. Reject the swap if either destination violates a forbidden-class constraint.
4. Temporarily exchange the two students.
5. Validate allocation integrity.
6. Recalculate the global cost.
7. Keep the swap only if it strictly reduces the current cost.
8. Otherwise restore both students to their original classes.

As with individual moves, the search stops when a complete pass produces no improvement.

## 14. One Complete Attempt

A complete allocation attempt consists of:

```text
Resolve WITH and WITHOUT preferences
              |
              v
Create empty classes and cohort targets
              |
              v
Order students by placement difficulty
              |
              v
Build greedy initial allocation
              |
              v
Improve through individual moves
              |
              v
Improve through pairwise swaps
              |
              v
Validate integrity
              |
              v
Compute final global cost
```

The attempt returns:

- the complete class allocation;
- the calculated global score.

If construction or validation fails, the outer search records the failure and proceeds to the next attempt.

## 15. Repeated Attempts and Best-Solution Selection

The application repeats the complete attempt up to the configured maximum number of attempts.

Because the student order contains randomisation and the local placement score includes a small random tie-breaking term, different attempts may produce different candidate allocations.

The selection policy is:

\[
x^* = \arg\min_{x \in X_{valid}} C(x)
\]

where \(X_{valid}\) is the set of valid candidate allocations generated during the run.

Only the best valid candidate is exported.

The progress state records:

- current attempt;
- maximum attempts;
- best score so far;
- latest attempt score.

## 16. Stop Behaviour

The web interface can request a graceful stop by writing a stop marker to temporary state.

The outer attempt loop checks this marker before starting a new costly attempt. If the marker is detected:

- the search is marked as stopped;
- no final workbook is produced by the execution path;
- the result state records the stopped status;
- the background worker clears the busy marker.

A stop request does not forcibly interrupt the inner move or swap loop currently in progress. It is therefore a cooperative stop mechanism checked between attempts.

## 17. Integrity Verification

Before a candidate is accepted and before output is written, the algorithm verifies:

1. No student is assigned to an explicit forbidden class.
2. The number of assigned students equals the number of input students.
3. No student name appears more than once.
4. The sorted set of output student names equals the sorted input set.

These checks are especially important because the local-search operations temporarily mutate class membership.

## 18. Complexity Characteristics

The algorithm is heuristic and its runtime depends on:

- number of students;
- number of classes;
- number of outer attempts;
- number of move iterations;
- number of swap iterations;
- cost-function evaluation time;
- number of relational constraints.

The pairwise-swap phase can be computationally expensive because it explores combinations of classes and student pairs. Recalculating the complete global score improves conceptual simplicity and correctness but is less efficient than maintaining incremental score deltas.

The current design prioritises clarity and practical cohort sizes over asymptotic optimisation.

## 19. Reproducibility and Stochastic Behaviour

The algorithm uses random shuffling and a random tie-breaking term. Consequently, two runs with identical input and weights may produce different allocations.

The best result is selected by the same objective function in both cases. Increasing the number of attempts increases the opportunity to find a lower-cost candidate but also increases runtime.

For scientific benchmarking, a future version could expose an explicit random seed and record it in the result metadata.

## 20. Output and Interpretation

When the search completes successfully, the selected allocation is exported to Excel.

The output contains:

- a global statistics sheet;
- one sheet per class;
- class-level means and proportions;
- counts of support-related attributes;
- English Plans and Class Type information;
- `WITH` satisfaction indicators;
- `WITHOUT` satisfaction indicators;
- Previous Class dominance indicators.

The global score should be used to compare candidate allocations generated under the same cohort and weight configuration. It should not be interpreted as an absolute educational quality score.

The final workbook remains a decision-support document. Authorised school staff may review and adjust specific placements where professional, pastoral or safeguarding considerations require it.

## 21. Limitations and Future Improvements

The current implementation has several deliberate limitations:

- It uses local search rather than an exact optimisation solver.
- It does not guarantee a globally optimal solution.
- The stop mechanism is checked between outer attempts.
- Randomness is not currently exposed through a user-configurable seed.
- Class capacities are inferred from cohort size and number of classes.
- The public Class Type model is tailored to the current workflow.
- The score combines criteria with different natural scales and is therefore configuration-dependent.

Possible future improvements include:

- incremental cost evaluation;
- tabu search, simulated annealing or another metaheuristic;
- explicit hard-constraint modelling;
- random-seed recording;
- stronger input feasibility diagnostics;
- configuration profiles per school;
- criterion-level score reporting in the user interface;
- automated sensitivity analysis for weight selection;
- solver-backed optimisation for selected hard constraints.
