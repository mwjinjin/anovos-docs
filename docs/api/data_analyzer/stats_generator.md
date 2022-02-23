# <code>stats_generator</code>
<p>This module generates all the descriptive statistics related to the ingested data. Descriptive statistics are
split into different metric types, and each function corresponds to one metric type. - global_summary -
measures_of_counts - measures_of_centralTendency - measures_of_cardinality - measures_of_dispersion -
measures_of_percentiles - measures_of_shape</p>
<p>Columns subjected to this analysis can be controlled by the right combination of arguments - list_of_cols and
drop_cols. All the above functions require the following arguments:</p>
<ul>
<li><strong>idf</strong>: Input dataframe - <strong>list_of_cols</strong>: This argument, in a list format, is used to specify the columns which
are subjected to the analysis in the input dataframe. Alternatively, instead of list, columns can be specified in a
single text format where different column names are separated by pipe delimiter “|”. The user can also use “all” as
an input to this argument to consider all columns. This is super useful instead of specifying all column names
manually. - <strong>drop_cols</strong>: In a list format, this argument is used to specify the columns that need to be dropped
from list_of_cols. Instead of a list, columns can be specified in a single text format where different column names
are separated by pipe delimiter “|”. It is most useful when coupled with the “all” value of list_of_cols,
when we need to consider all columns except a few handful of them. - <strong>print_impact</strong>: This argument is to print out
the statistics.</li>
</ul>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
# coding=utf-8
"""This module generates all the descriptive statistics related to the ingested data. Descriptive statistics are
split into different metric types, and each function corresponds to one metric type. - global_summary -
measures_of_counts - measures_of_centralTendency - measures_of_cardinality - measures_of_dispersion -
measures_of_percentiles - measures_of_shape

Columns subjected to this analysis can be controlled by the right combination of arguments - list_of_cols and
drop_cols. All the above functions require the following arguments:

- **idf**: Input dataframe - **list_of_cols**: This argument, in a list format, is used to specify the columns which
are subjected to the analysis in the input dataframe. Alternatively, instead of list, columns can be specified in a
single text format where different column names are separated by pipe delimiter “|”. The user can also use “all” as
an input to this argument to consider all columns. This is super useful instead of specifying all column names
manually. - **drop_cols**: In a list format, this argument is used to specify the columns that need to be dropped
from list_of_cols. Instead of a list, columns can be specified in a single text format where different column names
are separated by pipe delimiter “|”. It is most useful when coupled with the “all” value of list_of_cols,
when we need to consider all columns except a few handful of them. - **print_impact**: This argument is to print out
the statistics. """
import warnings

from pyspark.mllib.linalg import Vectors
from pyspark.mllib.stat import Statistics
from pyspark.sql import functions as F
from pyspark.sql import types as T

from anovos.shared.utils import transpose_dataframe, attributeType_segregation


def global_summary(spark, idf, list_of_cols="all", drop_cols=[], print_impact=True):
    """The global summary function computes the following universal statistics/metrics and returns a Spark DataFrame
    with schema – metric, value. - No. of rows - No. of columns - No. of categorical columns along with column names
    - No. of numerical columns along with the column names - No. of non-numerical non-categorical columns such as
    date type, array type etc. along with column names

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols.
        (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = True)

    Returns
    -------

    """
    if list_of_cols == "all":
        list_of_cols = idf.columns
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in idf.columns for x in list_of_cols) | (len(list_of_cols) == 0):
        raise TypeError("Invalid input for Column(s)")

    row_count = idf.count()
    col_count = len(list_of_cols)
    num_cols, cat_cols, other_cols = attributeType_segregation(idf.select(list_of_cols))
    numcol_count = len(num_cols)
    catcol_count = len(cat_cols)
    othercol_count = len(other_cols)
    if print_impact:
        print("No. of Rows: %s" % "{0:,}".format(row_count))
        print("No. of Columns: %s" % "{0:,}".format(col_count))
        print("Numerical Columns: %s" % "{0:,}".format(numcol_count))
        if numcol_count > 0:
            print(num_cols)
        print("Categorical Columns: %s" % "{0:,}".format(catcol_count))
        if catcol_count > 0:
            print(cat_cols)
        if othercol_count > 0:
            print("Other Columns: %s" % "{0:,}".format(othercol_count))
            print(other_cols)

    odf = spark.createDataFrame(
        [
            ["rows_count", str(row_count)],
            ["columns_count", str(col_count)],
            ["numcols_count", str(numcol_count)],
            ["numcols_name", ", ".join(num_cols)],
            ["catcols_count", str(catcol_count)],
            ["catcols_name", ", ".join(cat_cols)],
            ["othercols_count", str(othercol_count)],
            ["othercols_name", ", ".join(other_cols)],
        ],
        schema=["metric", "value"],
    )
    return odf


def missingCount_computation(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if list_of_cols == "all":
        num_cols, cat_cols, other_cols = attributeType_segregation(idf)
        list_of_cols = num_cols + cat_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in idf.columns for x in list_of_cols) | (len(list_of_cols) == 0):
        raise TypeError("Invalid input for Column(s)")

    idf_stats = idf.select(list_of_cols).summary("count")
    odf = (
        transpose_dataframe(idf_stats, "summary")
        .withColumn(
            "missing_count", F.lit(idf.count()) - F.col("count").cast(T.LongType())
        )
        .withColumn(
            "missing_pct", F.round(F.col("missing_count") / F.lit(idf.count()), 4)
        )
        .select(F.col("key").alias("attribute"), "missing_count", "missing_pct")
    )
    if print_impact:
        odf.show(len(list_of_cols))
    return odf


def nonzeroCount_computation(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of numerical columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all numerical columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    num_cols = attributeType_segregation(idf)[0]
    if list_of_cols == "all":
        list_of_cols = num_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in num_cols for x in list_of_cols):
        raise TypeError("Invalid input for Column(s)")

    if len(list_of_cols) == 0:
        warnings.warn(
            "No Non-Zero Count Computation - No numerical column(s) to analyze"
        )
        schema = T.StructType(
            [
                T.StructField("attribute", T.StringType(), True),
                T.StructField("nonzero_count", T.StringType(), True),
                T.StructField("nonzero_pct", T.StringType(), True),
            ]
        )
        odf = spark.sparkContext.emptyRDD().toDF(schema)
        return odf

    tmp = idf.select(list_of_cols).fillna(0).rdd.map(lambda row: Vectors.dense(row))
    nonzero_count = Statistics.colStats(tmp).numNonzeros()
    odf = spark.createDataFrame(
        zip(list_of_cols, [int(i) for i in nonzero_count]),
        schema=("attribute", "nonzero_count"),
    ).withColumn("nonzero_pct", F.round(F.col("nonzero_count") / F.lit(idf.count()), 4))
    if print_impact:
        odf.show(len(list_of_cols))
    return odf


def measures_of_counts(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """The Measures of Counts function computes different count metrics for each column (interchangeably called an
    attribute in the document). It returns a Spark DataFrame with schema – attribute, fill_count, fill_pct,
    missing_count, missing_pct, nonzero_count, nonzero_pct. - Fill Count/Rate is defined as number of rows with
    non-null values in a column both in terms of absolute count and its proportion to row count. It leverages count
    statistic from summary functionality of Spark SQL. - Missing Count/Rate is defined as null (or missing) values
    seen in a column both in terms of absolute count and its proportion to row count. It is directly derivable from
    Fill Count/Rate. - Non Zero Count/Rate is defined as non-zero values seen in a numerical column both in terms of
    absolute count and its proportion to row count. For categorical column, it will show null value. Also,
    it uses a subfunction nonzeroCount_computation, which is later called under measures_of_counts. Under the hood,
    it leverage Multivariate Statistical Summary of Spark MLlib.

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if list_of_cols == "all":
        num_cols, cat_cols, other_cols = attributeType_segregation(idf)
        list_of_cols = num_cols + cat_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))
    num_cols = attributeType_segregation(idf.select(list_of_cols))[0]

    if any(x not in idf.columns for x in list_of_cols) | (len(list_of_cols) == 0):
        raise TypeError("Invalid input for Column(s)")

    odf = (
        transpose_dataframe(idf.select(list_of_cols).summary("count"), "summary")
        .select(
            F.col("key").alias("attribute"),
            F.col("count").cast(T.LongType()).alias("fill_count"),
        )
        .withColumn("fill_pct", F.round(F.col("fill_count") / F.lit(idf.count()), 4))
        .withColumn(
            "missing_count", F.lit(idf.count()) - F.col("fill_count").cast(T.LongType())
        )
        .withColumn("missing_pct", F.round(1 - F.col("fill_pct"), 4))
        .join(nonzeroCount_computation(spark, idf, num_cols), "attribute", "full_outer")
    )

    if print_impact:
        odf.show(len(list_of_cols))
    return odf


def mode_computation(spark, idf, list_of_cols="all", drop_cols=[], print_impact=False):
    """

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of Discrete (Categorical + Integer) columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all discrete columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if list_of_cols == "all":
        num_cols, cat_cols, other_cols = attributeType_segregation(idf)
        list_of_cols = num_cols + cat_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))
    for i in idf.select(list_of_cols).dtypes:
        if i[1] not in ("string", "int", "bigint", "long"):
            list_of_cols.remove(i[0])

    if any(x not in idf.columns for x in list_of_cols):
        raise TypeError("Invalid input for Column(s)")

    if len(list_of_cols) == 0:
        warnings.warn("No Mode Computation - No discrete column(s) to analyze")
        schema = T.StructType(
            [
                T.StructField("attribute", T.StringType(), True),
                T.StructField("mode", T.StringType(), True),
                T.StructField("mode_rows", T.StringType(), True),
            ]
        )
        odf = spark.sparkContext.emptyRDD().toDF(schema)
        return odf

    mode = [
        list(
            idf.select(i)
            .dropna()
            .groupby(i)
            .count()
            .orderBy("count", ascending=False)
            .first()
            or [None, None]
        )
        for i in list_of_cols
    ]
    mode = [(str(i), str(j)) for i, j in mode]

    odf = spark.createDataFrame(
        zip(list_of_cols, mode), schema=("attribute", "metric")
    ).select(
        "attribute",
        (F.col("metric")["_1"]).alias("mode"),
        (F.col("metric")["_2"]).cast("long").alias("mode_rows"),
    )

    if print_impact:
        odf.show(len(list_of_cols))
    return odf


def measures_of_centralTendency(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """The Measures of Central Tendency function provides summary statistics that represents the centre point or most
    likely value of an attribute. It returns a Spark DataFrame with schema – attribute, mean, median, mode, mode_pct.

    - Mean is arithmetic average of a column i.e. sum of all values seen in the column divided by the number of rows.
    It leverage mean statistic from summary functionality of Spark SQL. - Median is 50th percentile or middle value
    in a column when the values are arranged in ascending or descending order. It leverage ‘50%’ statistic from
    summary functionality of Spark SQL. - Mode is most frequently seen value in a column. Mode is calculated only for
    discrete columns (categorical + Integer/Long columns) - Mode Pct is defined as % of rows seen with Mode value.
    Mode Pct is calculated only for discrete columns (categorical + Integer/Long columns)

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if list_of_cols == "all":
        num_cols, cat_cols, other_cols = attributeType_segregation(idf)
        list_of_cols = num_cols + cat_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in idf.columns for x in list_of_cols) | (len(list_of_cols) == 0):
        raise TypeError("Invalid input for Column(s)")

    odf = (
        transpose_dataframe(
            idf.select(list_of_cols).summary("mean", "50%", "count"), "summary"
        )
        .withColumn("mean", F.round(F.col("mean").cast(T.DoubleType()), 4))
        .withColumn("median", F.round(F.col("50%").cast(T.DoubleType()), 4))
        .withColumnRenamed("key", "attribute")
        .join(mode_computation(spark, idf, list_of_cols), "attribute", "full_outer")
        .withColumn(
            "mode_pct",
            F.round(F.col("mode_rows") / F.col("count").cast(T.DoubleType()), 4),
        )
        .select("attribute", "mean", "median", "mode", "mode_rows", "mode_pct")
    )

    if print_impact:
        odf.show(len(list_of_cols))
    return odf


def uniqueCount_computation(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of Discrete (Categorical + Integer) columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all discrete columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if list_of_cols == "all":
        list_of_cols = []
        for i in idf.dtypes:
            if i[1] in ("string", "int", "bigint", "long"):
                list_of_cols.append(i[0])
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in idf.columns for x in list_of_cols):
        raise TypeError("Invalid input for Column(s)")

    if len(list_of_cols) == 0:
        warnings.warn("No Unique Count Computation - No discrete column(s) to analyze")
        schema = T.StructType(
            [
                T.StructField("attribute", T.StringType(), True),
                T.StructField("unique_values", T.StringType(), True),
            ]
        )
        odf = spark.sparkContext.emptyRDD().toDF(schema)
        return odf

    uniquevalue_count = idf.agg(
        *(F.countDistinct(F.col(i)).alias(i) for i in list_of_cols)
    )
    odf = spark.createDataFrame(
        zip(list_of_cols, uniquevalue_count.rdd.map(list).collect()[0]),
        schema=("attribute", "unique_values"),
    )
    if print_impact:
        odf.show(len(list_of_cols))
    return odf


def measures_of_cardinality(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """The Measures of Cardinality function provides statistics that are related to unique values seen in an
    attribute. These statistics are calculated only for discrete columns (categorical + Integer/Long columns). It
    returns a Spark Dataframe with schema – attribute, unique_values, IDness.

    - Unique Value is defined as a distinct value count of a column. It relies on a subfunction
    uniqueCount_computation for its computation and leverages the countDistinct functionality of Spark SQL. - IDness
    is calculated as Unique Values divided by non-null values seen in a column. Non-null values count is used instead
    of total count because too many null values can give misleading results even if the column have all unique values
    (except null). It uses subfunctions - uniqueCount_computation and missingCount_computation.

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of Discrete (Categorical + Integer) columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all discrete columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if list_of_cols == "all":
        list_of_cols = []
        for i in idf.dtypes:
            if i[1] in ("string", "int", "bigint", "long"):
                list_of_cols.append(i[0])
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in idf.columns for x in list_of_cols):
        raise TypeError("Invalid input for Column(s)")

    if len(list_of_cols) == 0:
        warnings.warn("No Cardinality Computation - No discrete column(s) to analyze")
        schema = T.StructType(
            [
                T.StructField("attribute", T.StringType(), True),
                T.StructField("unique_values", T.StringType(), True),
                T.StructField("IDness", T.StringType(), True),
            ]
        )
        odf = spark.sparkContext.emptyRDD().toDF(schema)
        return odf

    odf = (
        uniqueCount_computation(spark, idf, list_of_cols)
        .join(
            missingCount_computation(spark, idf, list_of_cols),
            "attribute",
            "full_outer",
        )
        .withColumn(
            "IDness",
            F.round(
                F.col("unique_values") / (F.lit(idf.count()) - F.col("missing_count")),
                4,
            ),
        )
        .select("attribute", "unique_values", "IDness")
    )
    if print_impact:
        odf.show(len(list_of_cols))
    return odf


def measures_of_dispersion(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """The Measures of Dispersion function provides statistics that describe the spread of a numerical attribute.
    Alternatively, these statistics are also known as measures of spread. It returns a Spark DataFrame with schema –
    attribute, stddev, variance, cov, IQR, range.

    - Standard Deviation (stddev) measures how concentrated an attribute is around the mean or average and
    mathematically computed as  below. It leverages ‘stddev’ statistic from summary functionality of Spark SQL.

            s= X- X2n -1

            where:

            ` `X is an attribute value
            X is attribute mean
            n is no. of rows

    - Variance is the squared value of Standard Deviation. - Coefficient of Variance (cov) is computed as ratio of
    Standard Deviation & Mean. It leverages ‘stddev’ and ‘mean’ statistic from the summary functionality of Spark
    SQL. - Interquartile Range (IQR): It describes the difference between the third quartile (75th percentile) and
    the first quartile (25th percentile), telling us about the range where middle half values are seen. It leverage
    ‘25%’ and ‘75%’ statistics from the summary functionality of Spark SQL. - Range is simply the difference between
    the maximum value and the minimum value. It leverage ‘min’ and ‘max’ statistics from the summary functionality of
    Spark

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of numerical columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all numerical columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    num_cols = attributeType_segregation(idf)[0]
    if list_of_cols == "all":
        list_of_cols = num_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in num_cols for x in list_of_cols):
        raise TypeError("Invalid input for Column(s)")
    if len(list_of_cols) == 0:
        warnings.warn("No Dispersion Computation - No numerical column(s) to analyze")
        schema = T.StructType(
            [
                T.StructField("attribute", T.StringType(), True),
                T.StructField("stddev", T.StringType(), True),
                T.StructField("variance", T.StringType(), True),
                T.StructField("cov", T.StringType(), True),
                T.StructField("IQR", T.StringType(), True),
                T.StructField("range", T.StringType(), True),
            ]
        )
        odf = spark.sparkContext.emptyRDD().toDF(schema)
        return odf

    odf = (
        transpose_dataframe(
            idf.select(list_of_cols).summary(
                "stddev", "min", "max", "mean", "25%", "75%"
            ),
            "summary",
        )
        .withColumn("stddev", F.round(F.col("stddev").cast(T.DoubleType()), 4))
        .withColumn("variance", F.round(F.col("stddev") * F.col("stddev"), 4))
        .withColumn("range", F.round(F.col("max") - F.col("min"), 4))
        .withColumn("cov", F.round(F.col("stddev") / F.col("mean"), 4))
        .withColumn("IQR", F.round(F.col("75%") - F.col("25%"), 4))
        .select(
            F.col("key").alias("attribute"), "stddev", "variance", "cov", "IQR", "range"
        )
    )
    if print_impact:
        odf.show(len(list_of_cols))
    return odf


def measures_of_percentiles(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """The Measures of Percentiles function provides statistics at different percentiles. Nth percentile can be
    interpreted as N% of rows having values lesser than or equal to Nth percentile value. It is prominently used for
    quick detection of skewness or outlier. Alternatively, these statistics are also known as measures of position.
    These statistics are computed only for numerical attributes.

    It returns a Spark Dataframe with schema – attribute, min, 1%, 5%, 10%, 25%, 50%, 75%, 90%, 95%, 99%,
    max. It leverage ‘N%’ statistics from summary functionality of Spark SQL where N is 0 for min and 100 for max.

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of numerical columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all numerical columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    num_cols = attributeType_segregation(idf)[0]
    if list_of_cols == "all":
        list_of_cols = num_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in num_cols for x in list_of_cols):
        raise TypeError("Invalid input for Column(s)")
    if len(list_of_cols) == 0:
        warnings.warn("No Percentiles Computation - No numerical column(s) to analyze")
        schema = T.StructType(
            [
                T.StructField("attribute", T.StringType(), True),
                T.StructField("min", T.StringType(), True),
                T.StructField("1%", T.StringType(), True),
                T.StructField("5%", T.StringType(), True),
                T.StructField("10%", T.StringType(), True),
                T.StructField("25%", T.StringType(), True),
                T.StructField("50%", T.StringType(), True),
                T.StructField("75%", T.StringType(), True),
                T.StructField("90%", T.StringType(), True),
                T.StructField("95%", T.StringType(), True),
                T.StructField("99%", T.StringType(), True),
                T.StructField("max", T.StringType(), True),
            ]
        )
        odf = spark.sparkContext.emptyRDD().toDF(schema)
        return odf

    stats = ["min", "1%", "5%", "10%", "25%", "50%", "75%", "90%", "95%", "99%", "max"]
    odf = transpose_dataframe(
        idf.select(list_of_cols).summary(*stats), "summary"
    ).withColumnRenamed("key", "attribute")
    for i in odf.columns:
        if i != "attribute":
            odf = odf.withColumn(i, F.round(F.col(i).cast("Double"), 4))
    odf = odf.select(["attribute"] + stats)
    if print_impact:
        odf.show(len(list_of_cols))
    return odf


def measures_of_shape(spark, idf, list_of_cols="all", drop_cols=[], print_impact=False):
    """The Measures of Shapes function provides statistics related to the shape of an attribute's distribution.
    Alternatively, these statistics are also known as measures of the moment and are computed only for numerical
    attributes. It returns a Spark Dataframe with schema – attribute, skewness, kurtosis.

    - Skewness describes how much-skewed values are relative to a perfect bell curve observed in normal distribution
    and the direction of skew. If the majority of the values are at the left and the right tail is longer,
    we say that the distribution is skewed right or positively skewed; if the peak is toward the right and the left
    tail is longer, we say that the distribution is skewed left or negatively skewed. It leverage skewness
    functionality of Spark SQL. - (Excess) Kurtosis describes how tall and sharp the central peak is relative to a
    perfect bell curve observed in the normal distribution. The reference standard is a normal distribution,
    which has a kurtosis of 3. In token of this, often, the excess kurtosis is presented: excess kurtosis is simply
    kurtosis−3. Higher (positive) values indicate a higher, sharper peak; lower (negative) values indicate a less
    distinct peak. It leverages kurtosis functionality of Spark SQL.

    Parameters
    ----------
    idf :
        Input Dataframe
    list_of_cols :
        List of numerical columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all numerical columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)
    spark :


    Returns
    -------

    """

    num_cols = attributeType_segregation(idf)[0]
    if list_of_cols == "all":
        list_of_cols = num_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in num_cols for x in list_of_cols):
        raise TypeError("Invalid input for Column(s)")
    if len(list_of_cols) == 0:
        warnings.warn(
            "No Skewness/Kurtosis Computation - No numerical column(s) to analyze"
        )
        schema = T.StructType(
            [
                T.StructField("attribute", T.StringType(), True),
                T.StructField("skewness", T.StringType(), True),
                T.StructField("kurtosis", T.StringType(), True),
            ]
        )
        odf = spark.sparkContext.emptyRDD().toDF(schema)
        return odf

    shapes = []
    for i in list_of_cols:
        s, k = idf.select(F.skewness(i), F.kurtosis(i)).first()
        shapes.append([i, s, k])
    odf = (
        spark.createDataFrame(shapes, schema=("attribute", "skewness", "kurtosis"))
        .withColumn("skewness", F.round(F.col("skewness"), 4))
        .withColumn("kurtosis", F.round(F.col("kurtosis"), 4))
    )
    if print_impact:
        odf.show(len(list_of_cols))
    return odf
```
</pre>
</details>
## Functions
<dl>
<dt id="anovos.data_analyzer.stats_generator.global_summary"><code class="name flex">
<span>def <span class="ident">global_summary</span></span>(<span>spark, idf, list_of_cols='all', drop_cols=[], print_impact=True)</span>
</code></dt>
<dd>
<div class="desc"><p>The global summary function computes the following universal statistics/metrics and returns a Spark DataFrame
with schema – metric, value. - No. of rows - No. of columns - No. of categorical columns along with column names
- No. of numerical columns along with the column names - No. of non-numerical non-categorical columns such as
date type, array type etc. along with column names</p>
<h2 id="parameters">Parameters</h2>
<p>spark :
Spark Session
idf :
Input Dataframe
list_of_cols :
List of columns to analyse e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
"all" can be passed to include all columns for analysis.
Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols.
(Default value = "all")
drop_cols :
List of columns to be dropped e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
print_impact :
True, False (Default value = True)</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def global_summary(spark, idf, list_of_cols="all", drop_cols=[], print_impact=True):
    """The global summary function computes the following universal statistics/metrics and returns a Spark DataFrame
    with schema – metric, value. - No. of rows - No. of columns - No. of categorical columns along with column names
    - No. of numerical columns along with the column names - No. of non-numerical non-categorical columns such as
    date type, array type etc. along with column names

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols.
        (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = True)

    Returns
    -------

    """
    if list_of_cols == "all":
        list_of_cols = idf.columns
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in idf.columns for x in list_of_cols) | (len(list_of_cols) == 0):
        raise TypeError("Invalid input for Column(s)")

    row_count = idf.count()
    col_count = len(list_of_cols)
    num_cols, cat_cols, other_cols = attributeType_segregation(idf.select(list_of_cols))
    numcol_count = len(num_cols)
    catcol_count = len(cat_cols)
    othercol_count = len(other_cols)
    if print_impact:
        print("No. of Rows: %s" % "{0:,}".format(row_count))
        print("No. of Columns: %s" % "{0:,}".format(col_count))
        print("Numerical Columns: %s" % "{0:,}".format(numcol_count))
        if numcol_count > 0:
            print(num_cols)
        print("Categorical Columns: %s" % "{0:,}".format(catcol_count))
        if catcol_count > 0:
            print(cat_cols)
        if othercol_count > 0:
            print("Other Columns: %s" % "{0:,}".format(othercol_count))
            print(other_cols)

    odf = spark.createDataFrame(
        [
            ["rows_count", str(row_count)],
            ["columns_count", str(col_count)],
            ["numcols_count", str(numcol_count)],
            ["numcols_name", ", ".join(num_cols)],
            ["catcols_count", str(catcol_count)],
            ["catcols_name", ", ".join(cat_cols)],
            ["othercols_count", str(othercol_count)],
            ["othercols_name", ", ".join(other_cols)],
        ],
        schema=["metric", "value"],
    )
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_analyzer.stats_generator.measures_of_cardinality"><code class="name flex">
<span>def <span class="ident">measures_of_cardinality</span></span>(<span>spark, idf, list_of_cols='all', drop_cols=[], print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><p>The Measures of Cardinality function provides statistics that are related to unique values seen in an
attribute. These statistics are calculated only for discrete columns (categorical + Integer/Long columns). It
returns a Spark Dataframe with schema – attribute, unique_values, IDness.</p>
<ul>
<li>Unique Value is defined as a distinct value count of a column. It relies on a subfunction
uniqueCount_computation for its computation and leverages the countDistinct functionality of Spark SQL. - IDness
is calculated as Unique Values divided by non-null values seen in a column. Non-null values count is used instead
of total count because too many null values can give misleading results even if the column have all unique values
(except null). It uses subfunctions - uniqueCount_computation and missingCount_computation.</li>
</ul>
<h2 id="parameters">Parameters</h2>
<p>spark :
Spark Session
idf :
Input Dataframe
list_of_cols :
List of Discrete (Categorical + Integer) columns to analyse e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
"all" can be passed to include all discrete columns for analysis.
Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
drop_cols :
List of columns to be dropped e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
print_impact :
True, False (Default value = False)</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def measures_of_cardinality(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """The Measures of Cardinality function provides statistics that are related to unique values seen in an
    attribute. These statistics are calculated only for discrete columns (categorical + Integer/Long columns). It
    returns a Spark Dataframe with schema – attribute, unique_values, IDness.

    - Unique Value is defined as a distinct value count of a column. It relies on a subfunction
    uniqueCount_computation for its computation and leverages the countDistinct functionality of Spark SQL. - IDness
    is calculated as Unique Values divided by non-null values seen in a column. Non-null values count is used instead
    of total count because too many null values can give misleading results even if the column have all unique values
    (except null). It uses subfunctions - uniqueCount_computation and missingCount_computation.

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of Discrete (Categorical + Integer) columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all discrete columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if list_of_cols == "all":
        list_of_cols = []
        for i in idf.dtypes:
            if i[1] in ("string", "int", "bigint", "long"):
                list_of_cols.append(i[0])
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in idf.columns for x in list_of_cols):
        raise TypeError("Invalid input for Column(s)")

    if len(list_of_cols) == 0:
        warnings.warn("No Cardinality Computation - No discrete column(s) to analyze")
        schema = T.StructType(
            [
                T.StructField("attribute", T.StringType(), True),
                T.StructField("unique_values", T.StringType(), True),
                T.StructField("IDness", T.StringType(), True),
            ]
        )
        odf = spark.sparkContext.emptyRDD().toDF(schema)
        return odf

    odf = (
        uniqueCount_computation(spark, idf, list_of_cols)
        .join(
            missingCount_computation(spark, idf, list_of_cols),
            "attribute",
            "full_outer",
        )
        .withColumn(
            "IDness",
            F.round(
                F.col("unique_values") / (F.lit(idf.count()) - F.col("missing_count")),
                4,
            ),
        )
        .select("attribute", "unique_values", "IDness")
    )
    if print_impact:
        odf.show(len(list_of_cols))
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_analyzer.stats_generator.measures_of_centralTendency"><code class="name flex">
<span>def <span class="ident">measures_of_centralTendency</span></span>(<span>spark, idf, list_of_cols='all', drop_cols=[], print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><p>The Measures of Central Tendency function provides summary statistics that represents the centre point or most
likely value of an attribute. It returns a Spark DataFrame with schema – attribute, mean, median, mode, mode_pct.</p>
<ul>
<li>Mean is arithmetic average of a column i.e. sum of all values seen in the column divided by the number of rows.
It leverage mean statistic from summary functionality of Spark SQL. - Median is 50th percentile or middle value
in a column when the values are arranged in ascending or descending order. It leverage ‘50%’ statistic from
summary functionality of Spark SQL. - Mode is most frequently seen value in a column. Mode is calculated only for
discrete columns (categorical + Integer/Long columns) - Mode Pct is defined as % of rows seen with Mode value.
Mode Pct is calculated only for discrete columns (categorical + Integer/Long columns)</li>
</ul>
<h2 id="parameters">Parameters</h2>
<p>spark :
Spark Session
idf :
Input Dataframe
list_of_cols :
List of columns to analyse e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
"all" can be passed to include all columns for analysis.
Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
drop_cols :
List of columns to be dropped e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
print_impact :
True, False (Default value = False)</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def measures_of_centralTendency(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """The Measures of Central Tendency function provides summary statistics that represents the centre point or most
    likely value of an attribute. It returns a Spark DataFrame with schema – attribute, mean, median, mode, mode_pct.

    - Mean is arithmetic average of a column i.e. sum of all values seen in the column divided by the number of rows.
    It leverage mean statistic from summary functionality of Spark SQL. - Median is 50th percentile or middle value
    in a column when the values are arranged in ascending or descending order. It leverage ‘50%’ statistic from
    summary functionality of Spark SQL. - Mode is most frequently seen value in a column. Mode is calculated only for
    discrete columns (categorical + Integer/Long columns) - Mode Pct is defined as % of rows seen with Mode value.
    Mode Pct is calculated only for discrete columns (categorical + Integer/Long columns)

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if list_of_cols == "all":
        num_cols, cat_cols, other_cols = attributeType_segregation(idf)
        list_of_cols = num_cols + cat_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in idf.columns for x in list_of_cols) | (len(list_of_cols) == 0):
        raise TypeError("Invalid input for Column(s)")

    odf = (
        transpose_dataframe(
            idf.select(list_of_cols).summary("mean", "50%", "count"), "summary"
        )
        .withColumn("mean", F.round(F.col("mean").cast(T.DoubleType()), 4))
        .withColumn("median", F.round(F.col("50%").cast(T.DoubleType()), 4))
        .withColumnRenamed("key", "attribute")
        .join(mode_computation(spark, idf, list_of_cols), "attribute", "full_outer")
        .withColumn(
            "mode_pct",
            F.round(F.col("mode_rows") / F.col("count").cast(T.DoubleType()), 4),
        )
        .select("attribute", "mean", "median", "mode", "mode_rows", "mode_pct")
    )

    if print_impact:
        odf.show(len(list_of_cols))
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_analyzer.stats_generator.measures_of_counts"><code class="name flex">
<span>def <span class="ident">measures_of_counts</span></span>(<span>spark, idf, list_of_cols='all', drop_cols=[], print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><p>The Measures of Counts function computes different count metrics for each column (interchangeably called an
attribute in the document). It returns a Spark DataFrame with schema – attribute, fill_count, fill_pct,
missing_count, missing_pct, nonzero_count, nonzero_pct. - Fill Count/Rate is defined as number of rows with
non-null values in a column both in terms of absolute count and its proportion to row count. It leverages count
statistic from summary functionality of Spark SQL. - Missing Count/Rate is defined as null (or missing) values
seen in a column both in terms of absolute count and its proportion to row count. It is directly derivable from
Fill Count/Rate. - Non Zero Count/Rate is defined as non-zero values seen in a numerical column both in terms of
absolute count and its proportion to row count. For categorical column, it will show null value. Also,
it uses a subfunction nonzeroCount_computation, which is later called under measures_of_counts. Under the hood,
it leverage Multivariate Statistical Summary of Spark MLlib.</p>
<h2 id="parameters">Parameters</h2>
<p>spark :
Spark Session
idf :
Input Dataframe
list_of_cols :
List of columns to analyse e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
"all" can be passed to include all columns for analysis.
Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
drop_cols :
List of columns to be dropped e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
print_impact :
True, False (Default value = False)</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def measures_of_counts(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """The Measures of Counts function computes different count metrics for each column (interchangeably called an
    attribute in the document). It returns a Spark DataFrame with schema – attribute, fill_count, fill_pct,
    missing_count, missing_pct, nonzero_count, nonzero_pct. - Fill Count/Rate is defined as number of rows with
    non-null values in a column both in terms of absolute count and its proportion to row count. It leverages count
    statistic from summary functionality of Spark SQL. - Missing Count/Rate is defined as null (or missing) values
    seen in a column both in terms of absolute count and its proportion to row count. It is directly derivable from
    Fill Count/Rate. - Non Zero Count/Rate is defined as non-zero values seen in a numerical column both in terms of
    absolute count and its proportion to row count. For categorical column, it will show null value. Also,
    it uses a subfunction nonzeroCount_computation, which is later called under measures_of_counts. Under the hood,
    it leverage Multivariate Statistical Summary of Spark MLlib.

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if list_of_cols == "all":
        num_cols, cat_cols, other_cols = attributeType_segregation(idf)
        list_of_cols = num_cols + cat_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))
    num_cols = attributeType_segregation(idf.select(list_of_cols))[0]

    if any(x not in idf.columns for x in list_of_cols) | (len(list_of_cols) == 0):
        raise TypeError("Invalid input for Column(s)")

    odf = (
        transpose_dataframe(idf.select(list_of_cols).summary("count"), "summary")
        .select(
            F.col("key").alias("attribute"),
            F.col("count").cast(T.LongType()).alias("fill_count"),
        )
        .withColumn("fill_pct", F.round(F.col("fill_count") / F.lit(idf.count()), 4))
        .withColumn(
            "missing_count", F.lit(idf.count()) - F.col("fill_count").cast(T.LongType())
        )
        .withColumn("missing_pct", F.round(1 - F.col("fill_pct"), 4))
        .join(nonzeroCount_computation(spark, idf, num_cols), "attribute", "full_outer")
    )

    if print_impact:
        odf.show(len(list_of_cols))
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_analyzer.stats_generator.measures_of_dispersion"><code class="name flex">
<span>def <span class="ident">measures_of_dispersion</span></span>(<span>spark, idf, list_of_cols='all', drop_cols=[], print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><p>The Measures of Dispersion function provides statistics that describe the spread of a numerical attribute.
Alternatively, these statistics are also known as measures of spread. It returns a Spark DataFrame with schema –
attribute, stddev, variance, cov, IQR, range.</p>
<ul>
<li>
<p>Standard Deviation (stddev) measures how concentrated an attribute is around the mean or average and
mathematically computed as
below. It leverages ‘stddev’ statistic from summary functionality of Spark SQL.</p>
<pre><code>s= X- X2n -1

where:

&lt;code&gt; &lt;/code&gt;X is an attribute value
X is attribute mean
n is no. of rows
</code></pre>
</li>
<li>
<p>Variance is the squared value of Standard Deviation. - Coefficient of Variance (cov) is computed as ratio of
Standard Deviation &amp; Mean. It leverages ‘stddev’ and ‘mean’ statistic from the summary functionality of Spark
SQL. - Interquartile Range (IQR): It describes the difference between the third quartile (75th percentile) and
the first quartile (25th percentile), telling us about the range where middle half values are seen. It leverage
‘25%’ and ‘75%’ statistics from the summary functionality of Spark SQL. - Range is simply the difference between
the maximum value and the minimum value. It leverage ‘min’ and ‘max’ statistics from the summary functionality of
Spark</p>
</li>
</ul>
<h2 id="parameters">Parameters</h2>
<p>spark :
Spark Session
idf :
Input Dataframe
list_of_cols :
List of numerical columns to analyse e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
"all" can be passed to include all numerical columns for analysis.
Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
drop_cols :
List of columns to be dropped e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
print_impact :
True, False (Default value = False)</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def measures_of_dispersion(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """The Measures of Dispersion function provides statistics that describe the spread of a numerical attribute.
    Alternatively, these statistics are also known as measures of spread. It returns a Spark DataFrame with schema –
    attribute, stddev, variance, cov, IQR, range.

    - Standard Deviation (stddev) measures how concentrated an attribute is around the mean or average and
    mathematically computed as  below. It leverages ‘stddev’ statistic from summary functionality of Spark SQL.

            s= X- X2n -1

            where:

            ` `X is an attribute value
            X is attribute mean
            n is no. of rows

    - Variance is the squared value of Standard Deviation. - Coefficient of Variance (cov) is computed as ratio of
    Standard Deviation & Mean. It leverages ‘stddev’ and ‘mean’ statistic from the summary functionality of Spark
    SQL. - Interquartile Range (IQR): It describes the difference between the third quartile (75th percentile) and
    the first quartile (25th percentile), telling us about the range where middle half values are seen. It leverage
    ‘25%’ and ‘75%’ statistics from the summary functionality of Spark SQL. - Range is simply the difference between
    the maximum value and the minimum value. It leverage ‘min’ and ‘max’ statistics from the summary functionality of
    Spark

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of numerical columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all numerical columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    num_cols = attributeType_segregation(idf)[0]
    if list_of_cols == "all":
        list_of_cols = num_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in num_cols for x in list_of_cols):
        raise TypeError("Invalid input for Column(s)")
    if len(list_of_cols) == 0:
        warnings.warn("No Dispersion Computation - No numerical column(s) to analyze")
        schema = T.StructType(
            [
                T.StructField("attribute", T.StringType(), True),
                T.StructField("stddev", T.StringType(), True),
                T.StructField("variance", T.StringType(), True),
                T.StructField("cov", T.StringType(), True),
                T.StructField("IQR", T.StringType(), True),
                T.StructField("range", T.StringType(), True),
            ]
        )
        odf = spark.sparkContext.emptyRDD().toDF(schema)
        return odf

    odf = (
        transpose_dataframe(
            idf.select(list_of_cols).summary(
                "stddev", "min", "max", "mean", "25%", "75%"
            ),
            "summary",
        )
        .withColumn("stddev", F.round(F.col("stddev").cast(T.DoubleType()), 4))
        .withColumn("variance", F.round(F.col("stddev") * F.col("stddev"), 4))
        .withColumn("range", F.round(F.col("max") - F.col("min"), 4))
        .withColumn("cov", F.round(F.col("stddev") / F.col("mean"), 4))
        .withColumn("IQR", F.round(F.col("75%") - F.col("25%"), 4))
        .select(
            F.col("key").alias("attribute"), "stddev", "variance", "cov", "IQR", "range"
        )
    )
    if print_impact:
        odf.show(len(list_of_cols))
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_analyzer.stats_generator.measures_of_percentiles"><code class="name flex">
<span>def <span class="ident">measures_of_percentiles</span></span>(<span>spark, idf, list_of_cols='all', drop_cols=[], print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><p>The Measures of Percentiles function provides statistics at different percentiles. Nth percentile can be
interpreted as N% of rows having values lesser than or equal to Nth percentile value. It is prominently used for
quick detection of skewness or outlier. Alternatively, these statistics are also known as measures of position.
These statistics are computed only for numerical attributes.</p>
<p>It returns a Spark Dataframe with schema – attribute, min, 1%, 5%, 10%, 25%, 50%, 75%, 90%, 95%, 99%,
max. It leverage ‘N%’ statistics from summary functionality of Spark SQL where N is 0 for min and 100 for max.</p>
<h2 id="parameters">Parameters</h2>
<p>spark :
Spark Session
idf :
Input Dataframe
list_of_cols :
List of numerical columns to analyse e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
"all" can be passed to include all numerical columns for analysis.
Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
drop_cols :
List of columns to be dropped e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
print_impact :
True, False (Default value = False)</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def measures_of_percentiles(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """The Measures of Percentiles function provides statistics at different percentiles. Nth percentile can be
    interpreted as N% of rows having values lesser than or equal to Nth percentile value. It is prominently used for
    quick detection of skewness or outlier. Alternatively, these statistics are also known as measures of position.
    These statistics are computed only for numerical attributes.

    It returns a Spark Dataframe with schema – attribute, min, 1%, 5%, 10%, 25%, 50%, 75%, 90%, 95%, 99%,
    max. It leverage ‘N%’ statistics from summary functionality of Spark SQL where N is 0 for min and 100 for max.

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of numerical columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all numerical columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    num_cols = attributeType_segregation(idf)[0]
    if list_of_cols == "all":
        list_of_cols = num_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in num_cols for x in list_of_cols):
        raise TypeError("Invalid input for Column(s)")
    if len(list_of_cols) == 0:
        warnings.warn("No Percentiles Computation - No numerical column(s) to analyze")
        schema = T.StructType(
            [
                T.StructField("attribute", T.StringType(), True),
                T.StructField("min", T.StringType(), True),
                T.StructField("1%", T.StringType(), True),
                T.StructField("5%", T.StringType(), True),
                T.StructField("10%", T.StringType(), True),
                T.StructField("25%", T.StringType(), True),
                T.StructField("50%", T.StringType(), True),
                T.StructField("75%", T.StringType(), True),
                T.StructField("90%", T.StringType(), True),
                T.StructField("95%", T.StringType(), True),
                T.StructField("99%", T.StringType(), True),
                T.StructField("max", T.StringType(), True),
            ]
        )
        odf = spark.sparkContext.emptyRDD().toDF(schema)
        return odf

    stats = ["min", "1%", "5%", "10%", "25%", "50%", "75%", "90%", "95%", "99%", "max"]
    odf = transpose_dataframe(
        idf.select(list_of_cols).summary(*stats), "summary"
    ).withColumnRenamed("key", "attribute")
    for i in odf.columns:
        if i != "attribute":
            odf = odf.withColumn(i, F.round(F.col(i).cast("Double"), 4))
    odf = odf.select(["attribute"] + stats)
    if print_impact:
        odf.show(len(list_of_cols))
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_analyzer.stats_generator.measures_of_shape"><code class="name flex">
<span>def <span class="ident">measures_of_shape</span></span>(<span>spark, idf, list_of_cols='all', drop_cols=[], print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><p>The Measures of Shapes function provides statistics related to the shape of an attribute's distribution.
Alternatively, these statistics are also known as measures of the moment and are computed only for numerical
attributes. It returns a Spark Dataframe with schema – attribute, skewness, kurtosis.</p>
<ul>
<li>Skewness describes how much-skewed values are relative to a perfect bell curve observed in normal distribution
and the direction of skew. If the majority of the values are at the left and the right tail is longer,
we say that the distribution is skewed right or positively skewed; if the peak is toward the right and the left
tail is longer, we say that the distribution is skewed left or negatively skewed. It leverage skewness
functionality of Spark SQL. - (Excess) Kurtosis describes how tall and sharp the central peak is relative to a
perfect bell curve observed in the normal distribution. The reference standard is a normal distribution,
which has a kurtosis of 3. In token of this, often, the excess kurtosis is presented: excess kurtosis is simply
kurtosis−3. Higher (positive) values indicate a higher, sharper peak; lower (negative) values indicate a less
distinct peak. It leverages kurtosis functionality of Spark SQL.</li>
</ul>
<h2 id="parameters">Parameters</h2>
<p>idf :
Input Dataframe
list_of_cols :
List of numerical columns to analyse e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
"all" can be passed to include all numerical columns for analysis.
Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
drop_cols :
List of columns to be dropped e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
print_impact :
True, False (Default value = False)
spark :</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def measures_of_shape(spark, idf, list_of_cols="all", drop_cols=[], print_impact=False):
    """The Measures of Shapes function provides statistics related to the shape of an attribute's distribution.
    Alternatively, these statistics are also known as measures of the moment and are computed only for numerical
    attributes. It returns a Spark Dataframe with schema – attribute, skewness, kurtosis.

    - Skewness describes how much-skewed values are relative to a perfect bell curve observed in normal distribution
    and the direction of skew. If the majority of the values are at the left and the right tail is longer,
    we say that the distribution is skewed right or positively skewed; if the peak is toward the right and the left
    tail is longer, we say that the distribution is skewed left or negatively skewed. It leverage skewness
    functionality of Spark SQL. - (Excess) Kurtosis describes how tall and sharp the central peak is relative to a
    perfect bell curve observed in the normal distribution. The reference standard is a normal distribution,
    which has a kurtosis of 3. In token of this, often, the excess kurtosis is presented: excess kurtosis is simply
    kurtosis−3. Higher (positive) values indicate a higher, sharper peak; lower (negative) values indicate a less
    distinct peak. It leverages kurtosis functionality of Spark SQL.

    Parameters
    ----------
    idf :
        Input Dataframe
    list_of_cols :
        List of numerical columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all numerical columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)
    spark :


    Returns
    -------

    """

    num_cols = attributeType_segregation(idf)[0]
    if list_of_cols == "all":
        list_of_cols = num_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in num_cols for x in list_of_cols):
        raise TypeError("Invalid input for Column(s)")
    if len(list_of_cols) == 0:
        warnings.warn(
            "No Skewness/Kurtosis Computation - No numerical column(s) to analyze"
        )
        schema = T.StructType(
            [
                T.StructField("attribute", T.StringType(), True),
                T.StructField("skewness", T.StringType(), True),
                T.StructField("kurtosis", T.StringType(), True),
            ]
        )
        odf = spark.sparkContext.emptyRDD().toDF(schema)
        return odf

    shapes = []
    for i in list_of_cols:
        s, k = idf.select(F.skewness(i), F.kurtosis(i)).first()
        shapes.append([i, s, k])
    odf = (
        spark.createDataFrame(shapes, schema=("attribute", "skewness", "kurtosis"))
        .withColumn("skewness", F.round(F.col("skewness"), 4))
        .withColumn("kurtosis", F.round(F.col("kurtosis"), 4))
    )
    if print_impact:
        odf.show(len(list_of_cols))
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_analyzer.stats_generator.missingCount_computation"><code class="name flex">
<span>def <span class="ident">missingCount_computation</span></span>(<span>spark, idf, list_of_cols='all', drop_cols=[], print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><h2 id="parameters">Parameters</h2>
<p>spark :
Spark Session
idf :
Input Dataframe
list_of_cols :
List of columns to analyse e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
"all" can be passed to include all columns for analysis.
Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
drop_cols :
List of columns to be dropped e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
print_impact :
True, False (Default value = False)</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def missingCount_computation(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if list_of_cols == "all":
        num_cols, cat_cols, other_cols = attributeType_segregation(idf)
        list_of_cols = num_cols + cat_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in idf.columns for x in list_of_cols) | (len(list_of_cols) == 0):
        raise TypeError("Invalid input for Column(s)")

    idf_stats = idf.select(list_of_cols).summary("count")
    odf = (
        transpose_dataframe(idf_stats, "summary")
        .withColumn(
            "missing_count", F.lit(idf.count()) - F.col("count").cast(T.LongType())
        )
        .withColumn(
            "missing_pct", F.round(F.col("missing_count") / F.lit(idf.count()), 4)
        )
        .select(F.col("key").alias("attribute"), "missing_count", "missing_pct")
    )
    if print_impact:
        odf.show(len(list_of_cols))
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_analyzer.stats_generator.mode_computation"><code class="name flex">
<span>def <span class="ident">mode_computation</span></span>(<span>spark, idf, list_of_cols='all', drop_cols=[], print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><h2 id="parameters">Parameters</h2>
<p>spark :
Spark Session
idf :
Input Dataframe
list_of_cols :
List of Discrete (Categorical + Integer) columns to analyse e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
"all" can be passed to include all discrete columns for analysis.
Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
drop_cols :
List of columns to be dropped e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
print_impact :
True, False (Default value = False)</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def mode_computation(spark, idf, list_of_cols="all", drop_cols=[], print_impact=False):
    """

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of Discrete (Categorical + Integer) columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all discrete columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if list_of_cols == "all":
        num_cols, cat_cols, other_cols = attributeType_segregation(idf)
        list_of_cols = num_cols + cat_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))
    for i in idf.select(list_of_cols).dtypes:
        if i[1] not in ("string", "int", "bigint", "long"):
            list_of_cols.remove(i[0])

    if any(x not in idf.columns for x in list_of_cols):
        raise TypeError("Invalid input for Column(s)")

    if len(list_of_cols) == 0:
        warnings.warn("No Mode Computation - No discrete column(s) to analyze")
        schema = T.StructType(
            [
                T.StructField("attribute", T.StringType(), True),
                T.StructField("mode", T.StringType(), True),
                T.StructField("mode_rows", T.StringType(), True),
            ]
        )
        odf = spark.sparkContext.emptyRDD().toDF(schema)
        return odf

    mode = [
        list(
            idf.select(i)
            .dropna()
            .groupby(i)
            .count()
            .orderBy("count", ascending=False)
            .first()
            or [None, None]
        )
        for i in list_of_cols
    ]
    mode = [(str(i), str(j)) for i, j in mode]

    odf = spark.createDataFrame(
        zip(list_of_cols, mode), schema=("attribute", "metric")
    ).select(
        "attribute",
        (F.col("metric")["_1"]).alias("mode"),
        (F.col("metric")["_2"]).cast("long").alias("mode_rows"),
    )

    if print_impact:
        odf.show(len(list_of_cols))
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_analyzer.stats_generator.nonzeroCount_computation"><code class="name flex">
<span>def <span class="ident">nonzeroCount_computation</span></span>(<span>spark, idf, list_of_cols='all', drop_cols=[], print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><h2 id="parameters">Parameters</h2>
<p>spark :
Spark Session
idf :
Input Dataframe
list_of_cols :
List of numerical columns to analyse e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
"all" can be passed to include all numerical columns for analysis.
Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
drop_cols :
List of columns to be dropped e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
print_impact :
True, False (Default value = False)</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def nonzeroCount_computation(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of numerical columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all numerical columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    num_cols = attributeType_segregation(idf)[0]
    if list_of_cols == "all":
        list_of_cols = num_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in num_cols for x in list_of_cols):
        raise TypeError("Invalid input for Column(s)")

    if len(list_of_cols) == 0:
        warnings.warn(
            "No Non-Zero Count Computation - No numerical column(s) to analyze"
        )
        schema = T.StructType(
            [
                T.StructField("attribute", T.StringType(), True),
                T.StructField("nonzero_count", T.StringType(), True),
                T.StructField("nonzero_pct", T.StringType(), True),
            ]
        )
        odf = spark.sparkContext.emptyRDD().toDF(schema)
        return odf

    tmp = idf.select(list_of_cols).fillna(0).rdd.map(lambda row: Vectors.dense(row))
    nonzero_count = Statistics.colStats(tmp).numNonzeros()
    odf = spark.createDataFrame(
        zip(list_of_cols, [int(i) for i in nonzero_count]),
        schema=("attribute", "nonzero_count"),
    ).withColumn("nonzero_pct", F.round(F.col("nonzero_count") / F.lit(idf.count()), 4))
    if print_impact:
        odf.show(len(list_of_cols))
    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.data_analyzer.stats_generator.uniqueCount_computation"><code class="name flex">
<span>def <span class="ident">uniqueCount_computation</span></span>(<span>spark, idf, list_of_cols='all', drop_cols=[], print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><h2 id="parameters">Parameters</h2>
<p>spark :
Spark Session
idf :
Input Dataframe
list_of_cols :
List of Discrete (Categorical + Integer) columns to analyse e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
"all" can be passed to include all discrete columns for analysis.
Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
drop_cols :
List of columns to be dropped e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
print_impact :
True, False (Default value = False)</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def uniqueCount_computation(
    spark, idf, list_of_cols="all", drop_cols=[], print_impact=False
):
    """

    Parameters
    ----------
    spark :
        Spark Session
    idf :
        Input Dataframe
    list_of_cols :
        List of Discrete (Categorical + Integer) columns to analyse e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all discrete columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    print_impact :
        True, False (Default value = False)

    Returns
    -------

    """
    if list_of_cols == "all":
        list_of_cols = []
        for i in idf.dtypes:
            if i[1] in ("string", "int", "bigint", "long"):
                list_of_cols.append(i[0])
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in idf.columns for x in list_of_cols):
        raise TypeError("Invalid input for Column(s)")

    if len(list_of_cols) == 0:
        warnings.warn("No Unique Count Computation - No discrete column(s) to analyze")
        schema = T.StructType(
            [
                T.StructField("attribute", T.StringType(), True),
                T.StructField("unique_values", T.StringType(), True),
            ]
        )
        odf = spark.sparkContext.emptyRDD().toDF(schema)
        return odf

    uniquevalue_count = idf.agg(
        *(F.countDistinct(F.col(i)).alias(i) for i in list_of_cols)
    )
    odf = spark.createDataFrame(
        zip(list_of_cols, uniquevalue_count.rdd.map(list).collect()[0]),
        schema=("attribute", "unique_values"),
    )
    if print_impact:
        odf.show(len(list_of_cols))
    return odf
```
</pre>
</details>
</dd>
</dl>