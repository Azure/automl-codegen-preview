# Breakdown of generated code

Let's take a look at an example of code generated from a model by AutoML. The example files referenced can be found in the `example` folder in this repo.

## `script.py`
`script.py` contains the core logic needed to train a model using the previously used hyperparameters.
While intended to be executed in the context of an AzureML script run, with some modifications it can also be run standalone.

The script can roughly be broken down into several different parts: data loading, data preparation, data featurization, preprocessor and model specification,
then training.

#### Data loading
The function `get_training_dataset()` loads the previously used dataset. It assumes that the script is run in an AzureML script run under the same workspace
as the original experiment.

```python
def get_training_dataset():
    from azureml.core import Dataset, Workspace, Run
    
    ws = Run.get_context().experiment.workspace
    dataset = Dataset.get_by_id(workspace=ws, id='dataset_id')
    return dataset.to_pandas_dataframe()
```

When running as part of a script run, `Run.get_context().experiment.workspace` retrieves the correct workspace. However, if this script is run inside of a 
different workspace or run locally without using `ScriptRunConfig`, [you will need to modify it to specify the appropriate workspace explicitly](https://docs.microsoft.com/en-us/python/api/azureml-core/azureml.core.workspace.workspace?view=azure-ml-py).

Once the workspace has been retrieved, the original dataset is retrieved by its ID. Another dataset can also be specified, either using [`get_by_id()`](https://docs.microsoft.com/en-us/python/api/azureml-core/azureml.core.dataset.dataset?view=azure-ml-py#get-by-id-workspace--id-) if you know
its ID, or [`get_by_name()`](https://docs.microsoft.com/en-us/python/api/azureml-core/azureml.core.dataset.dataset?view=azure-ml-py#get-by-name-workspace--name--version--latest--) if you know its name.

You can also opt to replace this entire function with your own data loading mechanism; the only constraints are that the return value must be a Pandas dataframe and
that the data must have the same shape as in the original experiment.

#### Data preparation
The function `prepare_data()` cleans the data, splits out the feature and sample weight columns and prepares the data for use in training.
```python
def prepare_data(dataframe):
    from azureml.automl.runtime import data_cleaning
    
    label_column_name = 'demand'
    
    # extract the features, target and sample weight arrays
    y_train = dataframe[label_column_name].values
    X_train = dataframe.drop([label_column_name], axis=1)
    sample_weights = None
    X_train, y_train, sample_weights = data_cleaning._remove_nan_rows_in_X_y(X_train, y_train, sample_weights, is_timeseries=True, target_column=label_column_name)
    return X_train, y_train, sample_weights
````
Here, the dataframe from the data loading step is passed in. The label column (as well as sample weights, if originally specified) is extracted, then rows containing NaN are dropped from the input data.

If additional data preparation is desired, it can be done in this step.

#### Data featurization
The function `generate_data_transformation_config()` specifies the featurization step in the final scikit-learn pipeline. The featurizers used by AutoML
in the original experiment are reproduced here, along with their parameters.

In the example `script.py`, multiple timeseries-aware featurizers are collected into a scikit-learn pipeline, then wrapped in the [`TimeSeriesTransformer`](https://docs.microsoft.com/en-us/python/api/azureml-automl-runtime/azureml.automl.runtime.featurizer.transformer.timeseries.timeseries_transformer.timeseriestransformer?view=azure-ml-py).

With a classification/regression task, featurizers are instead combined with corresponding [`DataFrameMappers`](https://github.com/scikit-learn-contrib/sklearn-pandas) into [`TransformerAndMapper`](https://docs.microsoft.com/en-us/python/api/azureml-automl-runtime/azureml.automl.runtime.featurization.transformer_and_mapper.transformerandmapper?view=azure-ml-py) objects,
and then these combined objects are wrapped in the [`DataTransformer`](https://docs.microsoft.com/en-us/python/api/azureml-automl-runtime/azureml.automl.runtime.featurization.data_transformer.datatransformer?view=azure-ml-py).

#### Preprocessor specification
The function `generate_preprocessor_config()`, if present, specifies a preprocessing step to be done after featurization in the final scikit-learn pipeline.
Normally, this preprocessing step only consists of data standardization/normalization using [`sklearn.preprocessing`](https://scikit-learn.org/stable/modules/preprocessing.html).

In the example `script.py`, this step is not present; AutoML only specifies a preprocessing step for non-ensemble classification and regression models.

#### Model specification
`generate_algorithm_config()` specifies the actual algorithm and hyperparameters for the model to be used as the last stage of the final scikit-learn pipeline.
```python
def generate_algorithm_config():
    from azureml.automl.runtime.shared._exponential_smoothing import ExponentialSmoothing
    from numpy import array
    
    algorithm = ExponentialSmoothing(
        timeseries_param_dict={'time_column_name': 'timeStamp', 'grain_column_names': None, 'drop_column_names': [], 'overwrite_columns': True, 'dropna': False, 'transform_dictionary': {'min': '_automl_target_col', 'max': '_automl_target_col', 'mean': '_automl_target_col'}, 'max_horizon': 48, 'origin_time_colname': 'origin', 'country_or_region': None, 'n_cross_validations': 3, 'short_series_handling': True, 'max_cores_per_iteration': 1, 'feature_lags': None, 'target_aggregation_function': None, 'seasonality': 24, 'use_stl': None, 'freq': 'H', 'short_series_handling_configuration': 'auto', 'target_lags': [0], 'target_rolling_window_size': 0, 'arimax_raw_columns': ['precip', 'timeStamp', 'temp']}
    )
    
    return algorithm
```
For the example, the final model used `ExponentialSmoothing`, so it is reproduced with its hyperparameters here.

In the case of ensemble models, `generate_preprocessor_config_N()` (if needed) and `generate_algorithm_config_N()` will be defined for each learner in the ensemble model,
where `N` represents the placement of each learner in the ensemble model's list. In addition, `generate_algorithm_config_meta()` will be defined in the case of
stack ensemble models for the meta learner.

#### Training
Code generation emits `build_model_pipeline()` and `train_model()` for defining the scikit-learn pipeline and for calling `fit()` on it, respectively.

```python
def build_model_pipeline():
    from sklearn.pipeline import Pipeline
    from azureml.automl.runtime.shared.model_wrappers import ForecastingPipelineWrapper
    
    pipeline = Pipeline(
        steps=[('tst', generate_data_transformation_config()),
               ('model', generate_algorithm_config())]
    )
    forecast_pipeline_wrapper = ForecastingPipelineWrapper(pipeline, stddev=[515.970279075003])
    
    return forecast_pipeline_wrapper
```

Here, the scikit-learn pipeline includes the featurization step and then the algorithm. If a preprocessor was used, it would also be included here.

In the case of timeseries models, the scikit-learn pipeline is wrapped in a `ForecastingPipelineWrapper`; the `ForecastingPipelineWrapper` has some additional logic
needed to properly handle timeseries data depending on the algorithm used.

Once we have a pipeline, all that is left is to call `fit()` on it:

```python
def train_model(X, y, sample_weights):
    model_pipeline = build_model_pipeline()
    
    model = model_pipeline.fit(X, y)
    return model
```

The return value from `train_model()` is the model fitted on the input data.

## `script_run_notebook.ipynb`
`script_run_notebook.ipynb` serves as an easy way to run `script.py` on an AzureML compute cluster using a script run.

It is similar to the existing AutoML sample notebooks; however, there are a couple of key differences which are explained below.

#### Environment
Normally, when using AutoML, the training environment will be automatically set by the SDK. However, when running a script run, AutoML does not get a chance to
define the training environment and an environment must be defined for the script run to succeed.

Code generation reuses the environment that was used in the original experiment, if possible; this guarantees that the script run will not fail due to missing
dependencies and has the side benefit of preventing an image rebuild step, which saves time and compute resources.

If you make changes to `script.py` that require additional dependencies, or you would like to use your own environment, you will need to update the `Create environment`
cell in `script_run_notebook.ipynb` accordingly.

For additional documentation on AzureML environments, see [this page](https://docs.microsoft.com/en-us/python/api/azureml-core/azureml.core.environment.environment?view=azure-ml-py).

#### Submitting the experiment
Instead of creating an `AutoMLConfig` and then passing it to `experiment.submit()`, you need to create a [`ScriptRunConfig`](https://docs.microsoft.com/en-us/python/api/azureml-core/azureml.core.scriptrunconfig?view=azure-ml-py) and pass that instead.

```python
from azureml.core import ScriptRunConfig

src = ScriptRunConfig(source_directory=project_folder, 
                      script='script.py', 
                      compute_target=cpu_cluster, 
                      environment=myenv,
                      docker_runtime_config=docker_config)
 
run = experiment.submit(config=src)
```

All of the parameters in the above snippet are defined in previous cells in `script_run_notebook.ipynb`.

You may have additional needs for your script run if you modified `script.py` (for example, to add command line arguments). Those changes should be made in this cell.

#### Loading the fitted model
It is possible that the model will not serialize/deserialize correctly using `pickle.dump()` and `pickle.load()` due to pickle limitations (for example, lambda
functions cannot be serialized using pickle). Hence, `joblib.dump()` and `joblib.load()` are used by default instead.