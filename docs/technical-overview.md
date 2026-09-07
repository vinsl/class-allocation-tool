# Class Allocation Tool – Technical Overview

## 1. Purpose

The Class Allocation Tool is a decision-support application for assigning students to classes within a primary-school cohort.

It transforms an Excel cohort extract into a proposed class allocation while considering academic, behavioural, organisational and relational criteria. The generated workbook is intended for professional review by school leadership before final publication.

> This repository is a portfolio case study. The original production source code, school data and deployment configuration are proprietary and are not included.

## 2. System Context

The application is designed for authorised school staff who need to prepare a new class structure from an existing cohort dataset.

The operational workflow is:

```text
Excel cohort extract
        |
        v
Input validation and normalisation
        |
        v
Cohort and Class Type configuration
        |
        v
Weighted multi-criteria optimisation
        |
        v
Integrity checks and statistics
        |
        v
Formatted Excel output for human review
```

The system does not replace educational leadership. It produces a structured proposal that can be inspected, discussed and manually adjusted when necessary.

## 3. Architecture

The application is implemented as a local Flask web application with a browser-based user interface.

```text
+-----------------------+
| Browser user interface|
| Flask templates       |
| Bootstrap / JavaScript|
+-----------+-----------+
            |
            | HTTP requests and JSON polling
            v
+-----------------------+
| Flask application     |
| Input validation      |
| Session orchestration |
| Configuration         |
| Download endpoints    |
+-----+--------+--------+
      |        |
      |        +------------------+
      |                           |
      v                           v
+-------------+          +------------------+
| Excel input |          | Background worker|
| extraction  |          | optimisation     |
| normalising |          | progress updates |
+------+------+          +---------+--------+
       |                             |
       +-------------+---------------+
                     |
                     v
          +-------------------------+
          | Local filesystem state  |
          | Parameters              |
          | Progress                |
          | Results                 |
          | Logs                    |
          | Configuration           |
          +------------+------------+
                       |
                       v
              +--------------------+
              | Excel output       |
              | Statistics sheet   |
              | One sheet per class|
              +--------------------+
```

## 4. Components

### 4.1. Web Application Layer

The Flask application exposes the browser workflow and coordinates the allocation process.

Its responsibilities include:

- Rendering the main configuration page.
- Providing French and English interface variants.
- Receiving the uploaded Excel workbook.
- Performing an initial input validation.
- Showing the detected number of students.
- Reading cohort parameters and class configuration.
- Selecting recommended or custom weights.
- Starting the background allocation worker.
- Exposing progress and status endpoints.
- Handling stop and reset requests.
- Providing the generated workbook for download.

The main user-facing phases are:

1. Parameter configuration.
2. Allocation processing.
3. Result presentation.
4. Optional reset for a new run.

### 4.2. Input Extraction Layer

The extraction layer reads the Excel workbook with `pandas` and `openpyxl` and converts each valid row into an internal student object.

The extraction process includes:

- Column-name normalisation.
- Whitespace and line-break removal.
- Text cleaning.
- Score conversion and range validation.
- Boolean normalisation.
- Date conversion.
- Optional-column detection.
- Extraction of `WITH` and `WITHOUT` references from multiple columns.
- Construction of a typed in-memory student representation.

Academic Scores are expected to use the range 1–5. Missing Academic Scores are accepted by the allocation engine and replaced by the neutral default value 3 when a numerical target is calculated.

Invalid scores outside the accepted range raise an error rather than silently entering the optimisation process.

### 4.3. Internal Data Model

The application represents each student with the attributes required by the allocation engine, including:

- Name and optional first name.
- Gender information.
- Previous class information.
- Class Type source information.
- English Plans status.
- Academic Scores.
- Behaviour score.
- Individual Plans.
- School Assistant category.
- `WITH` preferences.
- `WITHOUT` constraints.
- Forbidden class reference.
- New-student information.
- Arrival date.

A class is represented by:

- A canonical class name.
- A list of assigned students.
- A Class Type.

The implemented Class Types are:

- Type A.
- Type B.
- Type C.

### 4.4. Allocation Layer

The allocation layer is responsible for:

- Building class definitions.
- Computing cohort-level targets.
- Resolving student and class references.
- Generating an initial allocation.
- Improving the allocation through local search.
- Computing the global cost.
- Selecting the best result across repeated attempts.
- Running integrity checks before export.

### 4.5. Output Layer

The output layer creates a formatted Excel workbook containing:

- A `Stats` worksheet with class-level and cohort-level indicators.
- One worksheet per class.
- Student attributes relevant to operational review.
- Conditional formatting for selected imbalances.
- Information about `WITH` satisfaction and `WITHOUT` constraints.
- Previous-class dominance indicators.

The workbook is designed to remain understandable to non-technical school staff.

## 5. Filesystem State Management

The application uses a local filesystem layout rather than a database.

```text
data/
├── input/       # Uploaded cohort workbooks
├── output/      # Generated allocation workbooks
├── tmp/         # Parameters, progress, status, results and stop signals
├── config/      # Persisted custom weights
└── logs/        # Application logs
```

Temporary state includes:

- `params.json`: parameters for the active allocation.
- `progress.json`: current iteration, maximum iterations, best score and last score.
- `result.json`: output metadata and final result state.
- `busy.json`: marker indicating that an allocation is running.
- `stop.json`: user stop signal.

This design is appropriate for a single local deployment and keeps the installation simple. It is not a multi-user distributed architecture.

## 6. Request and Processing Flow

### 6.1. File Preview

When the user selects an Excel file, the browser sends it to a preview endpoint. The server temporarily stores the file, attempts to parse it and returns the number of detected students.

The temporary preview file is deleted after processing.

### 6.2. Allocation Start

When the user starts an allocation, the application:

1. Checks that no other allocation is currently running.
2. Validates the uploaded file and required fields.
3. Limits the number of classes to the supported interface range.
4. Limits the number of iterations to the supported interface range.
5. Stores the workbook under a cohort-based input name.
6. Generates canonical class names.
7. Builds the Class Type configuration.
8. Selects recommended or custom weights.
9. Writes the allocation parameters to temporary state.
10. Marks the application as busy.
11. Starts a daemon background thread.

### 6.3. Background Execution

The worker loads the parameters, reads the cohort, builds shared indexes and runs repeated optimisation attempts.

The web interface remains responsive because the optimisation runs outside the request-handling path.

### 6.4. Progress Reporting

The worker writes a lightweight JSON snapshot after each attempt. The browser polls the progress endpoint and displays:

- Current attempt.
- Maximum number of attempts.
- Best score found so far.
- Score of the latest attempt.
- Estimated remaining time calculated by the browser.

Progress-writing failures do not interrupt the allocation process.

### 6.5. Completion

Once the search finishes:

1. The best allocation is selected.
2. The integrity checks are executed.
3. The Excel workbook is generated.
4. Result metadata are saved.
5. The busy marker is removed.
6. The browser displays the best score and download link.

## 7. Class Type Rules

Class Types are configured through the user interface and converted into internal class metadata.

The allocation rules are:

| Student Class Type | Allowed destination |
|---|---|
| Type A | Any class type |
| Type B | Type B classes only |
| Type C | Type C classes when at least one exists; otherwise Type B classes |

A student can also be associated with English Plans. These plans are handled as a preference in the cost function, while Class Type compatibility is enforced during initial placement and validated during processing.

## 8. Configuration and Persistence

The application provides two weight modes:

- Recommended weights: the default configuration stored in the allocation module.
- Custom weights: values entered through the configuration page and persisted as JSON.

Custom weights are loaded when the application starts and are reused in subsequent sessions.

The application also derives the displayed academic year from the current date for the user interface.

## 9. Logging and Diagnostics

Logging is configured at application level and records operational events such as:

- Page access.
- Upload and preview results.
- Input validation failures.
- Allocation start and completion.
- Attempt scores.
- New best solutions.
- Stop requests.
- Output generation.
- Unexpected worker errors.

Constraint diagnostics record the number of resolved `WITH` preferences, `WITHOUT` pairs, students without resolved `WITH` requests and highly constrained students.

## 10. Integrity and Safety Checks

Before a solution is accepted and before output is written, the application verifies that:

- No student is assigned to an explicitly forbidden class.
- The total number of assigned students matches the input count.
- No student appears more than once.
- The set of output student names matches the input set.

These checks protect against accidental loss, duplication or forbidden placement during local-search operations.

## 11. Deployment Characteristics

The current application is designed for local or internal deployment:

- Python runtime.
- Windows-compatible launcher.
- Local Flask server.
- Browser access from the deployment machine or approved local network.
- Local filesystem persistence.

The architecture deliberately avoids external services and databases. This lowers installation complexity for a school environment but also means that authentication, concurrent multi-user execution, centralised storage and distributed job processing are outside the current scope.

## 12. Limitations and Extension Points

The current architecture processes one cohort per run and uses a local filesystem for state management.

Potential future extensions include:

- Configurable input-schema mappings.
- School-specific configuration profiles.
- Authentication and role-based access control.
- Database-backed session management.
- Concurrent job isolation.
- Centralised audit logs.
- Containerised deployment.
- Automated backup and retention policies.
- A production WSGI server.

The allocation engine is intentionally separated from the web presentation layer, which makes the business rules and criteria easier to extend independently of the user interface.