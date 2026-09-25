---
# try also 'default' to start simple
theme: default
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
# background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
# page transition
transition: slide-left
# use UnoCSS
css: unocss

defaults:
  layout: center

---

##### From stringly typed to strongly typed

<br/>

## Insights from re-designing a library to get <br/> the most out of type hints

<br/>

Janoś Gabler, appliedAI institute for europe


<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

<br/>

<div class="flex gap-4">


<div>
<img src="/figures/aai-institute-logo.svg" class="rounded" width="250" hspace="10" vspace="10" />
</div>

<div>
<img src="/figures/transferlab-logo.svg" class="rounded" width="520" hspace="10" vspace="10" />
</div>

</div>


---
src: ./about_me.md
---

---


<div class="flex gap-4">
<div>

##### Today

<br/>
<br/>

- `optimagic` as running example
- Learnings are not specific to optimization
- No knowledge or interest in optimization needed

<!-- Was formerly called estimagic because of origins as statistical estimation package-->
<!-- Today we only use optimagic as an example -->
<!-- This talk is not specific to numerical optimization at all -->
<!-- Tomorrow at 10:30 I'll give a talk on optimagic -->

</div>
<div>

##### Tomorrow

<br/>

<img src="/figures/optimagic_logo.svg" class="rounded" width="250" hspace="60"/>

<br/>

<img src="/figures/optimagic-talk-schedule.png" class="rounded" width="500"/>

</div>
</div>



---


<div>
<img src="/figures/stringly-typed.png" class="rounded" width="800" />
</div>


---

# Stringly typed example

<div class="flex gap-4">
<div>

```python {all|10|11|12}
import optimagic as om
import numpy as np

def fun(x):
    return x @ x

res = om.minimize(
    fun=fun,
    params=np.arange(3),
    algorithm="scipy_lbfgsb",
    algo_options={"stopping_maxiter": 1000}
    error_handling="raise",
)
```

</div>
<div>

<br/>
<br/>

- Algorithms are selected via strings
- Algorithms are configured via<br/> `dicts[str, Any]`
- Error handling is configured with a string


</div>
</div>



---




# Stringly-typed libraries



<div class="flex gap-20">
<div>

<br/>

- **Scikit learn**
<!-- widespread `str | Callable` arguments -->
<!-- `classification_report` is a string -->
- Seaborn
<!-- Puts a more stringly typed interface on top of matplotlib -->
<!-- Pass `..._kws` dictionaries to low-level plotting functions -->
- scipy.optimize
- CVXPY
- ...
<!-- Select algorithm via strings -->
<!-- Configure algorithms with option dictionaries -->


</div>
<div>

```python{all|5-6}
from sklearn import svm, datasets
from sklearn.model_selection import cross_val_score
X, y = datasets.load_iris(return_X_y=True)
clf = svm.SVC(random_state=0)
cross_val_score(
    clf, X, y, cv=5, scoring='recall_macro'
)
```
```txt
array([0.96..., 0.96..., 0.96..., 0.93..., 1.        ])
```

<br/>

- `scoring` is `str | Callable`
<!-- many similar examples in sklearn -->


</div>
</div>


---


# Stringly-typed libraries



<div class="flex gap-20">
<div>

<br/>

- Scikit learn
<!-- widespread `str | Callable` arguments -->
<!-- `classification_report` is a string -->
- **Seaborn**
<!-- Puts a more stringly typed interface on top of matplotlib -->
<!-- Pass `..._kws` dictionaries to low-level plotting functions -->
- scipy.optimize
- CVXPY
- ...
<!-- Select algorithm via strings -->
<!-- Configure algorithms with option dictionaries -->


</div>
<div>

```python{all|8}
import seaborn as sns

mpg = sns.load_dataset("mpg")
sns.regplot(
    data=mpg,
    x="weight",
    y="acceleration",
    scatter_kws={"alpha": 0.5},
)
```

<br/>

- `scatter_kws` are passed through to `sns.scatterplot`

<!-- Many similar examples -->


</div>
</div>

---

# Advantages of stringly-typed code

- Beginner friendly and popular
- Easy to implement
  - when wrapping other packages <!-- e.g. wrapping optimizers -->
  - when stacking up functionality <!-- e.g. scater_kws in seaborn -->





---
layout: fact
---

##### Is stringly typed good then?

---

# Problem: Brittle comparisons

<br/>

<div class="flex gap-20">
<div>

Previous example

```python
res = om.minimize(
    ...
    error_handling="raise",
)
```

Snippet from an optimizer

```python{all|4}
try:
    value = fun(x):
except Exception as e:
  if error_handling == "rais":
    raise RuntimeError("...") from e
  else:
    value = penalty(x)
```


<!-- often typos are less obvious -->

</div>
<div>

<br/>
<br/>

- Comparison of strings is brittle
  - Typos in user passed strings
  - Typos by developers
- Requires:
  - extensive testing
  - exhaustive if-elif-else


</div>
</div>


---

# Solution: Use Enums internally

<style>
    .slidev-code {
        font-size: 0.9rem !important;
    }
</style>

<br/>
<br/>

<div class="flex gap-4">
<div>

```python{all|2}
from enum import Enum

class ErrorHandling(Enum):
    RAISE = "raise"
    IGNORE = "ignore"
```

```python{4|all|4}
error_handling = ErrorHandling(
    user_error_handling
)

```

```python{8|all|4}
try:
    value = fun(x):
except Exception as e:
  if error_handling == ErrorHandling.RAISE:
    raise RuntimeError("...") from e
  else:
    value = penalty(x)

```

</div>
<div>

<br/>

- User can still pass a string
- User-typos are caught at runtime

  ```python
  error_handling = ErrorHandling("rais")
  ```
  ```bash
  ValueError: 'rais' is not a valid ErrorHandling
  ```

- Type-checkers and linters would find the previous problem!

</div>
</div>


---

# Problem: Typos detonate at runtime

<div class="flex gap-4">
<div>

<br/>
<br/>

```python{all|4}
res = om.minimize(
    fun=fun,
    params=np.arange(3),
    algorithm="scipy_lfbgsb",
)
```
```bash
ValueError: Invalid algorithm: scipy_lfbgsb.
```

</div>
<div>

<br/>
<br/>

- Internal Enum is not enough!
- Typos in users' code are still only discovered at runtime


</div>
</div>

<br/>

<v-clicks >


<span style="color: #18a5a7; ">Problem: Users like typing a string instead of importing and using an Enum!</span>

</v-clicks >


<!-- Sklearn, plotly and others have safer options but they are not popular -->

---

# Solution: Make safer options fun to use

<div class="flex gap-4">
<div>

<br/>

<img src="/figures/algo-selection-1.png" class="rounded" width="360"/>

</div>
<div>

<br/>

- **Autocomplete shows all algorithms**
- Naming scheme to filter by library
- Attributes to filter by properties
    - This is in proof-of-concept stage
    - Now: gradient-based vs. gradient-free
    - Future: bounded, constrained, local, global, etc

</div>
</div>


---

# Solution: Make safer options fun to use

<div class="flex gap-4">
<div>

<br/>

<img src="/figures/algo-selection-3.png" class="rounded" width="360"/>

</div>
<div>

<br/>

- Autocomplete shows all algorithms
- **Naming scheme to filter by library**
- Attributes to filter by properties
    - This is in proof-of-concept stage
    - Now: gradient-based vs. gradient-free
    - Future: bounded, constrained, local, global, etc

</div>
</div>


---

# Solution: Make safer options fun to use

<div class="flex gap-4">
<div>

<br/>

<img src="/figures/algo-selection-2.png" class="rounded" width="360"/>

</div>
<div>

<br/>

- Autocomplete shows all algorithms
- Naming scheme to filter by library
- **Attributes to filter by properties**
    - This is in proof-of-concept stage
    - Now: gradient-based vs. gradient-free
    - Future: bounded, constrained, local, global, etc

</div>
</div>

---

# Problem: Bad IDE support for refactoring

<style>
.slidev-code {
    font-size: 1.25rem !important;
}
</style>

<div class="grid grid-cols-2 gap-4">
<div>

Imagine an optimizer history

```python
history = {
    "criterion_value": [],
    "params": [],
    "timestamp": [],
}
```

- used in several optimizers
- used in several plotting functions

</div>
<div>

<br/>

- For alignment with SciPy we want to rename `"criterion_value"` to `"fun_value"`
- Search-and-replace is your best shot!


</div>
</div>


---

# Solution: Replace dicts by dataclasses

<style>
    .slidev-code {
        font-size: 1.2rem !important;
    }
</style>

<div class="grid grid-cols-2 gap-4">
<div>

```python
from dataclasses import dataclass

@dataclass
class History:
    criterion_value: list[float]
    params: list[NDArray]
    timestamp: list[float]

```

</div>
<div>

<br/>

- Renaming with `f2` in any editor
- Bonus: Easier to document and type-hint

</div>
</div>


---

# Problem: Which option can I put there?

<style>
    .slidev-code {
        font-size: 1.25rem !important;
    }
</style>

```python {all|4-5}
res = om.minimize(
    fun=fun,
    params=np.arange(3),
    algorithm="scipy_lbfgsb",
    algo_options={"stopping_maxiter": 1000}
    error_handling="raise",
)
```



---

# Problem: Which option can I put there?


<!-- From the scipy.optimize documentation (which is super good!) -->

<br/>

<img src="/figures/options-1.png" class="rounded" width="680"/>


---

# Problem: Which option can I put there?

<br/>


<img src="/figures/options-2.png" class="rounded" width="750"/>

---

# Problem: Which option can I put there?

<br/>


<img src="/figures/options-3.png" class="rounded" width="600"/>


---

# Problem: Which option can I put here?

<style>
    .slidev-code {
        font-size: 1.25rem !important;
    }
</style>

```python {all|5-8}
res = om.minimize(
    fun=fun,
    params=np.arange(3),
    algorithm="scipy_lbfgsb",
    algo_options={
        "stopping_maxiter": 1000,
        "maxcorr": 12,
    }
    error_handling="raise",
)
```

---

# Problem: Which option can I put here?

<style>
    .slidev-code {
        font-size: 1.25rem !important;
    }
</style>

<div class="grid grid-cols-2 gap-18">
<div>

<br/>
<br/>

```python {7}
res = om.minimize(
    fun=fun,
    params=np.arange(3),
    algorithm="scipy_lbfgsb",
    algo_options={
        "stopping_maxiter": 1000,
        "maxcorr": 12,
    }
    error_handling="raise",
)
```


</div>
<div>

<br/>
<br/>
<br/>
<br/>

##### Did you spot<br/>the typo?

</div>
</div>


<br/>




---
layout: fact
---

##### Solution: Instead of a string, Enum, or algorithm class, we pass a configured algorithm instance!


---

# Solution: Configured algorithms

<br/>

<div class="flex gap-4">
<div>

<br/>

<img src="/figures/configured-algo-1.png" class="rounded" width="580"/>


</div>
<div>

<br/>
<br/>
<br/>

- Editor shows all<br/> supported options<br/>for selected optimizer
- Type-hints go beyond<br/>int or float

</div>
</div>

---

# Solution: Configured algorithms

<br/>

<div class="flex gap-4">
<div>

<br/>

<img src="/figures/configured-algo-2.png" class="rounded" width="580"/>


</div>
<div>

<br/>
<br/>
<br/>

- The more you type<br/>the more you zoom in<br/>on relevant options

</div>
</div>


---

# Solution: Configured algorithms

<br/>

<div class="flex gap-4">
<div>

<br/>

<img src="/figures/configured-algo-3.png" class="rounded" width="580"/>


</div>
<div>

<br/>
<br/>
<br/>

- The more you type<br/>the more you zoom in<br/>on relevant options

</div>
</div>

---

# Bonus

<br/>

- Algorithms can validate options (using Pydantic or similar approaches)
- Copy constructors can avoid the pitfalls of mutable option dictionaries

<br/>

```python
base_algo = om.algorithms.fides(stopping_max_iterations=1000)
algorithms = [base_algo.with_option(initial_radius=r) for r in [0.1, 0.2, 0.5]]

for algo in algorithms:
    minimize(
        # ...
        algorithm=algo,
        # ...
    )

```


---

# Adopting type-hints in a code-base


- Design your types before adding type-hints!
- Add mypy to pre-commits and CI!
- Configure mypy with a whitelist approach!
- Start adding type-hints to low-level files!


---

# Further reading

<br/>
<br/>

<div class="flex gap-12">
<div>

<img src="/figures/ep-typing-phone.png" class="rounded" width="230"/>

</div>

<div>
<img src="/figures/kobzol-phone.png" class="rounded" width="270"/>

</div>

<div>


<img src="/figures/robust_python.jpeg" class="rounded" width="270" />


</div>
</div>

[tinyurl.com/ep-typing](https://tinyurl.com/ep-typing)

---
src: ./acknowledgements.md
---


---


 # Looking for help!

<br/>


- 💡 Suggest an optimizer we should wrap
- 🤝 Help us wrap more optimizers (it's easy!)
- 🗣️ Provide feedback about anything you are missing

<br/>

<img src="/figures/optimagic-talk-schedule.png" class="rounded" width="450"/>
