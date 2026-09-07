# Class Allocation Tool
A school class-allocation system that transforms cohort data into balanced,
constraint-aware class assignments for primary education.

> Portfolio case study: the original implementation and school data are proprietary.
> This repository contains documentation, anonymised examples and a rewritten technical demonstration.

## Problem

Assigning students to classes is a multi-criteria decision problem.

A school must balance academic levels, behaviour, gender distribution,
language-support requirements, individual learning plans, social relationships
and operational constraints while still producing a usable result for school leadership.

Manual allocation is time-consuming, generally unofficially relying on teachers' paid time, and difficult to reproduce consistently.

## My solution

The tool provides a browser-based workflow that allows authorised staff to:

1. Upload a cohort spreadsheet.
2. Validate the detected student population.
3. Configure the target number and type of classes.
4. Use recommended or custom optimisation weights.
5. Run the allocation process.
6. Monitor progress and intermediate scores.
7. Download the resulting Excel workbook for review and manual adjustment.

This tool can be directly implemented on the school's internal server
to ensure data protection and resilience.

## Product walkthrough

![Class allocation configuration](demo/screenshots/configuration.png)

## Main capabilities

- Excel-based cohort import and validation.
- Configurable number of classes.
- Class types configuration (language- or level-based types for eg.).
- Recommended and custom optimisation weights.
- Multi-criteria allocation engine.
- Background execution with progress reporting.
- Best-solution tracking across multiple iterations.
- Excel export for leadership review.
- French and English user interface.
- Logging and basic error diagnostics.
- Reset and stop controls for long-running calculations.

## Allocation criteria already implemented

The optimisation model can account for:

- Behaviour scores.
- Academic levels.
- Gender balance.
- Individual learning plans.
- School-based and family-based support requirements.
- Second language placement constraints.
- Friendship requests.
- Incompatible student pairs.
- Reuse of previous-year class structures.

