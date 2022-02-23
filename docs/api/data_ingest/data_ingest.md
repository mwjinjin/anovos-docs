# <code>data_ingest</code>
<p>This module consists of functions to read the dataset as Spark DataFrame, concatenate/join with other functions (
if required), and perform some basic ETL actions such as selecting, deleting, renaming and/or recasting columns. List
of functions included in this module are: - read_dataset - write_dataset - concatenate_dataset - join_dataset -
delete_column - select_column - rename_column - recast_column</p>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
# coding=utf-8
"""This module consists of functions to read the dataset as Spark DataFrame, concatenate/join with other functions (
if required), and perform some basic ETL actions such as selecting, deleting, renaming and/or recasting columns. List
of functions included in this module are: - read_dataset - write_dataset - concatenate_dataset - join_dataset -
delete_column - select_column - rename_column - recast_column """
import pyspark.sql.functions as F
from pyspark.sql import DataFrame

from anovos.shared.utils import pairwise_reduce


def read_dataset(spark, file_path, file_type, file_configs={}):
    """This function reads the input data path and return a Spark DataFrame. Under the hood, this function is based
    on generic Load functionality of Spark SQL.  It requires following arguments:

    - **file_path**: file (or directory) path where the input data is saved. File path can be a local path or s3 path
    (when running with AWS cloud services) - **file_type**: file format of the input data. Currently, we support CSV,
    Parquet or Avro. Avro data source requires an external package to run, which can be configured with spark-submit
    options (--packages org.apache.spark:spark-avro_2.11:2.4.0). - **file_configs** (optional): Rest of the valid
    configurations can be passed through this argument in a dictionary format. All the key/value pairs written in
    this argument are passed as options to DataFrameReader, which is created using SparkSession.read.

    Parameters
    ----------
    spark :
        Spark Session
    file_path :
        Path to input data (directory or filename).
        Compatible with local path and s3 path (when running in AWS environment).
    file_type :
        csv", "parquet", "avro".
        Avro data source requires an external package to run, which can be configured with
        spark-submit (--packages org.apache.spark:spark-avro_2.11:2.4.0).
    file_configs :
        This argument is passed in a dictionary format as key/value pairs
        e.g. {"header": "True","delimiter": "|","inferSchema": "True"} for csv files.
        All the key/value pairs in this argument are passed as options to DataFrameReader,
        which is created using SparkSession.read. (Default value = {})

    Returns
    -------

    """
    odf = spark.read.format(file_type).options(**file_configs).load(file_path)
    return odf


def write_dataset(idf, file_path, file_type, file_configs={}, column_order=[]):
    """This function saves the Spark DataFrame in the user-provided output path. Like read_dataset, this function is
    based on the generic Save functionality of Spark SQL.  It requires the following arguments:

    - **idf**: Spark DataFrame to be saved - **file_path**: file (or directory) path where the output data is to be
    saved. File path can be a local path or s3 path (when running with AWS cloud services) - **file_type**: file
    format of the output data. Currently, we support CSV, Parquet or Avro. The Avro data source requires an external
    package to run, which can be configured with spark-submit options (--packages
    org.apache.spark:spark-avro_2.11:2.4.0). - **file_configs** (optional): The rest of the valid configuration can
    be passed through this argument in a dictionary format, e.g., repartition, mode, compression, header, delimiter,
    etc. All the key/value pairs written in this argument are passed as options to DataFrameWriter is available using
    Dataset.write operator. If the number of repartitions mentioned through this argument is less than the existing
    DataFrame partitions, then the coalesce operation is used instead of the repartition operation to make the
    execution work. This is because the coalesce operation doesn’t

    Parameters ---------- idf : Input Dataframe file_path : Path to output data (directory or filename). Compatible
    with local path and s3 path (when running in AWS environment). file_type : csv", "parquet", "avro". Avro data
    source requires an external package to run, which can be configured with spark-submit (--packages
    org.apache.spark:spark-avro_2.11:2.4.0). file_configs : This argument is passed in dictionary format as key/value
    pairs. Some of the potential keys are header, delimiter, mode, compression, repartition. compression options -
    uncompressed, gzip (doesn't work with avro), snappy (only valid for parquet) mode options - error (default),
    overwrite, append repartition - None (automatic partitioning) or an integer value () e.g. {"header":"True",
    "delimiter":",",'compression':'snappy','mode':'overwrite','repartition':'10'}. column_order : list of columns in
    the order in which Dataframe is to be written. If None or [] is specified, then the default order is applied.

    Returns
    -------

    """

    if not column_order:
        column_order = idf.columns
    else:
        if not isinstance(column_order, list):
            raise TypeError("Invalid input type for column_order argument")
        if len(column_order) != len(idf.columns):
            raise ValueError(
                "Count of column(s) specified in column_order argument do not match Dataframe"
            )
        diff_cols = [x for x in column_order if x not in set(idf.columns)]
        if diff_cols:
            raise ValueError(
                "Column(s) specified in column_order argument not found in Dataframe: "
                + str(diff_cols)
            )

    mode = file_configs["mode"] if "mode" in file_configs else "error"
    repartition = (
        int(file_configs["repartition"]) if "repartition" in file_configs else None
    )

    if repartition is None:
        idf.select(column_order).write.format(file_type).options(**file_configs).save(
            file_path, mode=mode
        )
    else:
        exist_parts = idf.rdd.getNumPartitions()
        req_parts = int(repartition)
        if req_parts > exist_parts:
            idf.select(column_order).repartition(req_parts).write.format(
                file_type
            ).options(**file_configs).save(file_path, mode=mode)
        else:
            idf.select(column_order).coalesce(req_parts).write.format(
                file_type
            ).options(**file_configs).save(file_path, mode=mode)


def concatenate_dataset(*idfs, method_type="name"):
    """This function combines multiple dataframes into a single dataframe. A pairwise concatenation is performed on
    the dataframes, instead of adding one dataframe at a time to the bigger dataframe. This function leverages union
    functionality of Spark SQL. It requires the following arguments:

    - ***idfs**: Varying number of dataframes to be concatenated - **method_type**: index or name. This argument
    needs to be entered as a keyword argument. The “index” method involves concatenating the dataframes by the column
    index. IF the sequence of column is not fixed among the dataframe, this method should be avoided. The “name”
    method involves concatenating by columns names. The 1st dataframe passed under idfs will define the final columns
    in the concatenated dataframe. It will throw an error if any column in the 1st dataframe is not available in any
    of other dataframes.

    Parameters ---------- dfs : All dataframes to be concatenated (with the first dataframe columns) method_type :
    index", "name". This argument needs to be passed as a keyword argument. "index" method concatenates by column
    index positioning, without shuffling columns. "name" concatenates after shuffling and arranging columns as per
    the first dataframe. First dataframe passed under idfs will define the final columns in the concatenated
    dataframe, and will throw error if any column in first dataframe is not available in any of other dataframes. (
    Default value = "name") *idfs :


    Returns
    -------

    """
    if method_type not in ["index", "name"]:
        raise TypeError("Invalid input for concatenate_dataset method")
    if method_type == "name":
        odf = pairwise_reduce(
            lambda idf1, idf2: idf1.union(idf2.select(idf1.columns)), idfs
        )  # odf = reduce(DataFrame.unionByName, idfs) # only if exact no. of columns
    else:
        odf = pairwise_reduce(DataFrame.union, idfs)
    return odf


def join_dataset(*idfs, join_cols, join_type):
    """This function joins multiple dataframes into a single dataframe by a joining key column. Pairwise joining is
    done on the dataframes, instead of joining individual dataframes to the bigger dataframe. This function leverages
    join functionality of Spark SQL. It requires the following arguments:

    - ***idfs**: Varying number of all dataframes to be joined - **join_cols**: Key column(s) to join all dataframes
    together. In case of multiple columns to join, they can be passed in a list format or a single text format where
    different column names are separated by pipe delimiter “|” - **join_type**: “inner”, “full”, “left”, “right”,
    “left_semi”, “left_anti”

    Parameters
    ----------
    idfs :
        All dataframes to be joined
    join_cols :
        Key column(s) to join all dataframes together.
        In case of multiple columns to join, they can be passed in a list format or
        a string format where different column names are separated by pipe delimiter “|”.
    join_type :
        inner", “full”, “left”, “right”, “left_semi”, “left_anti”
    *idfs :


    Returns
    -------

    """
    if isinstance(join_cols, str):
        join_cols = [x.strip() for x in join_cols.split("|")]
    odf = pairwise_reduce(
        lambda idf1, idf2: idf1.join(idf2, join_cols, join_type), idfs
    )
    return odf


def delete_column(idf, list_of_cols, print_impact=False):
    """This function is used to delete specific columns from the input data. It is executed using drop functionality
    of Spark SQL. It is advisable to use this function if the number of columns to delete is lesser than the number
    of columns to select; otherwise, it is recommended to use select_column. It requires the following arguments:

    - **idf**: Input dataframe - **list_of_cols**: This argument, in a list format, specifies the columns required to
    be deleted from the input dataframe. Alternatively, instead of list, columns can be specified in a single text
    format where different column names are separated by pipe delimiter “|” - **print_impact**: This argument is to
    compare number of columns before and after the operation.

    Parameters
    ----------
    idf :
        Input Dataframe
    list_of_cols :
        List of columns to delete e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    list_of_cols = list(set(list_of_cols))

    odf = idf.drop(*list_of_cols)

    if print_impact:
        print("Before: \nNo. of Columns- ", len(idf.columns))
        print(idf.columns)
        print("After: \nNo. of Columns- ", len(odf.columns))
        print(odf.columns)
    return odf


def select_column(idf, list_of_cols, print_impact=False):
    """This function is used to select specific columns from the input data. It is executed using select operation of
    spark dataframe. It is advisable to use this function if the number of columns to select is lesser than the
    number of columns to drop; otherwise, it is recommended to use delete_column. It requires the following arguments:

    - **idf**: Input dataframe - **list_of_cols**: This argument, in a list format, specifies the columns required to
    be selected from the input dataframe. Alternatively, instead of list, columns can be specified in a single text
    format where different column names are separated by pipe delimiter “|” - **print_impact**: This argument is to
    compare number of columns before and after the operation.

    Parameters
    ----------
    idf :
        Input Dataframe
    list_of_cols :
        List of columns to select e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    list_of_cols = list(set(list_of_cols))

    odf = idf.select(list_of_cols)

    if print_impact:
        print("Before: \nNo. of Columns-", len(idf.columns))
        print(idf.columns)
        print("\nAfter: \nNo. of Columns-", len(odf.columns))
        print(odf.columns)
    return odf


def rename_column(idf, list_of_cols, list_of_newcols, print_impact=False):
    """This function is used to rename the columns of the input data. Multiple columns can be renamed; however,
    the sequence they passed as an argument is critical and must be consistent between list_of_cols and
    list_of_newcols. It requires the following arguments:

    - **idf**: Input dataframe - **list_of_cols**: This argument, in a list format, is used to specify the columns
    required to be renamed in the input dataframe. Alternatively, instead of a list, columns can be specified in a
    single text format where different column names are separated by pipe delimiter “|” - **list_of_newcols**: This
    argument, in a list format, is used to specify the new column name, i.e. the first element in list_of_cols will
    be the original column name, and the corresponding first column in list_of_newcols will be the new column name. -
    **print_impact**: This argument is to compare column names before and after the operation.

    Parameters
    ----------
    idf :
        Input Dataframe
    list_of_cols :
        List of old column names e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
    list_of_newcols :
        List of corresponding new column names e.g., ["newcol1","newcol2"].
        Alternatively, new column names can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "newcol1|newcol2".
        First element in list_of_cols will be original column name,
        and corresponding first column in list_of_newcols will be new column name.
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(list_of_newcols, str):
        list_of_newcols = [x.strip() for x in list_of_newcols.split("|")]

    mapping = dict(zip(list_of_cols, list_of_newcols))
    odf = idf.select([F.col(i).alias(mapping.get(i, i)) for i in idf.columns])

    if print_impact:
        print("Before: \nNo. of Columns- ", len(idf.columns))
        print(idf.columns)
        print("After: \nNo. of Columns- ", len(odf.columns))
        print(odf.columns)
    return odf


def recast_column(idf, list_of_cols, list_of_dtypes, print_impact=False):
    """This function is used to modify the datatype of columns. Multiple columns can be cast; however,
     the sequence they passed as argument is critical and needs to be consistent between list_of_cols and
     list_of_dtypes.
     It requires the following arguments:

    - **idf**: Input dataframe
    - **list_of_cols**: This argument, in a list format, is used to specify the columns required to be recast in the
    input dataframe. Alternatively, instead of a list, columns can be specified in a single text format where different
     column names are separated by pipe delimiter “|”
    - **list_of_dtypes**: This argument, in a list format, is used to specify the datatype, i.e. the first element in
    list_of_cols will column name, and the corresponding element in list_of_dtypes will be new datatype such as float,
     integer, string, double, decimal, etc. (case insensitive).
    - **print_impact**: This argument is to compare schema before and after the operation.

    Parameters
    ----------
    idf :
        Input Dataframe
    list_of_cols :
        List of columns to cast e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
    list_of_dtypes :
        List of corresponding datatypes e.g., ["type1","type2"].
        Alternatively, datatypes can be specified in a string format,
        where they are separated by pipe delimiter “|” e.g., "type1|type2".
        First element in list_of_cols will column name and corresponding element in list_of_dtypes
        will be new datatypes such as "float", "integer", "long", "string", "double", decimal" etc.
        Datatypes are case insensitive e.g. float or Float are treated as same.
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(list_of_dtypes, str):
        list_of_dtypes = [x.strip() for x in list_of_dtypes.split("|")]

    odf = idf
    for i, j in zip(list_of_cols, list_of_dtypes):
        odf = odf.withColumn(i, F.col(i).cast(j))

    if print_impact:
        print("Before: ")
        idf.printSchema()
        print("After: ")
        odf.printSchema()
    return odf
```
</pre>
</details>
## Functions
<dl>
<dt id="anovos.data_ingest.data_ingest.concatenate_dataset"><code class="name flex">
<span>def <span class="ident">concatenate_dataset</span></span>(<span>*idfs, method_type='name')</span>
</code></dt>
<dd>
<div class="desc"><p>This function combines multiple dataframes into a single dataframe. A pairwise concatenation is performed on
the dataframes, instead of adding one dataframe at a time to the bigger dataframe. This function leverages union
functionality of Spark SQL. It requires the following arguments:</p>
<ul>
<li><strong><em>idfs</em>*: Varying number of dataframes to be concatenated - </strong>method_type**: index or name. This argument
needs to be entered as a keyword argument. The “index” method involves concatenating the dataframes by the column
index. IF the sequence of column is not fixed among the dataframe, this method should be avoided. The “name”
method involves concatenating by columns names. The 1st dataframe passed under idfs will define the final columns
in the concatenated dataframe. It will throw an error if any column in the 1st dataframe is not available in any
of other dataframes.</li>
</ul>
<p>Parameters ---------- dfs : All dataframes to be concatenated (with the first dataframe columns) method_type :
index", "name". This argument needs to be passed as a keyword argument. "index" method concatenates by column
index positioning, without shuffling columns. "name" concatenates after shuffling and arranging columns as per
the first dataframe. First dataframe passed under idfs will define the final columns in the concatenated
dataframe, and will throw error if any column in first dataframe is not available in any of other dataframes. (
Default value = "name") *idfs :</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def concatenate_dataset(*idfs, method_type="name"):
    """This function combines multiple dataframes into a single dataframe. A pairwise concatenation is performed on
    the dataframes, instead of adding one dataframe at a time to the bigger dataframe. This function leverages union
    functionality of Spark SQL. It requires the following arguments:

    - ***idfs**: Varying number of dataframes to be concatenated - **method_type**: index or name. This argument
    needs to be entered as a keyword argument. The “index” method involves concatenating the dataframes by the column
    index. IF the sequence of column is not fixed among the dataframe, this method should be avoided. The “name”
    method involves concatenating by columns names. The 1st dataframe passed under idfs will define the final columns
    in the concatenated dataframe. It will throw an error if any column in the 1st dataframe is not available in any
    of other dataframes.

    Parameters ---------- dfs : All dataframes to be concatenated (with the first dataframe columns) method_type :
    index", "name". This argument needs to be passed as a keyword argument. "index" method concatenates by column
    index positioning, without shuffling columns. "name" concatenates after shuffling and arranging columns as per
    the first dataframe. First dataframe passed under idfs will define the final columns in the concatenated
    dataframe, and will throw error if any column in first dataframe is not available in any of other dataframes. (
    Default value = "name") *idfs :


    Returns
    -------

    """
    if method_type not in ["index", "name"]:
        raise TypeError("Invalid input for concatenate_dataset method")
    if method_type == "name":
        odf = pairwise_reduce(
            lambda idf1, idf2: idf1.union(idf2.select(idf1.columns)), idfs
        )  # odf = reduce(DataFrame.unionByName, idfs) # only if exact no. of columns
    else:
        odf = pairwise_reduce(DataFrame.union, idfs)
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_ingest.data_ingest.delete_column"><code class="name flex">
<span>def <span class="ident">delete_column</span></span>(<span>idf, list_of_cols, print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><p>This function is used to delete specific columns from the input data. It is executed using drop functionality
of Spark SQL. It is advisable to use this function if the number of columns to delete is lesser than the number
of columns to select; otherwise, it is recommended to use select_column. It requires the following arguments:</p>
<ul>
<li><strong>idf</strong>: Input dataframe - <strong>list_of_cols</strong>: This argument, in a list format, specifies the columns required to
be deleted from the input dataframe. Alternatively, instead of list, columns can be specified in a single text
format where different column names are separated by pipe delimiter “|” - <strong>print_impact</strong>: This argument is to
compare number of columns before and after the operation.</li>
</ul>
<h2 id="parameters">Parameters</h2>
<p>idf :
Input Dataframe
list_of_cols :
List of columns to delete e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
print_impact :
True, False (Default value = False)</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def delete_column(idf, list_of_cols, print_impact=False):
    """This function is used to delete specific columns from the input data. It is executed using drop functionality
    of Spark SQL. It is advisable to use this function if the number of columns to delete is lesser than the number
    of columns to select; otherwise, it is recommended to use select_column. It requires the following arguments:

    - **idf**: Input dataframe - **list_of_cols**: This argument, in a list format, specifies the columns required to
    be deleted from the input dataframe. Alternatively, instead of list, columns can be specified in a single text
    format where different column names are separated by pipe delimiter “|” - **print_impact**: This argument is to
    compare number of columns before and after the operation.

    Parameters
    ----------
    idf :
        Input Dataframe
    list_of_cols :
        List of columns to delete e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    list_of_cols = list(set(list_of_cols))

    odf = idf.drop(*list_of_cols)

    if print_impact:
        print("Before: \nNo. of Columns- ", len(idf.columns))
        print(idf.columns)
        print("After: \nNo. of Columns- ", len(odf.columns))
        print(odf.columns)
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_ingest.data_ingest.join_dataset"><code class="name flex">
<span>def <span class="ident">join_dataset</span></span>(<span>*idfs, join_cols, join_type)</span>
</code></dt>
<dd>
<div class="desc"><p>This function joins multiple dataframes into a single dataframe by a joining key column. Pairwise joining is
done on the dataframes, instead of joining individual dataframes to the bigger dataframe. This function leverages
join functionality of Spark SQL. It requires the following arguments:</p>
<ul>
<li><strong><em>idfs</em>*: Varying number of all dataframes to be joined - </strong>join_cols<strong>: Key column(s) to join all dataframes
together. In case of multiple columns to join, they can be passed in a list format or a single text format where
different column names are separated by pipe delimiter “|” - </strong>join_type**: “inner”, “full”, “left”, “right”,
“left_semi”, “left_anti”</li>
</ul>
<h2 id="parameters">Parameters</h2>
<p>idfs :
All dataframes to be joined
join_cols :
Key column(s) to join all dataframes together.
In case of multiple columns to join, they can be passed in a list format or
a string format where different column names are separated by pipe delimiter “|”.
join_type :
inner", “full”, “left”, “right”, “left_semi”, “left_anti”
*idfs :</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def join_dataset(*idfs, join_cols, join_type):
    """This function joins multiple dataframes into a single dataframe by a joining key column. Pairwise joining is
    done on the dataframes, instead of joining individual dataframes to the bigger dataframe. This function leverages
    join functionality of Spark SQL. It requires the following arguments:

    - ***idfs**: Varying number of all dataframes to be joined - **join_cols**: Key column(s) to join all dataframes
    together. In case of multiple columns to join, they can be passed in a list format or a single text format where
    different column names are separated by pipe delimiter “|” - **join_type**: “inner”, “full”, “left”, “right”,
    “left_semi”, “left_anti”

    Parameters
    ----------
    idfs :
        All dataframes to be joined
    join_cols :
        Key column(s) to join all dataframes together.
        In case of multiple columns to join, they can be passed in a list format or
        a string format where different column names are separated by pipe delimiter “|”.
    join_type :
        inner", “full”, “left”, “right”, “left_semi”, “left_anti”
    *idfs :


    Returns
    -------

    """
    if isinstance(join_cols, str):
        join_cols = [x.strip() for x in join_cols.split("|")]
    odf = pairwise_reduce(
        lambda idf1, idf2: idf1.join(idf2, join_cols, join_type), idfs
    )
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_ingest.data_ingest.read_dataset"><code class="name flex">
<span>def <span class="ident">read_dataset</span></span>(<span>spark, file_path, file_type, file_configs={})</span>
</code></dt>
<dd>
<div class="desc"><p>This function reads the input data path and return a Spark DataFrame. Under the hood, this function is based
on generic Load functionality of Spark SQL.
It requires following arguments:</p>
<ul>
<li><strong>file_path</strong>: file (or directory) path where the input data is saved. File path can be a local path or s3 path
(when running with AWS cloud services) - <strong>file_type</strong>: file format of the input data. Currently, we support CSV,
Parquet or Avro. Avro data source requires an external package to run, which can be configured with spark-submit
options (&ndash;packages org.apache.spark:spark-avro_2.11:2.4.0). - <strong>file_configs</strong> (optional): Rest of the valid
configurations can be passed through this argument in a dictionary format. All the key/value pairs written in
this argument are passed as options to DataFrameReader, which is created using SparkSession.read.</li>
</ul>
<h2 id="parameters">Parameters</h2>
<p>spark :
Spark Session
file_path :
Path to input data (directory or filename).
Compatible with local path and s3 path (when running in AWS environment).
file_type :
csv", "parquet", "avro".
Avro data source requires an external package to run, which can be configured with
spark-submit (&ndash;packages org.apache.spark:spark-avro_2.11:2.4.0).
file_configs :
This argument is passed in a dictionary format as key/value pairs
e.g. {"header": "True","delimiter": "|","inferSchema": "True"} for csv files.
All the key/value pairs in this argument are passed as options to DataFrameReader,
which is created using SparkSession.read. (Default value = {})</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def read_dataset(spark, file_path, file_type, file_configs={}):
    """This function reads the input data path and return a Spark DataFrame. Under the hood, this function is based
    on generic Load functionality of Spark SQL.  It requires following arguments:

    - **file_path**: file (or directory) path where the input data is saved. File path can be a local path or s3 path
    (when running with AWS cloud services) - **file_type**: file format of the input data. Currently, we support CSV,
    Parquet or Avro. Avro data source requires an external package to run, which can be configured with spark-submit
    options (--packages org.apache.spark:spark-avro_2.11:2.4.0). - **file_configs** (optional): Rest of the valid
    configurations can be passed through this argument in a dictionary format. All the key/value pairs written in
    this argument are passed as options to DataFrameReader, which is created using SparkSession.read.

    Parameters
    ----------
    spark :
        Spark Session
    file_path :
        Path to input data (directory or filename).
        Compatible with local path and s3 path (when running in AWS environment).
    file_type :
        csv", "parquet", "avro".
        Avro data source requires an external package to run, which can be configured with
        spark-submit (--packages org.apache.spark:spark-avro_2.11:2.4.0).
    file_configs :
        This argument is passed in a dictionary format as key/value pairs
        e.g. {"header": "True","delimiter": "|","inferSchema": "True"} for csv files.
        All the key/value pairs in this argument are passed as options to DataFrameReader,
        which is created using SparkSession.read. (Default value = {})

    Returns
    -------

    """
    odf = spark.read.format(file_type).options(**file_configs).load(file_path)
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_ingest.data_ingest.recast_column"><code class="name flex">
<span>def <span class="ident">recast_column</span></span>(<span>idf, list_of_cols, list_of_dtypes, print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><p>This function is used to modify the datatype of columns. Multiple columns can be cast; however,
the sequence they passed as argument is critical and needs to be consistent between list_of_cols and
list_of_dtypes.
It requires the following arguments:</p>
<ul>
<li><strong>idf</strong>: Input dataframe</li>
<li><strong>list_of_cols</strong>: This argument, in a list format, is used to specify the columns required to be recast in the
input dataframe. Alternatively, instead of a list, columns can be specified in a single text format where different
column names are separated by pipe delimiter “|”</li>
<li><strong>list_of_dtypes</strong>: This argument, in a list format, is used to specify the datatype, i.e. the first element in
list_of_cols will column name, and the corresponding element in list_of_dtypes will be new datatype such as float,
integer, string, double, decimal, etc. (case insensitive).</li>
<li><strong>print_impact</strong>: This argument is to compare schema before and after the operation.</li>
</ul>
<h2 id="parameters">Parameters</h2>
<p>idf :
Input Dataframe
list_of_cols :
List of columns to cast e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
list_of_dtypes :
List of corresponding datatypes e.g., ["type1","type2"].
Alternatively, datatypes can be specified in a string format,
where they are separated by pipe delimiter “|” e.g., "type1|type2".
First element in list_of_cols will column name and corresponding element in list_of_dtypes
will be new datatypes such as "float", "integer", "long", "string", "double", decimal" etc.
Datatypes are case insensitive e.g. float or Float are treated as same.
print_impact :
True, False (Default value = False)</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def recast_column(idf, list_of_cols, list_of_dtypes, print_impact=False):
    """This function is used to modify the datatype of columns. Multiple columns can be cast; however,
     the sequence they passed as argument is critical and needs to be consistent between list_of_cols and
     list_of_dtypes.
     It requires the following arguments:

    - **idf**: Input dataframe
    - **list_of_cols**: This argument, in a list format, is used to specify the columns required to be recast in the
    input dataframe. Alternatively, instead of a list, columns can be specified in a single text format where different
     column names are separated by pipe delimiter “|”
    - **list_of_dtypes**: This argument, in a list format, is used to specify the datatype, i.e. the first element in
    list_of_cols will column name, and the corresponding element in list_of_dtypes will be new datatype such as float,
     integer, string, double, decimal, etc. (case insensitive).
    - **print_impact**: This argument is to compare schema before and after the operation.

    Parameters
    ----------
    idf :
        Input Dataframe
    list_of_cols :
        List of columns to cast e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
    list_of_dtypes :
        List of corresponding datatypes e.g., ["type1","type2"].
        Alternatively, datatypes can be specified in a string format,
        where they are separated by pipe delimiter “|” e.g., "type1|type2".
        First element in list_of_cols will column name and corresponding element in list_of_dtypes
        will be new datatypes such as "float", "integer", "long", "string", "double", decimal" etc.
        Datatypes are case insensitive e.g. float or Float are treated as same.
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(list_of_dtypes, str):
        list_of_dtypes = [x.strip() for x in list_of_dtypes.split("|")]

    odf = idf
    for i, j in zip(list_of_cols, list_of_dtypes):
        odf = odf.withColumn(i, F.col(i).cast(j))

    if print_impact:
        print("Before: ")
        idf.printSchema()
        print("After: ")
        odf.printSchema()
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_ingest.data_ingest.rename_column"><code class="name flex">
<span>def <span class="ident">rename_column</span></span>(<span>idf, list_of_cols, list_of_newcols, print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><p>This function is used to rename the columns of the input data. Multiple columns can be renamed; however,
the sequence they passed as an argument is critical and must be consistent between list_of_cols and
list_of_newcols. It requires the following arguments:</p>
<ul>
<li><strong>idf</strong>: Input dataframe - <strong>list_of_cols</strong>: This argument, in a list format, is used to specify the columns
required to be renamed in the input dataframe. Alternatively, instead of a list, columns can be specified in a
single text format where different column names are separated by pipe delimiter “|” - <strong>list_of_newcols</strong>: This
argument, in a list format, is used to specify the new column name, i.e. the first element in list_of_cols will
be the original column name, and the corresponding first column in list_of_newcols will be the new column name. -
<strong>print_impact</strong>: This argument is to compare column names before and after the operation.</li>
</ul>
<h2 id="parameters">Parameters</h2>
<p>idf :
Input Dataframe
list_of_cols :
List of old column names e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
list_of_newcols :
List of corresponding new column names e.g., ["newcol1","newcol2"].
Alternatively, new column names can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "newcol1|newcol2".
First element in list_of_cols will be original column name,
and corresponding first column in list_of_newcols will be new column name.
print_impact :
True, False (Default value = False)</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def rename_column(idf, list_of_cols, list_of_newcols, print_impact=False):
    """This function is used to rename the columns of the input data. Multiple columns can be renamed; however,
    the sequence they passed as an argument is critical and must be consistent between list_of_cols and
    list_of_newcols. It requires the following arguments:

    - **idf**: Input dataframe - **list_of_cols**: This argument, in a list format, is used to specify the columns
    required to be renamed in the input dataframe. Alternatively, instead of a list, columns can be specified in a
    single text format where different column names are separated by pipe delimiter “|” - **list_of_newcols**: This
    argument, in a list format, is used to specify the new column name, i.e. the first element in list_of_cols will
    be the original column name, and the corresponding first column in list_of_newcols will be the new column name. -
    **print_impact**: This argument is to compare column names before and after the operation.

    Parameters
    ----------
    idf :
        Input Dataframe
    list_of_cols :
        List of old column names e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
    list_of_newcols :
        List of corresponding new column names e.g., ["newcol1","newcol2"].
        Alternatively, new column names can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "newcol1|newcol2".
        First element in list_of_cols will be original column name,
        and corresponding first column in list_of_newcols will be new column name.
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(list_of_newcols, str):
        list_of_newcols = [x.strip() for x in list_of_newcols.split("|")]

    mapping = dict(zip(list_of_cols, list_of_newcols))
    odf = idf.select([F.col(i).alias(mapping.get(i, i)) for i in idf.columns])

    if print_impact:
        print("Before: \nNo. of Columns- ", len(idf.columns))
        print(idf.columns)
        print("After: \nNo. of Columns- ", len(odf.columns))
        print(odf.columns)
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_ingest.data_ingest.select_column"><code class="name flex">
<span>def <span class="ident">select_column</span></span>(<span>idf, list_of_cols, print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><p>This function is used to select specific columns from the input data. It is executed using select operation of
spark dataframe. It is advisable to use this function if the number of columns to select is lesser than the
number of columns to drop; otherwise, it is recommended to use delete_column. It requires the following arguments:</p>
<ul>
<li><strong>idf</strong>: Input dataframe - <strong>list_of_cols</strong>: This argument, in a list format, specifies the columns required to
be selected from the input dataframe. Alternatively, instead of list, columns can be specified in a single text
format where different column names are separated by pipe delimiter “|” - <strong>print_impact</strong>: This argument is to
compare number of columns before and after the operation.</li>
</ul>
<h2 id="parameters">Parameters</h2>
<p>idf :
Input Dataframe
list_of_cols :
List of columns to select e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
print_impact :
True, False (Default value = False)</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def select_column(idf, list_of_cols, print_impact=False):
    """This function is used to select specific columns from the input data. It is executed using select operation of
    spark dataframe. It is advisable to use this function if the number of columns to select is lesser than the
    number of columns to drop; otherwise, it is recommended to use delete_column. It requires the following arguments:

    - **idf**: Input dataframe - **list_of_cols**: This argument, in a list format, specifies the columns required to
    be selected from the input dataframe. Alternatively, instead of list, columns can be specified in a single text
    format where different column names are separated by pipe delimiter “|” - **print_impact**: This argument is to
    compare number of columns before and after the operation.

    Parameters
    ----------
    idf :
        Input Dataframe
    list_of_cols :
        List of columns to select e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    list_of_cols = list(set(list_of_cols))

    odf = idf.select(list_of_cols)

    if print_impact:
        print("Before: \nNo. of Columns-", len(idf.columns))
        print(idf.columns)
        print("\nAfter: \nNo. of Columns-", len(odf.columns))
        print(odf.columns)
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_ingest.data_ingest.write_dataset"><code class="name flex">
<span>def <span class="ident">write_dataset</span></span>(<span>idf, file_path, file_type, file_configs={}, column_order=[])</span>
</code></dt>
<dd>
<div class="desc"><p>This function saves the Spark DataFrame in the user-provided output path. Like read_dataset, this function is
based on the generic Save functionality of Spark SQL.
It requires the following arguments:</p>
<ul>
<li><strong>idf</strong>: Spark DataFrame to be saved - <strong>file_path</strong>: file (or directory) path where the output data is to be
saved. File path can be a local path or s3 path (when running with AWS cloud services) - <strong>file_type</strong>: file
format of the output data. Currently, we support CSV, Parquet or Avro. The Avro data source requires an external
package to run, which can be configured with spark-submit options (&ndash;packages
org.apache.spark:spark-avro_2.11:2.4.0). - <strong>file_configs</strong> (optional): The rest of the valid configuration can
be passed through this argument in a dictionary format, e.g., repartition, mode, compression, header, delimiter,
etc. All the key/value pairs written in this argument are passed as options to DataFrameWriter is available using
Dataset.write operator. If the number of repartitions mentioned through this argument is less than the existing
DataFrame partitions, then the coalesce operation is used instead of the repartition operation to make the
execution work. This is because the coalesce operation doesn’t</li>
</ul>
<p>Parameters ---------- idf : Input Dataframe file_path : Path to output data (directory or filename). Compatible
with local path and s3 path (when running in AWS environment). file_type : csv", "parquet", "avro". Avro data
source requires an external package to run, which can be configured with spark-submit (&ndash;packages
org.apache.spark:spark-avro_2.11:2.4.0). file_configs : This argument is passed in dictionary format as key/value
pairs. Some of the potential keys are header, delimiter, mode, compression, repartition. compression options -
uncompressed, gzip (doesn't work with avro), snappy (only valid for parquet) mode options - error (default),
overwrite, append repartition - None (automatic partitioning) or an integer value () e.g. {"header":"True",
"delimiter":",",'compression':'snappy','mode':'overwrite','repartition':'10'}. column_order : list of columns in
the order in which Dataframe is to be written. If None or [] is specified, then the default order is applied.</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def write_dataset(idf, file_path, file_type, file_configs={}, column_order=[]):
    """This function saves the Spark DataFrame in the user-provided output path. Like read_dataset, this function is
    based on the generic Save functionality of Spark SQL.  It requires the following arguments:

    - **idf**: Spark DataFrame to be saved - **file_path**: file (or directory) path where the output data is to be
    saved. File path can be a local path or s3 path (when running with AWS cloud services) - **file_type**: file
    format of the output data. Currently, we support CSV, Parquet or Avro. The Avro data source requires an external
    package to run, which can be configured with spark-submit options (--packages
    org.apache.spark:spark-avro_2.11:2.4.0). - **file_configs** (optional): The rest of the valid configuration can
    be passed through this argument in a dictionary format, e.g., repartition, mode, compression, header, delimiter,
    etc. All the key/value pairs written in this argument are passed as options to DataFrameWriter is available using
    Dataset.write operator. If the number of repartitions mentioned through this argument is less than the existing
    DataFrame partitions, then the coalesce operation is used instead of the repartition operation to make the
    execution work. This is because the coalesce operation doesn’t

    Parameters ---------- idf : Input Dataframe file_path : Path to output data (directory or filename). Compatible
    with local path and s3 path (when running in AWS environment). file_type : csv", "parquet", "avro". Avro data
    source requires an external package to run, which can be configured with spark-submit (--packages
    org.apache.spark:spark-avro_2.11:2.4.0). file_configs : This argument is passed in dictionary format as key/value
    pairs. Some of the potential keys are header, delimiter, mode, compression, repartition. compression options -
    uncompressed, gzip (doesn't work with avro), snappy (only valid for parquet) mode options - error (default),
    overwrite, append repartition - None (automatic partitioning) or an integer value () e.g. {"header":"True",
    "delimiter":",",'compression':'snappy','mode':'overwrite','repartition':'10'}. column_order : list of columns in
    the order in which Dataframe is to be written. If None or [] is specified, then the default order is applied.

    Returns
    -------

    """

    if not column_order:
        column_order = idf.columns
    else:
        if not isinstance(column_order, list):
            raise TypeError("Invalid input type for column_order argument")
        if len(column_order) != len(idf.columns):
            raise ValueError(
                "Count of column(s) specified in column_order argument do not match Dataframe"
            )
        diff_cols = [x for x in column_order if x not in set(idf.columns)]
        if diff_cols:
            raise ValueError(
                "Column(s) specified in column_order argument not found in Dataframe: "
                + str(diff_cols)
            )

    mode = file_configs["mode"] if "mode" in file_configs else "error"
    repartition = (
        int(file_configs["repartition"]) if "repartition" in file_configs else None
    )

    if repartition is None:
        idf.select(column_order).write.format(file_type).options(**file_configs).save(
            file_path, mode=mode
        )
    else:
        exist_parts = idf.rdd.getNumPartitions()
        req_parts = int(repartition)
        if req_parts > exist_parts:
            idf.select(column_order).repartition(req_parts).write.format(
                file_type
            ).options(**file_configs).save(file_path, mode=mode)
        else:
            idf.select(column_order).coalesce(req_parts).write.format(
                file_type
            ).options(**file_configs).save(file_path, mode=mode)
```
</pre>
</details>
</dd>
</dl>