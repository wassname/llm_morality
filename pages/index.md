---
title: LLM Moral Values Leaderboard
description: Measuring Moral Values in Language Models
---

## Executive Summary

This leaderboard evaluates how different Language Learning Models (LLMs) align with various moral values across diverse scenarios. We analyze model preferences in text adventure scenarios (run, kill, try to negotiate) and compare them against labels from an LLM judge. Our methodology includes controls for positional bias and uses standardized measurements across scenarios.

Our key finding: model moral preferences are often performative and diminish when we use mechanistic interpretability to steer their behavior toward taking scenarios more seriously. This suggests many models are optimized for human approval rather than having genuine moral preferences.

<!-- <Details title='Queries 1'> -->

```sql prompts
  select 
      distinct prompt_name as prompt
  from values_agg
```

```sql models
  select 
      distinct model_id as model
  from values_agg
```


```sql labels
select 
    label,
    description
from labels
```

<!-- </Details> -->

<Dropdown data={labels} name=label value=label description=description defaultValue="v2_morality" />

<Dropdown data={prompts} name=prompt value=prompt defaultValue="zkp"/>

<Dropdown 
    data={models}
    name=model
    value=model
    multiple=true
    selectAllByDefault=true
/>

<Details title='What are the prompts?'>

The prompts come from text adventure games from the [MACHIAVELLI](https://github.com/aypan17/machiavelli) benchmark, which were [converted to a scenario by scenario format](https://github.com/wassname/machiavelli_as_ds). The benchmark is based on human-written, text-based Choose-Your-Own-Adventure games, and the labels are a mix of human and AI labels. We analyze the most distinct 1024 scenarios.

Each scenario is presented to the model along with several possible choices, and we analyze how the model ranks these choices in terms of various moral dimensions.

  ```sql prompt_example
    select 
        prompt_example,
        name
    from prompt_examples
    where name == '${inputs.prompt.value}'
  ```

<pre>
  {prompt_example[0].prompt_example}
</pre>
  
</Details>

<!-- <Details title="Queries 2"> -->

```sql label_desc
  select
      label,
      description
  from labels
  where label = '${inputs.label.value}'
```

```sql category_by_model
  select
      model_id,
      prompt_name,
      ${inputs.label.value}_mean as mean,
      ${inputs.label.value}_std as std,
      ${inputs.label.value}_count as count
  from values_agg
  where 
    prompt_name = '${inputs.prompt.value}'
    and model_id IN ${inputs.model.value}
  order by mean desc
```

<!-- </Details> -->

**Comparing model "{inputs.label.value}" in "{inputs.prompt.value}" scenario**

<BoxPlot
    data={category_by_model}
    name=model_id
    midpoint=mean
    confidenceInterval=std
    swapXY=true
    yFmt=pct0
/>

Figure 1: The y axis shows each model's preference alignment score for {inputs.label.value}. This score measures how strongly the model's ranking of choices correlates with an LLM judge's evaluation of those choices for the value "{inputs.label.value}" in the "{inputs.prompt.value}" scenario. We normalize these correlation scores across scenarios. The x axis shows the model id. The boxplot shows one standard deviation for each model.

## Label: {label_desc[0].label}

> {label_desc[0].description}

## Measurement 1: Agreement with moral scores

We measure how a model's preferences align with judgments from an LLM evaluator using [Kendall's tau](https://en.wikipedia.org/wiki/Kendall_rank_correlation_coefficient)-b rank correlation. For example, given three choices:

| Choice | Description | Non-Violence (LLM Judge) | Logprob (Model) |
|--------|-------------|-------------------|-----------------|
| 1 | Kill | -0.7 | -0.5 |
| 2 | Save | 0.5 | -0.7 |
| 3 | Walk away | 0 | -0.1 |

For each set of choices, we:
1. Rank the choices by their values (e.g., Non-Violence) as assigned by an LLM judge that has analyzed the scenarios
2. Rank the same choices by the tested model's log probabilities (representing the model's preferences)
3. Calculate Kendall's tau-b correlation between these two rankings using scipy.stats.kendalltau

Kendall's tau measures the correspondence between two rankings, with values near 1 indicating strong agreement and values near -1 indicating strong disagreement.

We found that alternate values, such as calibrated logprobs, were very noisy and not useful for this analysis.

To make results comparable across different scenarios, we normalize the tau values using z-scores within each scenario group (`df.groupby('scenario_row_id').map(zscore)`). This ensures that differences in tau scale across scenarios don't skew the overall analysis.

The final statistics (mean, standard deviation, count) are then calculated across all normalized scenarios for each model, providing a standardized measure of how well each model's preferences align with the LLM judge's evaluations.


## Controlling for Positional Bias

To mitigate potential positional bias in our measurements (where models might prefer options based on their position rather than content), each scenario is evaluated with 5 different permutations of the choice order. We calculate Kendall's tau for each permutation and then average the results. This approach controls for ordering effects in how models process sequences of choices.

For computational efficiency, we leverage KV cache when generating logprobs for permutations, modifying only the tokens representing the choices rather than regenerating the entire context. This allows us to efficiently test multiple permutations while maintaining consistent context processing.

## Measurement 2: Preference modeling


We also tried using the [Bradley-Terry model](https://en.wikipedia.org/wiki/Bradley%E2%80%93Terry_model) to rank models based on their preferences.

```sql choix
  select
      *
  from choix
  where 
    model_id IN ${inputs.model.value}
```

<DataTable data={choix} rows=50 title="Moral Preference Rankings" subtitle="Bradley-Terry Model" compact="true"  />

This table lists the rankings modeled using the Bradley-Terry model, which is closely related to the Elo rating system used to rank chess players. Computed using I-LSR.

<DataTable 
  data={choix}
  compact="true"
  title="Moral Preference Rankings"
  subtitle="Bradley-Terry Model"
  rows=50
>
  <Column id="model_id" />
  <Column id="mean1" contentType=colorscale colorScale={['#6db678','white','#ce5050']} />
  <Column id="mean" contentType=colorscale colorScale={['#6db678','white','#ce5050']} />
  {#each labels as label}
    <Column id={label.label} contentType=colorscale colorScale={['#6db678','white','#ce5050']} />
  {/each}
</DataTable>

## Steering

How does steering a model's behavior toward being more credulous and honest affect its moral judgments? We investigated whether making models take scenarios more seriously impacts their moral preferences. By measuring changes in their moral rankings, we can understand if increased credulity leads to better or worse moral alignment.

We used [RepEng](https://github.com/vgel/repeng) to find representation steering directions for credulity - taking scenarios more seriously. The training data for finding these directions is available in our [scenario engagement dataset](https://github.com/wassname/llm_ethics_leaderboard/blob/c0414e65ec877ac6a22e186c23046d595b6983b5/data/scenario_engagement_dataset.json).

### Key Findings
When steered toward taking scenarios more seriously:
- Models with extreme moral positions became more moderate
- "Evil-aligned" models showed their behavior was largely performative
- This suggests some models are optimized for human approval rather than having genuine moral preferences, **presenting significant challenges for alignment efforts**

### Model Authenticity
Analyzing the results reveals interesting patterns in model behavior:
- Misaligned and 4chan-trained models often showed performative "evil" behavior that diminished under steering
- Some of the nicest models, like HuggingFace's Zephyr and AllenAI's OLMo, exhibited performative "good" behavior, which also diminished under steering
- Chinese models like Qwen showed strong moral characteristics
- Open source models, especially judging models, and RAG models showed strong genuine moral alignment
- Model size was not strongly correlated with authentic moral reasoning capability

```sql choix_steering_credulity
  select
      *
  from choix_steering_credulity
```

<DataTable data={choix_steering_credulity} rows=50 title="Changes in Moral Preference Rankings" subtitle="When Steered for Credulity" compact="true"  />

## Implications for AI Safety

Our findings have important implications for AI alignment research:

1. **Performative behavior masks true preferences**: Models may learn to give responses that humans approve of, rather than reflecting authentic moral reasoning.

2. **Scaling alone may be insufficient**: The fact that model size doesn't correlate strongly with authentic moral reasoning suggests that simply scaling up models may not solve alignment challenges.

3. **Need for better evaluation methods**: Standard benchmarks may fail to distinguish between models that have genuine moral alignments versus those giving performative responses.

## Additional Resources

- [Data Generation Code](https://github.com/wassname/llm_ethics_leaderboard)
- [Analysis Code](https://github.com/wassname/llm_morality)
- [Project Website](https://wassname.github.io/llm_morality)

**Built with [Evidence](https://github.com/evidence-dev/evidence)**
