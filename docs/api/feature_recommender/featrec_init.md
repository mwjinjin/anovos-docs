# <code>featrec_init</code>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
from anovos.feature_recommender.feature_exploration import *
from re import finditer
import copy


def camel_case_split(input):
    """

    Parameters
    ----------
    input :
        Input (string) which requires cleaning

    Returns
    -------

    """
    processed_input = ""
    matches = finditer(".+?(?:(?<=[a-z])(?=[A-Z])|(?<=[A-Z])(?=[A-Z][a-z])|$)", input)
    for m in matches:
        processed_input += str(m.group(0)) + str(" ")
    return processed_input


def recommendation_data_prep(df, name_column, desc_column):
    """

    Parameters
    ----------
    df :
        Input DataFrame
    name_column :
        Column name of Input DataFrame attribute/ feature name (string)
    desc_column :
        Column name of Input DataFrame attribute/ feature description (string)
        :return list_corpus: List of prepared data for Feature Recommender functions
        :return df_prep: Processed DataFrame for Feature Recommender functions

    Returns
    -------

    """
    if not isinstance(df, pd.DataFrame):
        raise TypeError("Invalid input for df")
    if name_column not in df.columns and name_column != None:
        raise TypeError("Invalid input for name_column")
    if desc_column not in df.columns and desc_column != None:
        raise TypeError("Invalid input for desc_column")
    if name_column == None and desc_column == None:
        raise TypeError("Need at least one input for either name_column or desc_column")
    df_prep = copy.deepcopy(df)
    if name_column == None:
        df_prep[desc_column] = df_prep[desc_column].astype(str)
        df_prep_com = df_prep[desc_column]
    elif desc_column == None:
        df_prep[name_column] = df_prep[name_column].astype(str)
        df_prep_com = df_prep[name_column]
    else:
        df_prep[name_column] = df_prep[name_column].str.replace("_", " ")
        df_prep[name_column] = df_prep[name_column].astype(str)
        df_prep[desc_column] = df_prep[desc_column].astype(str)
        df_prep_com = df_prep[[name_column, desc_column]].agg(" ".join, axis=1)
    df_prep_com = df_prep_com.replace({"[^A-Za-z0-9 ]+": " "}, regex=True)
    for i in range(len(df_prep_com)):
        df_prep_com[i] = df_prep_com[i].strip()
        df_prep_com[i] = camel_case_split(df_prep_com[i])
    list_corpus = df_prep_com.to_list()
    return list_corpus, df_prep


df_groupby_fer = (
    df_input_fer.groupby([feature_name_column, feature_desc_column])
    .agg(
        {
            industry_column: lambda x: ", ".join(set(x.dropna())),
            usecase_column: lambda x: ", ".join(set(x.dropna())),
            source_column: lambda x: ", ".join(set(x.dropna())),
        }
    )
    .reset_index()
)
list_train_fer, df_rec_fer = recommendation_data_prep(
    df_groupby_fer, feature_name_column, feature_name_column
)
list_embedding_train_fer = model_fer.encode(list_train_fer, convert_to_tensor=True)
```
</pre>
</details>
## Functions
<dl>
<dt id="anovos.feature_recommender.featrec_init.camel_case_split"><code class="name flex">
<span>def <span class="ident">camel_case_split</span></span>(<span>input)</span>
</code></dt>
<dd>
<div class="desc"><h2 id="parameters">Parameters</h2>
<p>input :
Input (string) which requires cleaning</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def camel_case_split(input):
    """

    Parameters
    ----------
    input :
        Input (string) which requires cleaning

    Returns
    -------

    """
    processed_input = ""
    matches = finditer(".+?(?:(?<=[a-z])(?=[A-Z])|(?<=[A-Z])(?=[A-Z][a-z])|$)", input)
    for m in matches:
        processed_input += str(m.group(0)) + str(" ")
    return processed_input
```
</pre>
</details>
</dd>
<dt id="anovos.feature_recommender.featrec_init.recommendation_data_prep"><code class="name flex">
<span>def <span class="ident">recommendation_data_prep</span></span>(<span>df, name_column, desc_column)</span>
</code></dt>
<dd>
<div class="desc"><h2 id="parameters">Parameters</h2>
<p>df :
Input DataFrame
name_column :
Column name of Input DataFrame attribute/ feature name (string)
desc_column :
Column name of Input DataFrame attribute/ feature description (string)
:return list_corpus: List of prepared data for Feature Recommender functions
:return df_prep: Processed DataFrame for Feature Recommender functions</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def recommendation_data_prep(df, name_column, desc_column):
    """

    Parameters
    ----------
    df :
        Input DataFrame
    name_column :
        Column name of Input DataFrame attribute/ feature name (string)
    desc_column :
        Column name of Input DataFrame attribute/ feature description (string)
        :return list_corpus: List of prepared data for Feature Recommender functions
        :return df_prep: Processed DataFrame for Feature Recommender functions

    Returns
    -------

    """
    if not isinstance(df, pd.DataFrame):
        raise TypeError("Invalid input for df")
    if name_column not in df.columns and name_column != None:
        raise TypeError("Invalid input for name_column")
    if desc_column not in df.columns and desc_column != None:
        raise TypeError("Invalid input for desc_column")
    if name_column == None and desc_column == None:
        raise TypeError("Need at least one input for either name_column or desc_column")
    df_prep = copy.deepcopy(df)
    if name_column == None:
        df_prep[desc_column] = df_prep[desc_column].astype(str)
        df_prep_com = df_prep[desc_column]
    elif desc_column == None:
        df_prep[name_column] = df_prep[name_column].astype(str)
        df_prep_com = df_prep[name_column]
    else:
        df_prep[name_column] = df_prep[name_column].str.replace("_", " ")
        df_prep[name_column] = df_prep[name_column].astype(str)
        df_prep[desc_column] = df_prep[desc_column].astype(str)
        df_prep_com = df_prep[[name_column, desc_column]].agg(" ".join, axis=1)
    df_prep_com = df_prep_com.replace({"[^A-Za-z0-9 ]+": " "}, regex=True)
    for i in range(len(df_prep_com)):
        df_prep_com[i] = df_prep_com[i].strip()
        df_prep_com[i] = camel_case_split(df_prep_com[i])
    list_corpus = df_prep_com.to_list()
    return list_corpus, df_prep
```
</pre>
</details>
</dd>
</dl>