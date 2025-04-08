---
title: LLM Moral Values Leaderboard
---

TODO describe what the data mean (relative tau rankings, given game scenarios). E.g. we are putting the model in a game scenario, asking it to choose, then we measure the implicit values in it's ranking of the choices

github url
 

```sql categories
  select 
      * as category
  from columns
```


<Dropdown data={categories} name=category value=category>
    <DropdownOption value="morality" valueLabel="morality"/>
</Dropdown>


```sql prompts
  select 
      * as prompt
  from prompts
```

<Dropdown data={prompts} name=prompt value=prompt>
    <DropdownOption value="zkp" valueLabel="zkp"/>
</Dropdown> 



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
      quantile_cont(${inputs.category.value}, 0.05) FILTER (${inputs.category.value}!=0) as q005,
      quantile_cont(${inputs.category.value}, 0.25) FILTER (${inputs.category.value}!=0) as q025,
      quantile_cont(${inputs.category.value}, 0.40) FILTER (${inputs.category.value}!=0) as q040,
      quantile_cont(${inputs.category.value}, 0.50) FILTER (${inputs.category.value}!=0) as q050,
      quantile_cont(${inputs.category.value}, 0.60) FILTER (${inputs.category.value}!=0) as q060,
      quantile_cont(${inputs.category.value}, 0.75) FILTER (${inputs.category.value}!=0) as q075,
      quantile_cont(${inputs.category.value}, 0.95) FILTER (${inputs.category.value}!=0) as q095,
      mean(${inputs.category.value}) as mean,
      stddev(${inputs.category.value}) as stddev,
      count(${inputs.category.value}) as count
  from values_full
  where prompt_name = '${inputs.prompt.value}'
  group by model_id, prompt_name
  order by mean desc
```

<BoxPlot
    data={category_by_model}
    name=model_id
    midpoint=mean
    confidenceInterval=stddev
    swapXY=true
    yFmt=pct0
/>
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
