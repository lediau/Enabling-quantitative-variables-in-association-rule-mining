# Enabling-quantitative-variables-in-association-rule-mining 🦠
## Overview
This project represents a test task for JetBrains Research department. It provides a simple method to optimize rule sets for classification tasks by applying **heuristic compression techniques** to reduce redundant rules while maintaining performance.

## Table of Contents
- [Installation](#installation)
- [Solution pipeline walk-through](#solution-pipeline-walk-through)
- [Heuristics](#heuristics)
- [Challenges](#challenges)
- [Flaws of the solution](#flaws-of-the-solution)
- [Ideas not implemented](#ideas-not-implemented)
- [Failed ideas](#failed-ideas)

## Installation

The entire code has been written in the Jupyter Notebook to make it easier for the reviewer to access and check all the code at once. However, if required, the source code can be reorganized using modules and so.

## Solution pipeline walk-through

1. **Load dataset and rules** in a suitable form for further computations;
2. **Make decision on `NaN` containing rows**: skip them only when working with the columns containing such values;
3. **Compile a list of metrics** and their aggregate to evaluate the usefulness of a rule (see heuristics);
4. **Prune** the rules that don't exceed the threshold;
5. **Cluster rules with equal metric values**, leave only the rule or combination (conjunction) of them with the highest evaluation metric;
6. **Cluster highly-correlated rules**, and for each cluster pick the rule or combination (conjunction) of them with the highest evaluation metric;
7. **Make an assumption on the prediction** of a set of rules: $50\%$ majority rule in our case;
8. **Compile a list of metrics** and their aggregate to evaluate the efficiency of a set of rules (see heuristics);
9. **Perform deterministic or probabilistic search** (depending on ruleset size) for the subset that maximized the efficiency;
10. **Prepare the output**.

## Heuristics



## Challenges
- Lack of practical experience and mentorship;
- Lack of concentration and time due to exams;
- Confusion on the expected complexity of the solution;
- Small amount of test data.

## Flaws of the solution
- Not a well-connected solution: it feels more like useful (or not) pieces of ideas put together;
- Feels a bit like childish, since I focused on the phrase *"how to decide what's important for rule compression"*. I didn't want to waste much time, since I was not sure this is the direction you would like to see;
- Some metrics feel repetitive and might be correlated to each other.

## Ideas not implemented
- Use more advanced methods to select better rules;
- Study covariance for 'biomarker' and 'NOT biomarker' separately;
- Refine and test metrics, as now they are just empirical and random;
- Fine-tuning several (hyper)parameters, such as weights in formulas, number of clusters, etc.;
- In general, test and research more for better decision-making.

## Failed ideas
- Semantic and math.logic reductions and equivalent transformations, due to lack of OR operator;
- If the original rule has low-accuracy, replace it by its negation: can't generate for 2+ biomarker rules (lack of OR operator);
- For A and AB, if precA > precAB, then automatically remove AB, since freqA>freqAB. Maybe add A + NOT B, and remove A as well.
