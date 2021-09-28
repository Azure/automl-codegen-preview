# Known issues/limitations
Listed below are the currently known issues and limitations of code generation.

* When running with a private preview SDK, an image build step is currently required before the experiment starts. This is handled for you, but will add to overall experiment runtime. The image build step will be removed in a future release when using a curated environment in production SDK instead of private indexes. This could be done even when using stable publicly/production Previews.  
* Currently only classification, regression, and forecasting tasks are supported.
* Currently DNN trained models are not supported.
* Streaming datasets are not supported.
* Depending on the SDK version used to train, if your featurization pipeline makes use of `CountVectorizer`, you may run into a pickling error when attempting to pickle the fitted model trained using codegen. You will know this is the case if `CountVectorizer` contains the parameter `tokenizer=lambda x: [x]`. To fix this, add the import `from azureml.automl.runtime.featurization.data_transformer import DataTransformer` and change the parameter to `tokenizer=DataTransformer._wrap_in_lst`.
