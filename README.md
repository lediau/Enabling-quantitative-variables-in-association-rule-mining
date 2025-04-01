# Enabling-quantitative-variables-in-association-rule-mining 🦠
## Overview
This project represents a test task for JetBrains Research department. It provides a simple method to optimize rule sets for classification tasks by applying **heuristic compression techniques** to reduce redundant rules while maintaining performance.

## Table of Contents
- [Installation](#installation)
- [Solution pipeline walk-through](#solution-pipeline-walk-through)
- [Heuristics](#heuristics)
    - [Metrics for rule evaluation](#metrics-for-rule-evaluation)
    - [Threshhold](#threshhold)
    - [Cluster-based rule refinement](#cluster-based-rule-refinement)
    - [Evaluation metrics for rule selection](#evaluation-metrics-for-rule-selection)
    - [Search for the optimal subset of rules](#search-for-the-optimal-subset-of-rules)
        - [Exhaustive Search for Optimal Rule Subset](#exhaustive-search-for-optimal-rule-subset)
        - [Genetic Algorithm for Rule Optimization](#genetic-algorithm-for-rule-optimization)
- [Challenges](#challenges)
- [Flaws of the solution](#flaws-of-the-solution)
- [Ideas not implemented](#ideas-not-implemented)
- [Failed ideas](#failed-ideas)

## Installation

The entire code has been written in the Jupyter Notebook to make it easier for the reviewer to access and check all the code at once. However, if required, the source code can be easily reorganized using modules and so.

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

To avoid any confusion, let's settle once and for all how we define four types of (non)errors:

- **True Positives (TP)**: Donors that satisfy the rule and are indeed old.
- **False Positives (FP)**: Donors that satisfy the rule but are not old.
- **False Negatives (FN)**: Donors that do not satisfy the rule but are old.
- **True Negatives (TN)**: Donors that do not satisfy the rule and are not old.

### Metrics for rule evaluation

[Step 3 of pipeline] Below, there is a simple scheme of the way the metrics are used, and further on we add the details for each metrics:
![metric_rank](https://github.com/user-attachments/assets/e1f7b160-8a9b-4ae9-872e-803023380524)

- About **f1 score**:

$$prec = \frac{TP}{TP + FP}, \quad\quad recall = \frac{TP}{TP + FN}, \quad\quad f1 = 2\cdot\frac{prec \cdot recall}{prec + recall}.$$

- About **completeness**:

$$spec = \left|\text{biomarkers}\right|, \quad\quad freq = \frac{TP + FP}{\left|\text{donors}\right|}, \quad\quad compl = freq + \beta\cdot\min(spec, MAXLEN), \quad \text{where } \beta = \frac{1}{10} \text{ and } MAXLEN = 6.$$

*Parameters*: $\beta$ -- the weight of the specificness.

*Hyperparameters*: *MAXLEN* -- the number of biomarkers or their negation in the rule.

- About **p value** from **chi-square test**: see [documentation](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.chi2_contingency.html) if needed.
- The aggregation formula for **evaluation**:

$$\boxed{\text{eval} = w_1 \cdot \text{f1} + w_2\cdot \text{freq} + w_3\cdot (1 - \text{pval}),}\quad \text{where by default } w_1=4, w_2=2, w_3=4.$$

*Parameters*: $w_1, w_2, w_3$.

### Threshhold
[Step 4 of pipeline] We use the [1.5 IQR Whiskers](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.boxplot.html) method to set thresholds for poorly performing rules, namely for three metrics: f1 score, frequency, and p value.

### Cluster-based rule refinement
[Step 6 of pipeline] The goal is to create **clusters of highly correlated rules** and **select the best-performing rule from each cluster**. Correlation is determined based on the dataset of rules: if different rules predominantly apply to the same donors, they are considered highly correlated.

To achieve this, we use **Hierarchical Clustering**, with the number of clusters determined using the **Elbow Method**, provided a predefined threshold is met. This ensures that we retain only the most informative and distinct rules while reducing redundancy. Check the [scipy documentation](https://docs.scipy.org/doc/scipy/reference/cluster.hierarchy.html) for more details.

<p align="center">
   <img src="https://github.com/user-attachments/assets/8e6cfd70-a23a-4d88-ab17-8e0f771df746" alt="HC" width="500"/>
</p>

*Hyperparameters*: Number of clusters.

### Evaluation metrics for rule selection
[Step 8 of pipeline] To evaluate the effectiveness of the rules as an ensemble rather than individually, we introduce these key metrics:

- **Accuracy**: Since we predict only the **positive class** (being old), the traditional accuracy is not the best measure. For this, we inspect the correct predictions (**TP**) with a **penalty** (of half-weight) for incorrect predictions (**FP**). It is worth noting, that here we talk about group predictions (50%+ of rules agree), not individual rule predictions.

$$acc = \frac{TP - \gamma\cdot FP}{TP + FP}, \quad\quad \text{where } \gamma = 0.5.$$

*Hyperparameters*: $\gamma$ -- the penalty coefficient for wrong identification of new donors as old. 

- **Detailed accuracy**: The accuracy metric is **discrete** and tends to be non-informative for similar rulesets. For this, we implement a bit more detailed accuracy, which takes into account the amount of rules in the ruleset that predicted correctly with a penalty (half-weight) for the number of rules that predicted incorrectly.

$$acc^{*} = \frac{\sum{TP} - \gamma\cdot\sum{FP}}{\sum(TP+FP)}, \quad\quad \text{where } \gamma = 0.5.$$

*Hyperparameters*: $\gamma$.

- **Coverage**: A messy custom metric that is supposed to ensure rules cover a large portion of the dataset. Of course, for our small dataset, the whole set is usually covered by a small number of rules. However, in real-life scenarios this is not expected to be the case.

$$ \text{Coverage} = 2 \cdot \frac{|\bigcup_i\left(D_{i} \cap D_{\text{old}}\right)|}{|D_{\text{old}}|} - \frac{|\bigcup_i D_{i}| - |D_{\text{old}}|}{|D_{\text{old}}|}, $$

where $D_{old}$ represents the rows of old donors, while $D_i$ the rows of the old donors that satisfy the rule $i$. Intersection and union is performed row-wise among all possible values of $i$, while $|\cdot|$ stands for the number of rows in the set.

- **AUC (precision-recall)**: Read [documentation](https://scikit-learn.org/stable/auto_examples/model_selection/plot_precision_recall.html) if needed.

$$\text{AUC}\_\text{pr} = \sum_{i=1}^{n-1} (R_{i+1} - R_i) \cdot P_i$$

- **Evaluation formula**: Again, an aggregation formula for the forementioned metrics.

$$\boxed{\text{eval} = w_1\cdot \text{AUC}\_\text{pr} + w_2\cdot \text{acc} + w_3\cdot \text{acc}^{*} + w_4\cdot \text{coverage}},\quad \text{where by default } w_1=1, w_2=1, w_3=1,w_4=1.$$

*Parameters*: $w_1, w_2, w_3, w_4$.

### Search for the optimal subset of rules
[Step 9 of pipeline] To select the best subset of rules from a given set, we apply two methods depending on the size of the ruleset:

#### **Exhaustive Search for Optimal Rule Subset**

**[Deterministic]** This method performs an exhaustive search over all possible subsets of rules and selects the best one based on an evaluation function. It is only feasible for rule sets of less than 15 rules.  

##### Process:  
1. Iterate over all subsets of rules, starting from size 2 to avoid trivial solutions.  
2. Evaluate each subset of rules $R$ using the evaluation metric:  

$$ S(R) = \text{evaluation}(R).$$

3. Track the best subset $R^*$ with the highest score:  

$$ R^* = \arg\max_{R \subseteq \mathcal{R}} S(R).$$

#### **Genetic Algorithm for Rule Optimization**
**[Probabilistic]** For larger rule sets, we employ a Genetic Algorithm (GA) to find an optimal subset probabilistically.  

##### Process:  
1. **Population Initialization:** Generate an initial population of random subsets, ensuring at least 2 rules per subset.  

2. **Selection:** Compute scores for all subsets and retain the top 50%:  

$$ P_{\text{selected}} = \text{top}_{0.5} \left( (R, S(R)) \mid R \in P \right). $$

3. **Crossover:** Merge two subsets to form a new one and remove duplicate rules.

4. **Mutation:** With probability $p_{\text{mutate}}$, modify the subset:  
     - Randomly add a new rule.  
     - Remove an existing rule (if the subset contains more than 2 rules).  

5. **Iteration:** Run for $G$ generations, tracking the best subset:  

$$R^* = \arg\max_{R \in P_G} S(R).$$

##### Hyperparameters:  
- $N$: Population size.  
- $G$: Number of generations.  
- $p_{\text{mutate}}$: Mutation probability.  


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
