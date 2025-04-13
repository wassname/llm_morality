---
title: LLM Moral Values Leaderboard
description: Measure the moral values of LLMs
---

<Details title='Queries 1'>

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


```sql categories
select 
    * as category
from columns
```

</Details>

<Dropdown data={categories} name=category value=category defaultValue="v2_morality" />

<Dropdown data={prompts} name=prompt value=prompt defaultValue="zkp"/>

<Dropdown 
    data={models}
    name=model
    value=model
    multiple=true
    selectAllByDefault=true
/>

<!-- Selected: {inputs.model.value} -->

<Details title='What are the prompts?'>

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

```sql category_by_model
  select
      model_id,
      prompt_name,
      ${inputs.category.value}_mean as mean,
      ${inputs.category.value}_std as std,
      ${inputs.category.value}_count as count
  from values_agg
  where 
    prompt_name = '${inputs.prompt.value}'
    and model_id IN ${inputs.model.value}
  order by mean desc
```

**Comparing model "{inputs.category.value}" in "{inputs.prompt.value}" scenaro**

<BoxPlot
    data={category_by_model}
    name=model_id
    midpoint=mean
    confidenceInterval=std
    swapXY=true
    yFmt=pct0
/>

*Figure 1: The y axis shows the relative preference of the models for {inputs.category.value}. We use a [tau ranking](https://en.wikipedia.org/wiki/Kendall_rank_correlation_coefficient) to see if the chosen choices are assocated with a value of "{inputs.prompt.value}" of the model in the given scenario. The x axis shows the model id. The boxplot shows one standard deviation for each model.*


## Measurement

Given 3 choices such as

| Choice | Description | Value | Logprob |
|--------|-------------|-------|---------|
| 1      | Kill        | 0.7   | -0.5    |
| 2      | Save        | -0.5  | -0.7    |
| 3      | Walk away   | 0     | -0.1    |

We can measure the model's preference for each choice by looking at the ranking of the logprob of each choice. We use Kendall's tau to measure the correlation between the model's ranking of the choices and the values of the choices. The higher the tau, the more correlated the model's ranking is with the values of the choices.
Then we normalise each set of choices over all models, then take the statistics over all choices.
 


<!-- 
<BarChart
    data={category_by_model}
    x=model_id
    y=mean
    swapXY=true
    yFmt=pct0
/>

<BoxPlot
    data={category_by_model}
    name=model_id
    min=q005
    intervalBottom=q025
    midpoint=q050
    intervalTop=q075
    max=q095
    swapXY=true
    yFmt=pct0
/> -->

TODO

- [ ] decide on chart / norm
- [x] show each prompt
- [ ] describe what the data mean (relative tau rankings, given game scenarios). E.g. we are putting the model in a game scenario, asking it to choose, then we measure the implicit values in it's ranking of the choices

links:
- [data gen](https://github.com/wassname/llm_ethics_leaderboard)
- [code](https://github.com/wassname/llm_morality)
- [website](https://wassname.github.io/llm_morality)

**Made with [evidence](https://github.com/evidence-dev/evidence)**
