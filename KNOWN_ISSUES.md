# Known issues/limitations
Listed below are the currently known issues and limitations of code generation.

* When running with a private preview SDK, an image build step is currently required before the experiment starts. This is handled for you, but will add to overall experiment runtime. The image build step will be removed in a future release when using a curated environment in production SDK instead of private indexes. This could be done even when using stable publicly/production Previews.  
* Currently only classification, regression, and forecasting tasks are supported.
* Streaming datasets are not supported.
* DNN trained models are not supported.
* `DataTransformer._engineered_feature_names_class` is currently loaded from a bytestring to map raw to engineered feature names, which may be problematic if the list of featurizers changes.