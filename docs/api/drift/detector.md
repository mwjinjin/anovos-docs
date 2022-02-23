# <code>detector</code>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
# coding=utf-8

from __future__ import division, print_function

import numpy as np
import pandas as pd
import pyspark
from loguru import logger
from pyspark.sql import SparkSession, DataFrame
from pyspark.sql import functions as F
from pyspark.sql import types as T
from scipy.stats import variation
import sympy as sp

from anovos.data_ingest.data_ingest import concatenate_dataset
from anovos.data_transformer.transformers import attribute_binning
from anovos.shared.utils import attributeType_segregation
from .distances import hellinger, psi, js_divergence, ks
from .validations import check_distance_method, check_list_of_columns


@check_distance_method
@check_list_of_columns
def statistics(
    spark: SparkSession,
    idf_target: DataFrame,
    idf_source: DataFrame,
    *,
    list_of_cols: list = "all",
    drop_cols: list = None,
    method_type: str = "PSI",
    bin_method: str = "equal_range",
    bin_size: int = 10,
    threshold: float = 0.1,
    pre_existing_source: bool = False,
    source_path: str = "NA",
    model_directory: str = "drift_statistics",
    print_impact: bool = False,
):
    """When the performance of a deployed machine learning model degrades in production, one potential reason is that
    the data used in training and prediction are not following the same distribution.

    Data drift mainly includes the following manifestations:

    - Covariate shift: training and test data follow different distributions. For example, An algorithm predicting
    income that is trained on younger population but tested on older population. - Prior probability shift: change of
    prior probability. For example in a spam classification problem, the proportion of spam emails changes from 0.2
    in training data to 0.6 in testing data. - Concept shift: the distribution of the target variable changes given
    fixed input values. For example in the same spam classification problem, emails tagged as spam in training data
    are more likely to be tagged as non-spam in testing data.

    In our module, we mainly focus on covariate shift detection.

    In summary, given 2 datasets, source and target datasets, we would like to quantify the drift of some numerical
    attributes from source to target datasets. The whole process can be broken down into 2 steps: (1) convert each
    attribute of interest in source and target datasets into source and target probability distributions. (2)
    calculate the statistical distance between source and target distributions for each attribute.

    In the first step, attribute_binning is firstly performed to bin the numerical attributes of the source dataset,
    which requires two input variables: bin_method and bin_size. The same binning method is applied on the target
    dataset to align two results. The probability distributions are computed by dividing the frequency of each bin by
    the total frequency.

    In the second step, 4 choices of statistical metrics are provided to measure the data drift of an attribute from
    source to target distribution: Population Stability Index (PSI), Jensen-Shannon Divergence (JSD),
    Hellinger Distance (HD) and Kolmogorov-Smirnov Distance (KS).

    They are calculated as below:
    For two discrete probability distributions *P=(p_1,…,p_k)* and *Q=(q_1,…,q_k),*

    ![https://raw.githubusercontent.com/anovos/anovos-docs/main/docs/assets/drift_stats_formulae.png](https://raw.githubusercontent.com/anovos/anovos-docs/main/docs/assets/drift_stats_formulae.png)

    A threshold can be set to flag out drifted attributes. If multiple statistical metrics have been calculated,
    an attribute will be marked as drifted if any of its statistical metric is larger than the threshold.

    This function can be used in many scenarios. For example:

    1. Attribute level data drift can be analysed together with the attribute importance of a machine learning model.
    The more important an attribute is, the more attention it needs to be given if drift presents. 2. To analyse data
    drift over time, one can treat one dataset as the source / baseline dataset and multiple datasets as the target
    datasets. Drift analysis can be performed between the source dataset and each of the target dataset to quantify
    the drift over time.

    ---------

    - *idf_target*: Input target Dataframe - *idf_source*: Input source Dataframe - *list_of_cols*: List of columns
    to check drift (list or string of col names separated by |). Use ‘all’ - to include all non-array columns (
    excluding drop_cols). - *drop_cols*: List of columns to be dropped (list or string of col names separated by |) -
    method: PSI, JSD, HD, KS (list or string of methods separated by |). Use ‘all’ - to calculate all metrics. -
    *bin_method*: equal_frequency or equal_range - *bin_size*: 10 - 20 (recommended for PSI), >100 (other method
    types) - *threshold*: To flag attributes meeting drift threshold - *pre_existing_source*: True if binning model &
    frequency counts/attribute exists already, False Otherwise. - *source_path*: If pre_existing_source is True,
    this argument is path for the source dataset details - drift_statistics folder. drift_statistics folder must
    contain attribute_binning & frequency_counts folders. If pre_existing_source is False, this argument can be used
    for saving the details. Default "NA" for temporarily saving source dataset attribute_binning folder

    Parameters
    ----------
    spark :
        Spark Session
    idf_target :
        Input Dataframe
    idf_source :
        Baseline/Source Dataframe. This argument is ignored if pre_existing_source is True.
    list_of_cols :
        List of columns to check drift e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all (non-array) columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols.
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
    method_type :
        PSI", "JSD", "HD", "KS","all".
        "all" can be passed to calculate all drift metrics.
        One or more methods can be passed in a form of list or string where different metrics are separated
        by pipe delimiter “|” e.g. ["PSI", "JSD"] or "PSI|JSD"
    bin_method :
        equal_frequency", "equal_range".
        In "equal_range" method, each bin is of equal size/width and in "equal_frequency", each bin
        has equal no. of rows, though the width of bins may vary.
    bin_size :
        Number of bins for creating histogram
    threshold :
        A column is flagged if any drift metric is above the threshold.
    pre_existing_source :
        Boolean argument – True or False. True if the drift_statistics folder (binning model &
        frequency counts for each attribute) exists already, False Otherwise.
    source_path :
        If pre_existing_source is False, this argument can be used for saving the drift_statistics folder.
        The drift_statistics folder will have attribute_binning (binning model) & frequency_counts sub-folders.
        If pre_existing_source is True, this argument is path for referring the drift_statistics folder.
        Default "NA" for temporarily saving source dataset attribute_binning folder.
    model_directory :
        If pre_existing_source is False, this argument can be used for saving the drift stats to folder.
        The default drift statics directory is drift_statistics folder will have attribute_binning
        If pre_existing_source is True, this argument is model_directory for referring the drift statistics dir.
        Default "drift_statistics" for temporarily saving source dataset attribute_binning folder.
    print_impact :
        True, False
    spark :
        type spark: SparkSession :
    idf_target :
        type idf_target: DataFrame :
    idf_source :
        type idf_source: DataFrame :
    spark: SparkSession :

    idf_target: DataFrame :

    idf_source: DataFrame :

    * :

    list_of_cols: list :
         (Default value = "all")
    drop_cols: list :
         (Default value = None)
    method_type: str :
         (Default value = "PSI")
    bin_method: str :
         (Default value = "equal_range")
    bin_size: int :
         (Default value = 10)
    threshold: float :
         (Default value = 0.1)
    pre_existing_source: bool :
         (Default value = False)
    source_path: str :
         (Default value = "NA")
    model_directory: str :
         (Default value = "drift_statistics")
    print_impact: bool :
         (Default value = False)

    Returns
    -------

    """
    drop_cols = drop_cols or []
    num_cols = attributeType_segregation(idf_target.select(list_of_cols))[0]

    if not pre_existing_source:
        source_bin = attribute_binning(
            spark,
            idf_source,
            list_of_cols=num_cols,
            method_type=bin_method,
            bin_size=bin_size,
            pre_existing_model=False,
            model_path=source_path + "/" + model_directory,
        )
        source_bin.persist(pyspark.StorageLevel.MEMORY_AND_DISK).count()

    target_bin = attribute_binning(
        spark,
        idf_target,
        list_of_cols=num_cols,
        method_type=bin_method,
        bin_size=bin_size,
        pre_existing_model=True,
        model_path=source_path + "/" + model_directory,
    )

    target_bin.persist(pyspark.StorageLevel.MEMORY_AND_DISK).count()
    result = {"attribute": [], "flagged": []}

    for method in method_type:
        result[method] = []

    for i in list_of_cols:
        if pre_existing_source:
            x = spark.read.csv(
                source_path + "/" + model_directory + "/frequency_counts/" + i,
                header=True,
                inferSchema=True,
            )
        else:
            x = (
                source_bin.groupBy(i)
                .agg((F.count(i) / idf_source.count()).alias("p"))
                .fillna(-1)
            )
            x.coalesce(1).write.csv(
                source_path + "/" + model_directory + "/frequency_counts/" + i,
                header=True,
                mode="overwrite",
            )

        y = (
            target_bin.groupBy(i)
            .agg((F.count(i) / idf_target.count()).alias("q"))
            .fillna(-1)
        )

        xy = (
            x.join(y, i, "full_outer")
            .fillna(0.0001, subset=["p", "q"])
            .replace(0, 0.0001)
            .orderBy(i)
        )
        p = np.array(xy.select("p").rdd.flatMap(lambda x: x).collect())
        q = np.array(xy.select("q").rdd.flatMap(lambda x: x).collect())

        result["attribute"].append(i)
        counter = 0

        for idx, method in enumerate(method_type):
            drift_function = {
                "PSI": psi,
                "JSD": js_divergence,
                "HD": hellinger,
                "KS": ks,
            }
            metric = float(round(drift_function[method](p, q), 4))
            result[method].append(metric)
            if counter == 0:
                if metric > threshold:
                    result["flagged"].append(1)
                    counter = 1
            if (idx == (len(method_type) - 1)) & (counter == 0):
                result["flagged"].append(0)

    odf = (
        spark.createDataFrame(
            pd.DataFrame.from_dict(result, orient="index").transpose()
        )
        .select(["attribute"] + method_type + ["flagged"])
        .orderBy(F.desc("flagged"))
    )

    if print_impact:
        logger.info("All Attributes:")
        odf.show(len(list_of_cols))
        logger.info("Attributes meeting Data Drift threshold:")
        drift = odf.where(F.col("flagged") == 1)
        drift.show(drift.count())

    return odf


def stability_index_computation(
    spark,
    *idfs,
    list_of_cols="all",
    drop_cols=[],
    metric_weightages={"mean": 0.5, "stddev": 0.3, "kurtosis": 0.2},
    existing_metric_path="",
    appended_metric_path="",
    threshold=1,
    print_impact=False,
):
    """The data stability is represented by a single metric to summarise the stability of an attribute over multiple
    time periods. For example, given 6 datasets collected in 6 consecutive time periods (D1, D2, …, D6),
    data stability index of an attribute measures how stable the attribute is from D1 to D6.

    The major difference between data drift and data stability is that data drift analysis is only based on 2
    datasets: source and target. However data stability analysis can be performed on multiple datasets. In addition,
    the source dataset is not required indicating that the stability index can be directly computed among multiple
    target datasets by comparing the statistical properties among them.

    In summary, given N datasets representing different time periods, we would like to measure the stability of some
    numerical attributes from the first to the N-th dataset.

    The whole process can be broken down into 2 steps: (1) Choose a few statistical metrics to describe the
    distribution of each attribute at each time period. (2) Compute attribute level stability by combining the
    stability of each statistical metric over time periods.

    In the first step, we choose mean, standard deviation and kurtosis as the statistical metrics in our
    implementation. Intuitively, they represent different aspects of a distribution: mean measures central tendency,
    standard deviation measures dispersion and kurtosis measures shape of a distribution. Reasons of selecting those
    3 metrics will be explained in a later section. With mean, standard deviation and kurtosis computed for each
    attribute at each time interval, we can form 3 arrays of size N for each attribute.

    In the second step, Coefficient of Variation (CV) is used to measure the stability of each metric. CV represents
    the ratio of the standard deviation to the mean, which is a unitless statistic to compare the relative variation
    from one array to another. Considering the wide potential range of CV, the absolute value of CV is then mapped to
    an integer between 0 and 4 according to the table below, where 0 indicates highly unstable and 4 indicates highly
    stable. We call this integer a metric stability index.

    | abs(CV) Interval | Metric Stability Index |
    |------------------|------------------------|
    | [0, 0.03)        | 4                      |
    | [0.03, 0.1)      | 3                      |
    | [0.1, 0.2)       | 2                      |
    | [0.2, 0.5)       | 1                      |
    | [0.5, +inf)      | 0                      |


    Finally, the attribute stability index (SI) is a weighted sum of 3 metric stability indexes, where we assign 50%
    for mean, 30% for standard deviation and 20% for kurtosis. The final output is a float between 0 and 4 and an
    attribute can be classified as one of the following categories: very unstable (0≤SI<1), unstable (1≤SI<2),
    marginally stable (2≤SI<3), stable (3≤SI<3.5) and very stable (3.5≤SI≤4).

    For example, there are 6 samples of attribute X from T1 to T6. For each sample, we have computed the statistical
    metrics of X from T1 to T6:

    | idx | Mean | Standard deviation | Kurtosis |
    |-----|------|--------------------|----------|
    | 1   | 11   | 2                  | 3.9      |
    | 2   | 12   | 1                  | 4.2      |
    | 3   | 15   | 3                  | 4.0      |
    | 4   | 10   | 2                  | 4.1      |
    | 5   | 11   | 1                  | 4.2      |
    | 6   | 13   | 0.5                | 4.0      |

    Then we calculate the Coefficient of Variation for each array:

    - CV of mean = CV([11, 12, 15, 10, 11, 13]) = 0.136
    - CV of standard deviation = CV([2, 1, 3, 2, 1, 0.5]) = 0.529
    - CV of kurtosis = CV([3.9, 4.2, 4.0, 4.1, 4.2, 4.0]) = 0.027

    Metric stability indexes are then computed by mapping each CV value to an integer accordingly. As a result,
    metric stability index is 2 for mean, 0 for standard deviation and 4 for kurtosis.

    Why mean is chosen over median?

    - Dummy variables which take only the value 0 or 1 are frequently seen in machine learning features. Mean of a
    dummy variable represents the proportion of value 1 and median of a dummy variable is either 0 or 1 whichever is
    more frequent. However, CV may not work well when 0 appears in the array or the array contains both positive and
    negative values. For example, intuitively [0,0,0,0,0,1,0,0,0] is a stable array but its CV is 2.83 which is
    extremely high, but cv of [0.45,0.44,0.48,0.49,0.42,0.52,0.49,0.47,0.48] is 0.06 which is much more reasonable.
    Thus we decided to use mean instead of median. Although median is considered as a more robust choice,
    outlier treatment can be applied prior to data stability analysis to handle this issue.

    Why kurtosis is chosen over skewness?

    - Kurtosis is a positive value (note that we use kurtosis instead of excess kurtosis which) but skewness can
    range from –inf to +inf. Usually, if skewness is between -0.5 and 0.5, the distribution is approximately
    symmetric. Thus, if the skewness fluctuates around 0, the CV is highly likely to be high or invalid because the
    mean will be close to 0.

    Stability index is preferred in the following scenario:

    - Pairwise drift analysis can be performed between the source dataset and each of the target dataset to quantify
    the drift over time. However this can be time-consuming especially when the number of target dataset is large. In
    this case, measuring data stability instead of data drift would be a much faster alternative and the
    source/baseline dataset is not required as well

    Troubleshooting

    - If the attribute stability index appears to be nan, it may due to one of the following reasons: - One metric (
    likely to be kurtosis) is nan. For example, the kurtosis of a sample is nan If its standard deviation is 0. - The
    mean of a metric from the first to the N-th dataset is zero, causing the denominator of CV to be 0. For example,
    when mean of attribute X is always zero for all datasets, its stability index would be nan.

    Limitation

    - Limitation of CV: CV may not work well when 0 appears in the array or the array contains both positive and
    negative values.

    ------

    - *idfs*: Input Dataframes (flexible) - *list_of_cols*: Numerical columns (in list format or string separated by
    |). Use ‘all’ - to include all numerical columns (excluding drop_cols). - *drop_cols*: List of columns to be
    dropped (list or string of col names separated by |) - *metric_weightages*: A dictionary with key being the
    metric name (mean,stdev,kurtosis) and value being the weightage of the metric (between 0 and 1). Sum of all
    weightages must be 1. - *existing_metric_path*: this argument is path for pre-existing metrics of historical
    datasets  <idx,attribute,mean,stdev,kurtosis>. idx is index number of historical datasets assigned in
    chronological order - *appended_metric_path*: this argument is path for saving input dataframes metrics after
    appending to the historical datasets' metrics. - *threshold*: To flag unstable attributes meeting the threshold


    Parameters
    ----------
    spark :
        Spark Session
    idfs :
        Variable number of input dataframes
    list_of_cols :
        List of numerical columns to check stability e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all numerical columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    metric_weightages :
        Takes input in dictionary format with keys being the metric name - "mean","stdev","kurtosis"
        and value being the weightage of the metric (between 0 and 1). Sum of all weightages must be 1.
         (Default value = {"mean": 0.5)
    existing_metric_path :
        This argument is path for referring pre-existing metrics of historical datasets and is
        of schema [idx, attribute, mean, stdev, kurtosis].
        idx is index number of historical datasets assigned in chronological order. (Default value = "")
    appended_metric_path :
        This argument is path for saving input dataframes metrics after appending to the
        historical datasets' metrics. (Default value = "")
    threshold :
        A column is flagged if the stability index is below the threshold, which varies between 0 to 4.
        The following criteria can be used to classifiy stability_index (SI): very unstable: 0≤SI<1,
        unstable: 1≤SI<2, marginally stable: 2≤SI<3, stable: 3≤SI<3.5 and very stable: 3.5≤SI≤4. (Default value = 1)
    print_impact :
        True, False (Default value = False)
    *idfs :

    "stddev": 0.3 :

    "kurtosis": 0.2} :


    Returns
    -------

    """

    num_cols = attributeType_segregation(idfs[0])[0]
    if list_of_cols == "all":
        list_of_cols = num_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in num_cols for x in list_of_cols) | (len(list_of_cols) == 0):
        raise TypeError("Invalid input for Column(s)")

    if (
        round(
            metric_weightages.get("mean", 0)
            + metric_weightages.get("stddev", 0)
            + metric_weightages.get("kurtosis", 0),
            3,
        )
        != 1
    ):
        raise ValueError(
            "Invalid input for metric weightages. Either metric name is incorrect or sum of metric weightages is not 1.0"
        )

    if existing_metric_path:
        existing_metric_df = spark.read.csv(
            existing_metric_path, header=True, inferSchema=True
        )
        dfs_count = existing_metric_df.select(F.max(F.col("idx"))).first()[0]
    else:
        schema = T.StructType(
            [
                T.StructField("idx", T.IntegerType(), True),
                T.StructField("attribute", T.StringType(), True),
                T.StructField("mean", T.DoubleType(), True),
                T.StructField("stddev", T.DoubleType(), True),
                T.StructField("kurtosis", T.DoubleType(), True),
            ]
        )
        existing_metric_df = spark.sparkContext.emptyRDD().toDF(schema)
        dfs_count = 0

    metric_ls = []
    for idf in idfs:
        for i in list_of_cols:
            mean, stddev, kurtosis = idf.select(
                F.mean(i), F.stddev(i), F.kurtosis(i)
            ).first()
            metric_ls.append(
                [dfs_count + 1, i, mean, stddev, kurtosis + 3.0 if kurtosis else None]
            )
        dfs_count += 1

    new_metric_df = spark.createDataFrame(
        metric_ls, schema=("idx", "attribute", "mean", "stddev", "kurtosis")
    )
    appended_metric_df = concatenate_dataset(existing_metric_df, new_metric_df)

    if appended_metric_path:
        appended_metric_df.coalesce(1).write.csv(
            appended_metric_path, header=True, mode="overwrite"
        )

    result = []
    for i in list_of_cols:
        i_output = [i]
        for metric in ["mean", "stddev", "kurtosis"]:
            metric_stats = (
                appended_metric_df.where(F.col("attribute") == i)
                .orderBy("idx")
                .select(metric)
                .fillna(np.nan)
                .rdd.flatMap(list)
                .collect()
            )
            metric_cv = round(float(variation([a for a in metric_stats])), 4) or None
            i_output.append(metric_cv)
        result.append(i_output)

    schema = T.StructType(
        [
            T.StructField("attribute", T.StringType(), True),
            T.StructField("mean_cv", T.FloatType(), True),
            T.StructField("stddev_cv", T.FloatType(), True),
            T.StructField("kurtosis_cv", T.FloatType(), True),
        ]
    )

    odf = spark.createDataFrame(result, schema=schema)

    def score_cv(cv, thresholds=[0.03, 0.1, 0.2, 0.5]):
        """

        Parameters
        ----------
        cv :
            param thresholds: (Default value = [0.03)
        0 :
            1:
        0 :
            2:
        0 :
            5]:
        thresholds :
             (Default value = [0.03)
        0.1 :

        0.2 :

        0.5] :


        Returns
        -------

        """
        if cv is None:
            return None
        else:
            cv = abs(cv)
            stability_index = [4, 3, 2, 1, 0]
            for i, thresh in enumerate(thresholds):
                if cv < thresh:
                    return stability_index[i]
            return stability_index[-1]

    f_score_cv = F.udf(score_cv, T.IntegerType())

    odf = (
        odf.replace(np.nan, None)
        .withColumn("mean_si", f_score_cv(F.col("mean_cv")))
        .withColumn("stddev_si", f_score_cv(F.col("stddev_cv")))
        .withColumn("kurtosis_si", f_score_cv(F.col("kurtosis_cv")))
        .withColumn(
            "stability_index",
            F.round(
                (
                    F.col("mean_si") * metric_weightages.get("mean", 0)
                    + F.col("stddev_si") * metric_weightages.get("stddev", 0)
                    + F.col("kurtosis_si") * metric_weightages.get("kurtosis", 0)
                ),
                4,
            ),
        )
        .withColumn(
            "flagged",
            F.when(
                (F.col("stability_index") < threshold)
                | (F.col("stability_index").isNull()),
                1,
            ).otherwise(0),
        )
    )

    if print_impact:
        logger.info("All Attributes:")
        odf.show(len(list_of_cols))
        logger.info("Potential Unstable Attributes:")
        unstable = odf.where(F.col("flagged") == 1)
        unstable.show(unstable.count())

    return odf


def feature_stability_estimation(
    spark,
    attribute_stats,
    attribute_transformation,
    metric_weightages={"mean": 0.5, "stddev": 0.3, "kurtosis": 0.2},
    threshold=1,
    print_impact=False,
):
    """

    Parameters
    ----------
    spark :
        Spark Session
    attribute_stats :
        Spark dataframe. The intermediate dataframe saved by running function
        stabilityIndex_computation with schema [idx, attribute, mean, stddev, kurtosis].
        It should contain all the attributes used in argument attribute_transformation.
    attribute_transformation :
        Takes input in dictionary format: each key-value combination represents one
        new feature. Each key is a string containing all the attributes involved in
        the new feature seperated by '|'. Each value is the transformation of the
        attributes in string. For example, {'X|Y|Z': 'X**2+Y/Z', 'A': 'log(A)'}
    metric_weightages :
        Takes input in dictionary format with keys being the metric name - "mean","stdev","kurtosis"
        and value being the weightage of the metric (between 0 and 1). Sum of all weightages must be 1. (Default value = {"mean": 0.5)
    threshold :
        A column is flagged if the stability index is below the threshold, which varies between 0 to 4.
        The following criteria can be used to classifiy stability_index (SI): very unstable: 0≤SI<1,
        unstable: 1≤SI<2, marginally stable: 2≤SI<3, stable: 3≤SI<3.5 and very stable: 3.5≤SI≤4. (Default value = 1)
    print_impact :
        True, False (Default value = False)
    "stddev": 0.3 :

    "kurtosis": 0.2} :


    Returns
    -------

    """

    def stats_estimation(attributes, transformation, mean, stddev):
        """

        Parameters
        ----------
        attributes :
            param transformation:
        mean :
            param stddev:
        transformation :

        stddev :


        Returns
        -------

        """
        attribute_means = list(zip(attributes, mean))
        first_dev = []
        second_dev = []
        est_mean = 0
        est_var = 0
        for attr, s in zip(attributes, stddev):
            first_dev = sp.diff(transformation, attr)
            second_dev = sp.diff(transformation, attr, 2)

            est_mean += s**2 * second_dev.subs(attribute_means) / 2
            est_var += s**2 * (first_dev.subs(attribute_means)) ** 2

        transformation = sp.parse_expr(transformation)
        est_mean += transformation.subs(attribute_means)

        return [float(est_mean), float(est_var)]

    f_stats_estimation = F.udf(stats_estimation, T.ArrayType(T.FloatType()))

    index = (
        attribute_stats.select("idx")
        .distinct()
        .orderBy("idx")
        .rdd.flatMap(list)
        .collect()
    )
    attribute_names = list(attribute_transformation.keys())
    transformations = list(attribute_transformation.values())

    feature_metric = []
    for attributes, transformation in zip(attribute_names, transformations):
        attributes = [x.strip() for x in attributes.split("|")]
        for idx in index:
            attr_mean_list, attr_stddev_list = [], []
            for attr in attributes:
                df_temp = attribute_stats.where(
                    (F.col("idx") == idx) & (F.col("attribute") == attr)
                )
                if df_temp.count() == 0:
                    raise TypeError(
                        "Invalid input for attribute_stats: all involved attributes must have available statistics across all time periods (idx)"
                    )
                attr_mean_list.append(
                    df_temp.select("mean").rdd.flatMap(lambda x: x).collect()[0]
                )
                attr_stddev_list.append(
                    df_temp.select("stddev").rdd.flatMap(lambda x: x).collect()[0]
                )
            feature_metric.append(
                [idx, transformation, attributes, attr_mean_list, attr_stddev_list]
            )

    schema = T.StructType(
        [
            T.StructField("idx", T.IntegerType(), True),
            T.StructField("transformation", T.StringType(), True),
            T.StructField("attributes", T.ArrayType(T.StringType()), True),
            T.StructField("attr_mean_list", T.ArrayType(T.FloatType()), True),
            T.StructField("attr_stddev_list", T.ArrayType(T.FloatType()), True),
        ]
    )

    df_feature_metric = (
        spark.createDataFrame(feature_metric, schema=schema)
        .withColumn(
            "est_feature_stats",
            f_stats_estimation(
                "attributes", "transformation", "attr_mean_list", "attr_stddev_list"
            ),
        )
        .withColumn("est_feature_mean", F.col("est_feature_stats")[0])
        .withColumn("est_feature_stddev", F.sqrt(F.col("est_feature_stats")[1]))
        .select(
            "idx",
            "attributes",
            "transformation",
            "est_feature_mean",
            "est_feature_stddev",
        )
    )

    output = []
    for idx, i in enumerate(transformations):
        i_output = [i]
        for metric in ["est_feature_mean", "est_feature_stddev"]:
            metric_stats = (
                df_feature_metric.where(F.col("transformation") == i)
                .orderBy("idx")
                .select(metric)
                .fillna(np.nan)
                .rdd.flatMap(list)
                .collect()
            )
            metric_cv = round(float(variation([a for a in metric_stats])), 4) or None
            i_output.append(metric_cv)
        output.append(i_output)

    schema = T.StructType(
        [
            T.StructField("feature_formula", T.StringType(), True),
            T.StructField("mean_cv", T.FloatType(), True),
            T.StructField("stddev_cv", T.FloatType(), True),
        ]
    )

    odf = spark.createDataFrame(output, schema=schema)

    def score_cv(cv, thresholds=[0.03, 0.1, 0.2, 0.5]):
        """

        Parameters
        ----------
        cv :
            param thresholds: (Default value = [0.03)
        0 :
            1:
        0 :
            2:
        0 :
            5]:
        thresholds :
             (Default value = [0.03)
        0.1 :

        0.2 :

        0.5] :


        Returns
        -------

        """
        if cv is None:
            return None
        else:
            cv = abs(cv)
            stability_index = [4, 3, 2, 1, 0]
            for i, thresh in enumerate(thresholds):
                if cv < thresh:
                    return stability_index[i]
            return stability_index[-1]

    f_score_cv = F.udf(score_cv, T.IntegerType())

    odf = (
        odf.replace(np.nan, None)
        .withColumn("mean_si", f_score_cv(F.col("mean_cv")))
        .withColumn("stddev_si", f_score_cv(F.col("stddev_cv")))
        .withColumn(
            "stability_index_lower_bound",
            F.round(
                F.col("mean_si") * metric_weightages.get("mean", 0)
                + F.col("stddev_si") * metric_weightages.get("stddev", 0),
                4,
            ),
        )
        .withColumn(
            "stability_index_upper_bound",
            F.round(
                F.col("stability_index_lower_bound")
                + 4 * metric_weightages.get("kurtosis", 0),
                4,
            ),
        )
        .withColumn(
            "flagged_lower",
            F.when(
                (F.col("stability_index_lower_bound") < threshold)
                | (F.col("stability_index_lower_bound").isNull()),
                1,
            ).otherwise(0),
        )
        .withColumn(
            "flagged_upper",
            F.when(
                (F.col("stability_index_upper_bound") < threshold)
                | (F.col("stability_index_upper_bound").isNull()),
                1,
            ).otherwise(0),
        )
    )

    if print_impact:
        logger.info("All Features:")
        odf.show(len(attribute_names), False)
        logger.info(
            "Potential Unstable Features Identified by Both Lower and Upper Bounds:"
        )
        unstable = odf.where(F.col("flagged_upper") == 1)
        unstable.show(unstable.count())

    return odf
```
</pre>
</details>
## Functions
<dl>
<dt id="anovos.drift.detector.feature_stability_estimation"><code class="name flex">
<span>def <span class="ident">feature_stability_estimation</span></span>(<span>spark, attribute_stats, attribute_transformation, metric_weightages={'mean': 0.5, 'stddev': 0.3, 'kurtosis': 0.2}, threshold=1, print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><h2 id="parameters">Parameters</h2>
<p>spark :
Spark Session
attribute_stats :
Spark dataframe. The intermediate dataframe saved by running function
stabilityIndex_computation with schema [idx, attribute, mean, stddev, kurtosis].
It should contain all the attributes used in argument attribute_transformation.
attribute_transformation :
Takes input in dictionary format: each key-value combination represents one
new feature. Each key is a string containing all the attributes involved in
the new feature seperated by '|'. Each value is the transformation of the
attributes in string. For example, {'X|Y|Z': 'X**2+Y/Z', 'A': 'log(A)'}
metric_weightages :
Takes input in dictionary format with keys being the metric name - "mean","stdev","kurtosis"
and value being the weightage of the metric (between 0 and 1). Sum of all weightages must be 1. (Default value = {"mean": 0.5)
threshold :
A column is flagged if the stability index is below the threshold, which varies between 0 to 4.
The following criteria can be used to classifiy stability_index (SI): very unstable: 0≤SI&lt;1,
unstable: 1≤SI&lt;2, marginally stable: 2≤SI&lt;3, stable: 3≤SI&lt;3.5 and very stable: 3.5≤SI≤4. (Default value = 1)
print_impact :
True, False (Default value = False)
"stddev": 0.3 :</p>
<p>"kurtosis": 0.2} :</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def feature_stability_estimation(
    spark,
    attribute_stats,
    attribute_transformation,
    metric_weightages={"mean": 0.5, "stddev": 0.3, "kurtosis": 0.2},
    threshold=1,
    print_impact=False,
):
    """

    Parameters
    ----------
    spark :
        Spark Session
    attribute_stats :
        Spark dataframe. The intermediate dataframe saved by running function
        stabilityIndex_computation with schema [idx, attribute, mean, stddev, kurtosis].
        It should contain all the attributes used in argument attribute_transformation.
    attribute_transformation :
        Takes input in dictionary format: each key-value combination represents one
        new feature. Each key is a string containing all the attributes involved in
        the new feature seperated by '|'. Each value is the transformation of the
        attributes in string. For example, {'X|Y|Z': 'X**2+Y/Z', 'A': 'log(A)'}
    metric_weightages :
        Takes input in dictionary format with keys being the metric name - "mean","stdev","kurtosis"
        and value being the weightage of the metric (between 0 and 1). Sum of all weightages must be 1. (Default value = {"mean": 0.5)
    threshold :
        A column is flagged if the stability index is below the threshold, which varies between 0 to 4.
        The following criteria can be used to classifiy stability_index (SI): very unstable: 0≤SI<1,
        unstable: 1≤SI<2, marginally stable: 2≤SI<3, stable: 3≤SI<3.5 and very stable: 3.5≤SI≤4. (Default value = 1)
    print_impact :
        True, False (Default value = False)
    "stddev": 0.3 :

    "kurtosis": 0.2} :


    Returns
    -------

    """

    def stats_estimation(attributes, transformation, mean, stddev):
        """

        Parameters
        ----------
        attributes :
            param transformation:
        mean :
            param stddev:
        transformation :

        stddev :


        Returns
        -------

        """
        attribute_means = list(zip(attributes, mean))
        first_dev = []
        second_dev = []
        est_mean = 0
        est_var = 0
        for attr, s in zip(attributes, stddev):
            first_dev = sp.diff(transformation, attr)
            second_dev = sp.diff(transformation, attr, 2)

            est_mean += s**2 * second_dev.subs(attribute_means) / 2
            est_var += s**2 * (first_dev.subs(attribute_means)) ** 2

        transformation = sp.parse_expr(transformation)
        est_mean += transformation.subs(attribute_means)

        return [float(est_mean), float(est_var)]

    f_stats_estimation = F.udf(stats_estimation, T.ArrayType(T.FloatType()))

    index = (
        attribute_stats.select("idx")
        .distinct()
        .orderBy("idx")
        .rdd.flatMap(list)
        .collect()
    )
    attribute_names = list(attribute_transformation.keys())
    transformations = list(attribute_transformation.values())

    feature_metric = []
    for attributes, transformation in zip(attribute_names, transformations):
        attributes = [x.strip() for x in attributes.split("|")]
        for idx in index:
            attr_mean_list, attr_stddev_list = [], []
            for attr in attributes:
                df_temp = attribute_stats.where(
                    (F.col("idx") == idx) & (F.col("attribute") == attr)
                )
                if df_temp.count() == 0:
                    raise TypeError(
                        "Invalid input for attribute_stats: all involved attributes must have available statistics across all time periods (idx)"
                    )
                attr_mean_list.append(
                    df_temp.select("mean").rdd.flatMap(lambda x: x).collect()[0]
                )
                attr_stddev_list.append(
                    df_temp.select("stddev").rdd.flatMap(lambda x: x).collect()[0]
                )
            feature_metric.append(
                [idx, transformation, attributes, attr_mean_list, attr_stddev_list]
            )

    schema = T.StructType(
        [
            T.StructField("idx", T.IntegerType(), True),
            T.StructField("transformation", T.StringType(), True),
            T.StructField("attributes", T.ArrayType(T.StringType()), True),
            T.StructField("attr_mean_list", T.ArrayType(T.FloatType()), True),
            T.StructField("attr_stddev_list", T.ArrayType(T.FloatType()), True),
        ]
    )

    df_feature_metric = (
        spark.createDataFrame(feature_metric, schema=schema)
        .withColumn(
            "est_feature_stats",
            f_stats_estimation(
                "attributes", "transformation", "attr_mean_list", "attr_stddev_list"
            ),
        )
        .withColumn("est_feature_mean", F.col("est_feature_stats")[0])
        .withColumn("est_feature_stddev", F.sqrt(F.col("est_feature_stats")[1]))
        .select(
            "idx",
            "attributes",
            "transformation",
            "est_feature_mean",
            "est_feature_stddev",
        )
    )

    output = []
    for idx, i in enumerate(transformations):
        i_output = [i]
        for metric in ["est_feature_mean", "est_feature_stddev"]:
            metric_stats = (
                df_feature_metric.where(F.col("transformation") == i)
                .orderBy("idx")
                .select(metric)
                .fillna(np.nan)
                .rdd.flatMap(list)
                .collect()
            )
            metric_cv = round(float(variation([a for a in metric_stats])), 4) or None
            i_output.append(metric_cv)
        output.append(i_output)

    schema = T.StructType(
        [
            T.StructField("feature_formula", T.StringType(), True),
            T.StructField("mean_cv", T.FloatType(), True),
            T.StructField("stddev_cv", T.FloatType(), True),
        ]
    )

    odf = spark.createDataFrame(output, schema=schema)

    def score_cv(cv, thresholds=[0.03, 0.1, 0.2, 0.5]):
        """

        Parameters
        ----------
        cv :
            param thresholds: (Default value = [0.03)
        0 :
            1:
        0 :
            2:
        0 :
            5]:
        thresholds :
             (Default value = [0.03)
        0.1 :

        0.2 :

        0.5] :


        Returns
        -------

        """
        if cv is None:
            return None
        else:
            cv = abs(cv)
            stability_index = [4, 3, 2, 1, 0]
            for i, thresh in enumerate(thresholds):
                if cv < thresh:
                    return stability_index[i]
            return stability_index[-1]

    f_score_cv = F.udf(score_cv, T.IntegerType())

    odf = (
        odf.replace(np.nan, None)
        .withColumn("mean_si", f_score_cv(F.col("mean_cv")))
        .withColumn("stddev_si", f_score_cv(F.col("stddev_cv")))
        .withColumn(
            "stability_index_lower_bound",
            F.round(
                F.col("mean_si") * metric_weightages.get("mean", 0)
                + F.col("stddev_si") * metric_weightages.get("stddev", 0),
                4,
            ),
        )
        .withColumn(
            "stability_index_upper_bound",
            F.round(
                F.col("stability_index_lower_bound")
                + 4 * metric_weightages.get("kurtosis", 0),
                4,
            ),
        )
        .withColumn(
            "flagged_lower",
            F.when(
                (F.col("stability_index_lower_bound") < threshold)
                | (F.col("stability_index_lower_bound").isNull()),
                1,
            ).otherwise(0),
        )
        .withColumn(
            "flagged_upper",
            F.when(
                (F.col("stability_index_upper_bound") < threshold)
                | (F.col("stability_index_upper_bound").isNull()),
                1,
            ).otherwise(0),
        )
    )

    if print_impact:
        logger.info("All Features:")
        odf.show(len(attribute_names), False)
        logger.info(
            "Potential Unstable Features Identified by Both Lower and Upper Bounds:"
        )
        unstable = odf.where(F.col("flagged_upper") == 1)
        unstable.show(unstable.count())

    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.drift.detector.stability_index_computation"><code class="name flex">
<span>def <span class="ident">stability_index_computation</span></span>(<span>spark, *idfs, list_of_cols='all', drop_cols=[], metric_weightages={'mean': 0.5, 'stddev': 0.3, 'kurtosis': 0.2}, existing_metric_path='', appended_metric_path='', threshold=1, print_impact=False)</span>
</code></dt>
<dd>
<div class="desc"><p>The data stability is represented by a single metric to summarise the stability of an attribute over multiple
time periods. For example, given 6 datasets collected in 6 consecutive time periods (D1, D2, …, D6),
data stability index of an attribute measures how stable the attribute is from D1 to D6.</p>
<p>The major difference between data drift and data stability is that data drift analysis is only based on 2
datasets: source and target. However data stability analysis can be performed on multiple datasets. In addition,
the source dataset is not required indicating that the stability index can be directly computed among multiple
target datasets by comparing the statistical properties among them.</p>
<p>In summary, given N datasets representing different time periods, we would like to measure the stability of some
numerical attributes from the first to the N-th dataset.</p>
<p>The whole process can be broken down into 2 steps: (1) Choose a few statistical metrics to describe the
distribution of each attribute at each time period. (2) Compute attribute level stability by combining the
stability of each statistical metric over time periods.</p>
<p>In the first step, we choose mean, standard deviation and kurtosis as the statistical metrics in our
implementation. Intuitively, they represent different aspects of a distribution: mean measures central tendency,
standard deviation measures dispersion and kurtosis measures shape of a distribution. Reasons of selecting those
3 metrics will be explained in a later section. With mean, standard deviation and kurtosis computed for each
attribute at each time interval, we can form 3 arrays of size N for each attribute.</p>
<p>In the second step, Coefficient of Variation (CV) is used to measure the stability of each metric. CV represents
the ratio of the standard deviation to the mean, which is a unitless statistic to compare the relative variation
from one array to another. Considering the wide potential range of CV, the absolute value of CV is then mapped to
an integer between 0 and 4 according to the table below, where 0 indicates highly unstable and 4 indicates highly
stable. We call this integer a metric stability index.</p>
<table>
<thead>
<tr>
<th>abs(CV) Interval</th>
<th>Metric Stability Index</th>
</tr>
</thead>
<tbody>
<tr>
<td>[0, 0.03)</td>
<td>4</td>
</tr>
<tr>
<td>[0.03, 0.1)</td>
<td>3</td>
</tr>
<tr>
<td>[0.1, 0.2)</td>
<td>2</td>
</tr>
<tr>
<td>[0.2, 0.5)</td>
<td>1</td>
</tr>
<tr>
<td>[0.5, +inf)</td>
<td>0</td>
</tr>
</tbody>
</table>
<p>Finally, the attribute stability index (SI) is a weighted sum of 3 metric stability indexes, where we assign 50%
for mean, 30% for standard deviation and 20% for kurtosis. The final output is a float between 0 and 4 and an
attribute can be classified as one of the following categories: very unstable (0≤SI&lt;1), unstable (1≤SI&lt;2),
marginally stable (2≤SI&lt;3), stable (3≤SI&lt;3.5) and very stable (3.5≤SI≤4).</p>
<p>For example, there are 6 samples of attribute X from T1 to T6. For each sample, we have computed the statistical
metrics of X from T1 to T6:</p>
<table>
<thead>
<tr>
<th>idx</th>
<th>Mean</th>
<th>Standard deviation</th>
<th>Kurtosis</th>
</tr>
</thead>
<tbody>
<tr>
<td>1</td>
<td>11</td>
<td>2</td>
<td>3.9</td>
</tr>
<tr>
<td>2</td>
<td>12</td>
<td>1</td>
<td>4.2</td>
</tr>
<tr>
<td>3</td>
<td>15</td>
<td>3</td>
<td>4.0</td>
</tr>
<tr>
<td>4</td>
<td>10</td>
<td>2</td>
<td>4.1</td>
</tr>
<tr>
<td>5</td>
<td>11</td>
<td>1</td>
<td>4.2</td>
</tr>
<tr>
<td>6</td>
<td>13</td>
<td>0.5</td>
<td>4.0</td>
</tr>
</tbody>
</table>
<p>Then we calculate the Coefficient of Variation for each array:</p>
<ul>
<li>CV of mean = CV([11, 12, 15, 10, 11, 13]) = 0.136</li>
<li>CV of standard deviation = CV([2, 1, 3, 2, 1, 0.5]) = 0.529</li>
<li>CV of kurtosis = CV([3.9, 4.2, 4.0, 4.1, 4.2, 4.0]) = 0.027</li>
</ul>
<p>Metric stability indexes are then computed by mapping each CV value to an integer accordingly. As a result,
metric stability index is 2 for mean, 0 for standard deviation and 4 for kurtosis.</p>
<p>Why mean is chosen over median?</p>
<ul>
<li>Dummy variables which take only the value 0 or 1 are frequently seen in machine learning features. Mean of a
dummy variable represents the proportion of value 1 and median of a dummy variable is either 0 or 1 whichever is
more frequent. However, CV may not work well when 0 appears in the array or the array contains both positive and
negative values. For example, intuitively [0,0,0,0,0,1,0,0,0] is a stable array but its CV is 2.83 which is
extremely high, but cv of [0.45,0.44,0.48,0.49,0.42,0.52,0.49,0.47,0.48] is 0.06 which is much more reasonable.
Thus we decided to use mean instead of median. Although median is considered as a more robust choice,
outlier treatment can be applied prior to data stability analysis to handle this issue.</li>
</ul>
<p>Why kurtosis is chosen over skewness?</p>
<ul>
<li>Kurtosis is a positive value (note that we use kurtosis instead of excess kurtosis which) but skewness can
range from –inf to +inf. Usually, if skewness is between -0.5 and 0.5, the distribution is approximately
symmetric. Thus, if the skewness fluctuates around 0, the CV is highly likely to be high or invalid because the
mean will be close to 0.</li>
</ul>
<p>Stability index is preferred in the following scenario:</p>
<ul>
<li>Pairwise drift analysis can be performed between the source dataset and each of the target dataset to quantify
the drift over time. However this can be time-consuming especially when the number of target dataset is large. In
this case, measuring data stability instead of data drift would be a much faster alternative and the
source/baseline dataset is not required as well</li>
</ul>
<p>Troubleshooting</p>
<ul>
<li>If the attribute stability index appears to be nan, it may due to one of the following reasons: - One metric (
likely to be kurtosis) is nan. For example, the kurtosis of a sample is nan If its standard deviation is 0. - The
mean of a metric from the first to the N-th dataset is zero, causing the denominator of CV to be 0. For example,
when mean of attribute X is always zero for all datasets, its stability index would be nan.</li>
</ul>
<p>Limitation</p>
<ul>
<li>Limitation of CV: CV may not work well when 0 appears in the array or the array contains both positive and
negative values.</li>
</ul>
<hr>
<ul>
<li><em>idfs</em>: Input Dataframes (flexible) - <em>list_of_cols</em>: Numerical columns (in list format or string separated by
|). Use ‘all’ - to include all numerical columns (excluding drop_cols). - <em>drop_cols</em>: List of columns to be
dropped (list or string of col names separated by |) - <em>metric_weightages</em>: A dictionary with key being the
metric name (mean,stdev,kurtosis) and value being the weightage of the metric (between 0 and 1). Sum of all
weightages must be 1. - <em>existing_metric_path</em>: this argument is path for pre-existing metrics of historical
datasets
<idx,attribute,mean,stdev,kurtosis>. idx is index number of historical datasets assigned in
chronological order - <em>appended_metric_path</em>: this argument is path for saving input dataframes metrics after
appending to the historical datasets' metrics. - <em>threshold</em>: To flag unstable attributes meeting the threshold</li>
</ul>
<h2 id="parameters">Parameters</h2>
<p>spark :
Spark Session
idfs :
Variable number of input dataframes
list_of_cols :
List of numerical columns to check stability e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
"all" can be passed to include all numerical columns for analysis.
Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
drop_cols :
List of columns to be dropped e.g., ["col1","col2"].
Alternatively, columns can be specified in a string format,
where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
metric_weightages :
Takes input in dictionary format with keys being the metric name - "mean","stdev","kurtosis"
and value being the weightage of the metric (between 0 and 1). Sum of all weightages must be 1.
(Default value = {"mean": 0.5)
existing_metric_path :
This argument is path for referring pre-existing metrics of historical datasets and is
of schema [idx, attribute, mean, stdev, kurtosis].
idx is index number of historical datasets assigned in chronological order. (Default value = "")
appended_metric_path :
This argument is path for saving input dataframes metrics after appending to the
historical datasets' metrics. (Default value = "")
threshold :
A column is flagged if the stability index is below the threshold, which varies between 0 to 4.
The following criteria can be used to classifiy stability_index (SI): very unstable: 0≤SI&lt;1,
unstable: 1≤SI&lt;2, marginally stable: 2≤SI&lt;3, stable: 3≤SI&lt;3.5 and very stable: 3.5≤SI≤4. (Default value = 1)
print_impact :
True, False (Default value = False)
*idfs :</p>
<p>"stddev": 0.3 :</p>
<p>"kurtosis": 0.2} :</p>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
def stability_index_computation(
    spark,
    *idfs,
    list_of_cols="all",
    drop_cols=[],
    metric_weightages={"mean": 0.5, "stddev": 0.3, "kurtosis": 0.2},
    existing_metric_path="",
    appended_metric_path="",
    threshold=1,
    print_impact=False,
):
    """The data stability is represented by a single metric to summarise the stability of an attribute over multiple
    time periods. For example, given 6 datasets collected in 6 consecutive time periods (D1, D2, …, D6),
    data stability index of an attribute measures how stable the attribute is from D1 to D6.

    The major difference between data drift and data stability is that data drift analysis is only based on 2
    datasets: source and target. However data stability analysis can be performed on multiple datasets. In addition,
    the source dataset is not required indicating that the stability index can be directly computed among multiple
    target datasets by comparing the statistical properties among them.

    In summary, given N datasets representing different time periods, we would like to measure the stability of some
    numerical attributes from the first to the N-th dataset.

    The whole process can be broken down into 2 steps: (1) Choose a few statistical metrics to describe the
    distribution of each attribute at each time period. (2) Compute attribute level stability by combining the
    stability of each statistical metric over time periods.

    In the first step, we choose mean, standard deviation and kurtosis as the statistical metrics in our
    implementation. Intuitively, they represent different aspects of a distribution: mean measures central tendency,
    standard deviation measures dispersion and kurtosis measures shape of a distribution. Reasons of selecting those
    3 metrics will be explained in a later section. With mean, standard deviation and kurtosis computed for each
    attribute at each time interval, we can form 3 arrays of size N for each attribute.

    In the second step, Coefficient of Variation (CV) is used to measure the stability of each metric. CV represents
    the ratio of the standard deviation to the mean, which is a unitless statistic to compare the relative variation
    from one array to another. Considering the wide potential range of CV, the absolute value of CV is then mapped to
    an integer between 0 and 4 according to the table below, where 0 indicates highly unstable and 4 indicates highly
    stable. We call this integer a metric stability index.

    | abs(CV) Interval | Metric Stability Index |
    |------------------|------------------------|
    | [0, 0.03)        | 4                      |
    | [0.03, 0.1)      | 3                      |
    | [0.1, 0.2)       | 2                      |
    | [0.2, 0.5)       | 1                      |
    | [0.5, +inf)      | 0                      |


    Finally, the attribute stability index (SI) is a weighted sum of 3 metric stability indexes, where we assign 50%
    for mean, 30% for standard deviation and 20% for kurtosis. The final output is a float between 0 and 4 and an
    attribute can be classified as one of the following categories: very unstable (0≤SI<1), unstable (1≤SI<2),
    marginally stable (2≤SI<3), stable (3≤SI<3.5) and very stable (3.5≤SI≤4).

    For example, there are 6 samples of attribute X from T1 to T6. For each sample, we have computed the statistical
    metrics of X from T1 to T6:

    | idx | Mean | Standard deviation | Kurtosis |
    |-----|------|--------------------|----------|
    | 1   | 11   | 2                  | 3.9      |
    | 2   | 12   | 1                  | 4.2      |
    | 3   | 15   | 3                  | 4.0      |
    | 4   | 10   | 2                  | 4.1      |
    | 5   | 11   | 1                  | 4.2      |
    | 6   | 13   | 0.5                | 4.0      |

    Then we calculate the Coefficient of Variation for each array:

    - CV of mean = CV([11, 12, 15, 10, 11, 13]) = 0.136
    - CV of standard deviation = CV([2, 1, 3, 2, 1, 0.5]) = 0.529
    - CV of kurtosis = CV([3.9, 4.2, 4.0, 4.1, 4.2, 4.0]) = 0.027

    Metric stability indexes are then computed by mapping each CV value to an integer accordingly. As a result,
    metric stability index is 2 for mean, 0 for standard deviation and 4 for kurtosis.

    Why mean is chosen over median?

    - Dummy variables which take only the value 0 or 1 are frequently seen in machine learning features. Mean of a
    dummy variable represents the proportion of value 1 and median of a dummy variable is either 0 or 1 whichever is
    more frequent. However, CV may not work well when 0 appears in the array or the array contains both positive and
    negative values. For example, intuitively [0,0,0,0,0,1,0,0,0] is a stable array but its CV is 2.83 which is
    extremely high, but cv of [0.45,0.44,0.48,0.49,0.42,0.52,0.49,0.47,0.48] is 0.06 which is much more reasonable.
    Thus we decided to use mean instead of median. Although median is considered as a more robust choice,
    outlier treatment can be applied prior to data stability analysis to handle this issue.

    Why kurtosis is chosen over skewness?

    - Kurtosis is a positive value (note that we use kurtosis instead of excess kurtosis which) but skewness can
    range from –inf to +inf. Usually, if skewness is between -0.5 and 0.5, the distribution is approximately
    symmetric. Thus, if the skewness fluctuates around 0, the CV is highly likely to be high or invalid because the
    mean will be close to 0.

    Stability index is preferred in the following scenario:

    - Pairwise drift analysis can be performed between the source dataset and each of the target dataset to quantify
    the drift over time. However this can be time-consuming especially when the number of target dataset is large. In
    this case, measuring data stability instead of data drift would be a much faster alternative and the
    source/baseline dataset is not required as well

    Troubleshooting

    - If the attribute stability index appears to be nan, it may due to one of the following reasons: - One metric (
    likely to be kurtosis) is nan. For example, the kurtosis of a sample is nan If its standard deviation is 0. - The
    mean of a metric from the first to the N-th dataset is zero, causing the denominator of CV to be 0. For example,
    when mean of attribute X is always zero for all datasets, its stability index would be nan.

    Limitation

    - Limitation of CV: CV may not work well when 0 appears in the array or the array contains both positive and
    negative values.

    ------

    - *idfs*: Input Dataframes (flexible) - *list_of_cols*: Numerical columns (in list format or string separated by
    |). Use ‘all’ - to include all numerical columns (excluding drop_cols). - *drop_cols*: List of columns to be
    dropped (list or string of col names separated by |) - *metric_weightages*: A dictionary with key being the
    metric name (mean,stdev,kurtosis) and value being the weightage of the metric (between 0 and 1). Sum of all
    weightages must be 1. - *existing_metric_path*: this argument is path for pre-existing metrics of historical
    datasets  <idx,attribute,mean,stdev,kurtosis>. idx is index number of historical datasets assigned in
    chronological order - *appended_metric_path*: this argument is path for saving input dataframes metrics after
    appending to the historical datasets' metrics. - *threshold*: To flag unstable attributes meeting the threshold


    Parameters
    ----------
    spark :
        Spark Session
    idfs :
        Variable number of input dataframes
    list_of_cols :
        List of numerical columns to check stability e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all numerical columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols. (Default value = "all")
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2". (Default value = [])
    metric_weightages :
        Takes input in dictionary format with keys being the metric name - "mean","stdev","kurtosis"
        and value being the weightage of the metric (between 0 and 1). Sum of all weightages must be 1.
         (Default value = {"mean": 0.5)
    existing_metric_path :
        This argument is path for referring pre-existing metrics of historical datasets and is
        of schema [idx, attribute, mean, stdev, kurtosis].
        idx is index number of historical datasets assigned in chronological order. (Default value = "")
    appended_metric_path :
        This argument is path for saving input dataframes metrics after appending to the
        historical datasets' metrics. (Default value = "")
    threshold :
        A column is flagged if the stability index is below the threshold, which varies between 0 to 4.
        The following criteria can be used to classifiy stability_index (SI): very unstable: 0≤SI<1,
        unstable: 1≤SI<2, marginally stable: 2≤SI<3, stable: 3≤SI<3.5 and very stable: 3.5≤SI≤4. (Default value = 1)
    print_impact :
        True, False (Default value = False)
    *idfs :

    "stddev": 0.3 :

    "kurtosis": 0.2} :


    Returns
    -------

    """

    num_cols = attributeType_segregation(idfs[0])[0]
    if list_of_cols == "all":
        list_of_cols = num_cols
    if isinstance(list_of_cols, str):
        list_of_cols = [x.strip() for x in list_of_cols.split("|")]
    if isinstance(drop_cols, str):
        drop_cols = [x.strip() for x in drop_cols.split("|")]

    list_of_cols = list(set([e for e in list_of_cols if e not in drop_cols]))

    if any(x not in num_cols for x in list_of_cols) | (len(list_of_cols) == 0):
        raise TypeError("Invalid input for Column(s)")

    if (
        round(
            metric_weightages.get("mean", 0)
            + metric_weightages.get("stddev", 0)
            + metric_weightages.get("kurtosis", 0),
            3,
        )
        != 1
    ):
        raise ValueError(
            "Invalid input for metric weightages. Either metric name is incorrect or sum of metric weightages is not 1.0"
        )

    if existing_metric_path:
        existing_metric_df = spark.read.csv(
            existing_metric_path, header=True, inferSchema=True
        )
        dfs_count = existing_metric_df.select(F.max(F.col("idx"))).first()[0]
    else:
        schema = T.StructType(
            [
                T.StructField("idx", T.IntegerType(), True),
                T.StructField("attribute", T.StringType(), True),
                T.StructField("mean", T.DoubleType(), True),
                T.StructField("stddev", T.DoubleType(), True),
                T.StructField("kurtosis", T.DoubleType(), True),
            ]
        )
        existing_metric_df = spark.sparkContext.emptyRDD().toDF(schema)
        dfs_count = 0

    metric_ls = []
    for idf in idfs:
        for i in list_of_cols:
            mean, stddev, kurtosis = idf.select(
                F.mean(i), F.stddev(i), F.kurtosis(i)
            ).first()
            metric_ls.append(
                [dfs_count + 1, i, mean, stddev, kurtosis + 3.0 if kurtosis else None]
            )
        dfs_count += 1

    new_metric_df = spark.createDataFrame(
        metric_ls, schema=("idx", "attribute", "mean", "stddev", "kurtosis")
    )
    appended_metric_df = concatenate_dataset(existing_metric_df, new_metric_df)

    if appended_metric_path:
        appended_metric_df.coalesce(1).write.csv(
            appended_metric_path, header=True, mode="overwrite"
        )

    result = []
    for i in list_of_cols:
        i_output = [i]
        for metric in ["mean", "stddev", "kurtosis"]:
            metric_stats = (
                appended_metric_df.where(F.col("attribute") == i)
                .orderBy("idx")
                .select(metric)
                .fillna(np.nan)
                .rdd.flatMap(list)
                .collect()
            )
            metric_cv = round(float(variation([a for a in metric_stats])), 4) or None
            i_output.append(metric_cv)
        result.append(i_output)

    schema = T.StructType(
        [
            T.StructField("attribute", T.StringType(), True),
            T.StructField("mean_cv", T.FloatType(), True),
            T.StructField("stddev_cv", T.FloatType(), True),
            T.StructField("kurtosis_cv", T.FloatType(), True),
        ]
    )

    odf = spark.createDataFrame(result, schema=schema)

    def score_cv(cv, thresholds=[0.03, 0.1, 0.2, 0.5]):
        """

        Parameters
        ----------
        cv :
            param thresholds: (Default value = [0.03)
        0 :
            1:
        0 :
            2:
        0 :
            5]:
        thresholds :
             (Default value = [0.03)
        0.1 :

        0.2 :

        0.5] :


        Returns
        -------

        """
        if cv is None:
            return None
        else:
            cv = abs(cv)
            stability_index = [4, 3, 2, 1, 0]
            for i, thresh in enumerate(thresholds):
                if cv < thresh:
                    return stability_index[i]
            return stability_index[-1]

    f_score_cv = F.udf(score_cv, T.IntegerType())

    odf = (
        odf.replace(np.nan, None)
        .withColumn("mean_si", f_score_cv(F.col("mean_cv")))
        .withColumn("stddev_si", f_score_cv(F.col("stddev_cv")))
        .withColumn("kurtosis_si", f_score_cv(F.col("kurtosis_cv")))
        .withColumn(
            "stability_index",
            F.round(
                (
                    F.col("mean_si") * metric_weightages.get("mean", 0)
                    + F.col("stddev_si") * metric_weightages.get("stddev", 0)
                    + F.col("kurtosis_si") * metric_weightages.get("kurtosis", 0)
                ),
                4,
            ),
        )
        .withColumn(
            "flagged",
            F.when(
                (F.col("stability_index") < threshold)
                | (F.col("stability_index").isNull()),
                1,
            ).otherwise(0),
        )
    )

    if print_impact:
        logger.info("All Attributes:")
        odf.show(len(list_of_cols))
        logger.info("Potential Unstable Attributes:")
        unstable = odf.where(F.col("flagged") == 1)
        unstable.show(unstable.count())

    return odf
```
</pre>
</details>
</dd>
<dt id="anovos.drift.detector.statistics"><code class="name flex">
<span>def <span class="ident">statistics</span></span>(<span>spark: pyspark.sql.session.SparkSession, idf_target: pyspark.sql.dataframe.DataFrame, idf_source: pyspark.sql.dataframe.DataFrame, *, list_of_cols: list = 'all', drop_cols: list = None, method_type: str = 'PSI', bin_method: str = 'equal_range', bin_size: int = 10, threshold: float = 0.1, pre_existing_source: bool = False, source_path: str = 'NA', model_directory: str = 'drift_statistics', print_impact: bool = False)</span>
</code></dt>
<dd>
<div class="desc"><p>When the performance of a deployed machine learning model degrades in production, one potential reason is that
the data used in training and prediction are not following the same distribution.</p>
<p>Data drift mainly includes the following manifestations:</p>
<ul>
<li>Covariate shift: training and test data follow different distributions. For example, An algorithm predicting
income that is trained on younger population but tested on older population. - Prior probability shift: change of
prior probability. For example in a spam classification problem, the proportion of spam emails changes from 0.2
in training data to 0.6 in testing data. - Concept shift: the distribution of the target variable changes given
fixed input values. For example in the same spam classification problem, emails tagged as spam in training data
are more likely to be tagged as non-spam in testing data.</li>
</ul>
<p>In our module, we mainly focus on covariate shift detection.</p>
<p>In summary, given 2 datasets, source and target datasets, we would like to quantify the drift of some numerical
attributes from source to target datasets. The whole process can be broken down into 2 steps: (1) convert each
attribute of interest in source and target datasets into source and target probability distributions. (2)
calculate the statistical distance between source and target distributions for each attribute.</p>
<p>In the first step, attribute_binning is firstly performed to bin the numerical attributes of the source dataset,
which requires two input variables: bin_method and bin_size. The same binning method is applied on the target
dataset to align two results. The probability distributions are computed by dividing the frequency of each bin by
the total frequency.</p>
<p>In the second step, 4 choices of statistical metrics are provided to measure the data drift of an attribute from
source to target distribution: Population Stability Index (PSI), Jensen-Shannon Divergence (JSD),
Hellinger Distance (HD) and Kolmogorov-Smirnov Distance (KS).</p>
<p>They are calculated as below:
For two discrete probability distributions <em>P=(p_1,…,p_k)</em> and <em>Q=(q_1,…,q_k),</em></p>
<p><img alt="https://raw.githubusercontent.com/anovos/anovos-docs/main/docs/assets/drift_stats_formulae.png" src="https://raw.githubusercontent.com/anovos/anovos-docs/main/docs/assets/drift_stats_formulae.png"></p>
<p>A threshold can be set to flag out drifted attributes. If multiple statistical metrics have been calculated,
an attribute will be marked as drifted if any of its statistical metric is larger than the threshold.</p>
<p>This function can be used in many scenarios. For example:</p>
<ol>
<li>Attribute level data drift can be analysed together with the attribute importance of a machine learning model.
The more important an attribute is, the more attention it needs to be given if drift presents. 2. To analyse data
drift over time, one can treat one dataset as the source / baseline dataset and multiple datasets as the target
datasets. Drift analysis can be performed between the source dataset and each of the target dataset to quantify
the drift over time.</li>
</ol>
<hr>
<ul>
<li><em>idf_target</em>: Input target Dataframe - <em>idf_source</em>: Input source Dataframe - <em>list_of_cols</em>: List of columns
to check drift (list or string of col names separated by |). Use ‘all’ - to include all non-array columns (
excluding drop_cols). - <em>drop_cols</em>: List of columns to be dropped (list or string of col names separated by |) -
method: PSI, JSD, HD, KS (list or string of methods separated by |). Use ‘all’ - to calculate all metrics. -
<em>bin_method</em>: equal_frequency or equal_range - <em>bin_size</em>: 10 - 20 (recommended for PSI), &gt;100 (other method
types) - <em>threshold</em>: To flag attributes meeting drift threshold - <em>pre_existing_source</em>: True if binning model &amp;
frequency counts/attribute exists already, False Otherwise. - <em>source_path</em>: If pre_existing_source is True,
this argument is path for the source dataset details - drift_statistics folder. drift_statistics folder must
contain attribute_binning &amp; frequency_counts folders. If pre_existing_source is False, this argument can be used
for saving the details. Default "NA" for temporarily saving source dataset attribute_binning folder</li>
</ul>
<h2 id="parameters">Parameters</h2>
<dl>
<dt>spark :</dt>
<dt>Spark Session</dt>
<dt>idf_target :</dt>
<dt>Input Dataframe</dt>
<dt>idf_source :</dt>
<dt>Baseline/Source Dataframe. This argument is ignored if pre_existing_source is True.</dt>
<dt>list_of_cols :</dt>
<dt>List of columns to check drift e.g., ["col1","col2"].</dt>
<dt>Alternatively, columns can be specified in a string format,</dt>
<dt>where different column names are separated by pipe delimiter “|” e.g., "col1|col2".</dt>
<dt>"all" can be passed to include all (non-array) columns for analysis.</dt>
<dt>Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in</dt>
<dt>drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols.</dt>
<dt>drop_cols :</dt>
<dt>List of columns to be dropped e.g., ["col1","col2"].</dt>
<dt>Alternatively, columns can be specified in a string format,</dt>
<dt>where different column names are separated by pipe delimiter “|” e.g., "col1|col2".</dt>
<dt>method_type :</dt>
<dt>PSI", "JSD", "HD", "KS","all".</dt>
<dt>"all" can be passed to calculate all drift metrics.</dt>
<dt>One or more methods can be passed in a form of list or string where different metrics are separated</dt>
<dt>by pipe delimiter “|” e.g. ["PSI", "JSD"] or "PSI|JSD"</dt>
<dt>bin_method :</dt>
<dt>equal_frequency", "equal_range".</dt>
<dt>In "equal_range" method, each bin is of equal size/width and in "equal_frequency", each bin</dt>
<dt>has equal no. of rows, though the width of bins may vary.</dt>
<dt>bin_size :</dt>
<dt>Number of bins for creating histogram</dt>
<dt>threshold :</dt>
<dt>A column is flagged if any drift metric is above the threshold.</dt>
<dt>pre_existing_source :</dt>
<dt>Boolean argument – True or False. True if the drift_statistics folder (binning model &amp;</dt>
<dt>frequency counts for each attribute) exists already, False Otherwise.</dt>
<dt>source_path :</dt>
<dt>If pre_existing_source is False, this argument can be used for saving the drift_statistics folder.</dt>
<dt>The drift_statistics folder will have attribute_binning (binning model) &amp; frequency_counts sub-folders.</dt>
<dt>If pre_existing_source is True, this argument is path for referring the drift_statistics folder.</dt>
<dt>Default "NA" for temporarily saving source dataset attribute_binning folder.</dt>
<dt>model_directory :</dt>
<dt>If pre_existing_source is False, this argument can be used for saving the drift stats to folder.</dt>
<dt>The default drift statics directory is drift_statistics folder will have attribute_binning</dt>
<dt>If pre_existing_source is True, this argument is model_directory for referring the drift statistics dir.</dt>
<dt>Default "drift_statistics" for temporarily saving source dataset attribute_binning folder.</dt>
<dt>print_impact :</dt>
<dt>True, False</dt>
<dt>spark :</dt>
<dt>type spark: SparkSession :</dt>
<dt>idf_target :</dt>
<dt>type idf_target: DataFrame :</dt>
<dt>idf_source :</dt>
<dt>type idf_source: DataFrame :</dt>
<dt><strong><code>spark</code></strong> :&ensp;<code>SparkSession :</code></dt>
<dd>&nbsp;</dd>
<dt><strong><code>idf_target</code></strong> :&ensp;<code>DataFrame :</code></dt>
<dd>&nbsp;</dd>
<dt><strong><code>idf_source</code></strong> :&ensp;<code>DataFrame :</code></dt>
<dd>&nbsp;</dd>
</dl>
<ul>
<li>:</li>
</ul>
<dl>
<dt><strong><code>list_of_cols</code></strong> :&ensp;<code>list :</code></dt>
<dd>(Default value = "all")</dd>
<dt><strong><code>drop_cols</code></strong> :&ensp;<code>list :</code></dt>
<dd>(Default value = None)</dd>
<dt><strong><code>method_type</code></strong> :&ensp;<code>str :</code></dt>
<dd>(Default value = "PSI")</dd>
<dt><strong><code>bin_method</code></strong> :&ensp;<code>str :</code></dt>
<dd>(Default value = "equal_range")</dd>
<dt><strong><code>bin_size</code></strong> :&ensp;<code>int :</code></dt>
<dd>(Default value = 10)</dd>
<dt><strong><code>threshold</code></strong> :&ensp;<code>float :</code></dt>
<dd>(Default value = 0.1)</dd>
<dt><strong><code>pre_existing_source</code></strong> :&ensp;<code>bool :</code></dt>
<dd>(Default value = False)</dd>
<dt><strong><code>source_path</code></strong> :&ensp;<code>str :</code></dt>
<dd>(Default value = "NA")</dd>
<dt><strong><code>model_directory</code></strong> :&ensp;<code>str :</code></dt>
<dd>(Default value = "drift_statistics")</dd>
<dt><strong><code>print_impact</code></strong> :&ensp;<code>bool :</code></dt>
<dd>(Default value = False)</dd>
</dl>
<h2 id="returns">Returns</h2></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre>
```python
@check_distance_method
@check_list_of_columns
def statistics(
    spark: SparkSession,
    idf_target: DataFrame,
    idf_source: DataFrame,
    *,
    list_of_cols: list = "all",
    drop_cols: list = None,
    method_type: str = "PSI",
    bin_method: str = "equal_range",
    bin_size: int = 10,
    threshold: float = 0.1,
    pre_existing_source: bool = False,
    source_path: str = "NA",
    model_directory: str = "drift_statistics",
    print_impact: bool = False,
):
    """When the performance of a deployed machine learning model degrades in production, one potential reason is that
    the data used in training and prediction are not following the same distribution.

    Data drift mainly includes the following manifestations:

    - Covariate shift: training and test data follow different distributions. For example, An algorithm predicting
    income that is trained on younger population but tested on older population. - Prior probability shift: change of
    prior probability. For example in a spam classification problem, the proportion of spam emails changes from 0.2
    in training data to 0.6 in testing data. - Concept shift: the distribution of the target variable changes given
    fixed input values. For example in the same spam classification problem, emails tagged as spam in training data
    are more likely to be tagged as non-spam in testing data.

    In our module, we mainly focus on covariate shift detection.

    In summary, given 2 datasets, source and target datasets, we would like to quantify the drift of some numerical
    attributes from source to target datasets. The whole process can be broken down into 2 steps: (1) convert each
    attribute of interest in source and target datasets into source and target probability distributions. (2)
    calculate the statistical distance between source and target distributions for each attribute.

    In the first step, attribute_binning is firstly performed to bin the numerical attributes of the source dataset,
    which requires two input variables: bin_method and bin_size. The same binning method is applied on the target
    dataset to align two results. The probability distributions are computed by dividing the frequency of each bin by
    the total frequency.

    In the second step, 4 choices of statistical metrics are provided to measure the data drift of an attribute from
    source to target distribution: Population Stability Index (PSI), Jensen-Shannon Divergence (JSD),
    Hellinger Distance (HD) and Kolmogorov-Smirnov Distance (KS).

    They are calculated as below:
    For two discrete probability distributions *P=(p_1,…,p_k)* and *Q=(q_1,…,q_k),*

    ![https://raw.githubusercontent.com/anovos/anovos-docs/main/docs/assets/drift_stats_formulae.png](https://raw.githubusercontent.com/anovos/anovos-docs/main/docs/assets/drift_stats_formulae.png)

    A threshold can be set to flag out drifted attributes. If multiple statistical metrics have been calculated,
    an attribute will be marked as drifted if any of its statistical metric is larger than the threshold.

    This function can be used in many scenarios. For example:

    1. Attribute level data drift can be analysed together with the attribute importance of a machine learning model.
    The more important an attribute is, the more attention it needs to be given if drift presents. 2. To analyse data
    drift over time, one can treat one dataset as the source / baseline dataset and multiple datasets as the target
    datasets. Drift analysis can be performed between the source dataset and each of the target dataset to quantify
    the drift over time.

    ---------

    - *idf_target*: Input target Dataframe - *idf_source*: Input source Dataframe - *list_of_cols*: List of columns
    to check drift (list or string of col names separated by |). Use ‘all’ - to include all non-array columns (
    excluding drop_cols). - *drop_cols*: List of columns to be dropped (list or string of col names separated by |) -
    method: PSI, JSD, HD, KS (list or string of methods separated by |). Use ‘all’ - to calculate all metrics. -
    *bin_method*: equal_frequency or equal_range - *bin_size*: 10 - 20 (recommended for PSI), >100 (other method
    types) - *threshold*: To flag attributes meeting drift threshold - *pre_existing_source*: True if binning model &
    frequency counts/attribute exists already, False Otherwise. - *source_path*: If pre_existing_source is True,
    this argument is path for the source dataset details - drift_statistics folder. drift_statistics folder must
    contain attribute_binning & frequency_counts folders. If pre_existing_source is False, this argument can be used
    for saving the details. Default "NA" for temporarily saving source dataset attribute_binning folder

    Parameters
    ----------
    spark :
        Spark Session
    idf_target :
        Input Dataframe
    idf_source :
        Baseline/Source Dataframe. This argument is ignored if pre_existing_source is True.
    list_of_cols :
        List of columns to check drift e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
        "all" can be passed to include all (non-array) columns for analysis.
        Please note that this argument is used in conjunction with drop_cols i.e. a column mentioned in
        drop_cols argument is not considered for analysis even if it is mentioned in list_of_cols.
    drop_cols :
        List of columns to be dropped e.g., ["col1","col2"].
        Alternatively, columns can be specified in a string format,
        where different column names are separated by pipe delimiter “|” e.g., "col1|col2".
    method_type :
        PSI", "JSD", "HD", "KS","all".
        "all" can be passed to calculate all drift metrics.
        One or more methods can be passed in a form of list or string where different metrics are separated
        by pipe delimiter “|” e.g. ["PSI", "JSD"] or "PSI|JSD"
    bin_method :
        equal_frequency", "equal_range".
        In "equal_range" method, each bin is of equal size/width and in "equal_frequency", each bin
        has equal no. of rows, though the width of bins may vary.
    bin_size :
        Number of bins for creating histogram
    threshold :
        A column is flagged if any drift metric is above the threshold.
    pre_existing_source :
        Boolean argument – True or False. True if the drift_statistics folder (binning model &
        frequency counts for each attribute) exists already, False Otherwise.
    source_path :
        If pre_existing_source is False, this argument can be used for saving the drift_statistics folder.
        The drift_statistics folder will have attribute_binning (binning model) & frequency_counts sub-folders.
        If pre_existing_source is True, this argument is path for referring the drift_statistics folder.
        Default "NA" for temporarily saving source dataset attribute_binning folder.
    model_directory :
        If pre_existing_source is False, this argument can be used for saving the drift stats to folder.
        The default drift statics directory is drift_statistics folder will have attribute_binning
        If pre_existing_source is True, this argument is model_directory for referring the drift statistics dir.
        Default "drift_statistics" for temporarily saving source dataset attribute_binning folder.
    print_impact :
        True, False
    spark :
        type spark: SparkSession :
    idf_target :
        type idf_target: DataFrame :
    idf_source :
        type idf_source: DataFrame :
    spark: SparkSession :

    idf_target: DataFrame :

    idf_source: DataFrame :

    * :

    list_of_cols: list :
         (Default value = "all")
    drop_cols: list :
         (Default value = None)
    method_type: str :
         (Default value = "PSI")
    bin_method: str :
         (Default value = "equal_range")
    bin_size: int :
         (Default value = 10)
    threshold: float :
         (Default value = 0.1)
    pre_existing_source: bool :
         (Default value = False)
    source_path: str :
         (Default value = "NA")
    model_directory: str :
         (Default value = "drift_statistics")
    print_impact: bool :
         (Default value = False)

    Returns
    -------

    """
    drop_cols = drop_cols or []
    num_cols = attributeType_segregation(idf_target.select(list_of_cols))[0]

    if not pre_existing_source:
        source_bin = attribute_binning(
            spark,
            idf_source,
            list_of_cols=num_cols,
            method_type=bin_method,
            bin_size=bin_size,
            pre_existing_model=False,
            model_path=source_path + "/" + model_directory,
        )
        source_bin.persist(pyspark.StorageLevel.MEMORY_AND_DISK).count()

    target_bin = attribute_binning(
        spark,
        idf_target,
        list_of_cols=num_cols,
        method_type=bin_method,
        bin_size=bin_size,
        pre_existing_model=True,
        model_path=source_path + "/" + model_directory,
    )

    target_bin.persist(pyspark.StorageLevel.MEMORY_AND_DISK).count()
    result = {"attribute": [], "flagged": []}

    for method in method_type:
        result[method] = []

    for i in list_of_cols:
        if pre_existing_source:
            x = spark.read.csv(
                source_path + "/" + model_directory + "/frequency_counts/" + i,
                header=True,
                inferSchema=True,
            )
        else:
            x = (
                source_bin.groupBy(i)
                .agg((F.count(i) / idf_source.count()).alias("p"))
                .fillna(-1)
            )
            x.coalesce(1).write.csv(
                source_path + "/" + model_directory + "/frequency_counts/" + i,
                header=True,
                mode="overwrite",
            )

        y = (
            target_bin.groupBy(i)
            .agg((F.count(i) / idf_target.count()).alias("q"))
            .fillna(-1)
        )

        xy = (
            x.join(y, i, "full_outer")
            .fillna(0.0001, subset=["p", "q"])
            .replace(0, 0.0001)
            .orderBy(i)
        )
        p = np.array(xy.select("p").rdd.flatMap(lambda x: x).collect())
        q = np.array(xy.select("q").rdd.flatMap(lambda x: x).collect())

        result["attribute"].append(i)
        counter = 0

        for idx, method in enumerate(method_type):
            drift_function = {
                "PSI": psi,
                "JSD": js_divergence,
                "HD": hellinger,
                "KS": ks,
            }
            metric = float(round(drift_function[method](p, q), 4))
            result[method].append(metric)
            if counter == 0:
                if metric > threshold:
                    result["flagged"].append(1)
                    counter = 1
            if (idx == (len(method_type) - 1)) & (counter == 0):
                result["flagged"].append(0)

    odf = (
        spark.createDataFrame(
            pd.DataFrame.from_dict(result, orient="index").transpose()
        )
        .select(["attribute"] + method_type + ["flagged"])
        .orderBy(F.desc("flagged"))
    )

    if print_impact:
        logger.info("All Attributes:")
        odf.show(len(list_of_cols))
        logger.info("Attributes meeting Data Drift threshold:")
        drift = odf.where(F.col("flagged") == 1)
        drift.show(drift.count())

    return odf
```
</pre>
</details>
</dd>
</dl>