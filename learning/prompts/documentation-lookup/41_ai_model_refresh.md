Prompt:
How can I make sure my Apache Beam pipeline uses the latest version of the ML model?

Response:
Production ML workflows often involve updating the ML model with new data. You can ensure that your Apache Beam pipeline uses the latest version of the ML model by leveraging the RunInference API alongside Apache Beam's [side inputs](https://beam.apache.org/documentation/programming-guide/#side-inputs) feature. Side inputs are additional inputs that can be provided to a [`ParDo`](https://beam.apache.org/documentation/programming-guide/#pardo) transform, in addition to the main input [`PCollection`](https://beam.apache.org/documentation/basics/#pcollection).

The `RunInference` transform accepts an optional input parameter `model_metadata_pcoll`, which is a side input `PCollection` emitting `ModelMetadata`. `ModelMetadata` is a `NamedTuple` object containing `model_id` and `model_name`, used to load the model for inference and identify it in the metrics generated by the `RunInference` transform. The URL or path to the model should be compatible with the respective [`ModelHandler` requirements](https://beam.apache.org/documentation/ml/about-ml/#modify-a-python-pipeline-to-use-an-ml-model).

If the main collection emits inputs before the `model_metadata_pcoll` side input is available, the main `PCollection` will be buffered until the `model_metadata_pcoll` side input is emitted.

For more information on `ModelMetadata`, refer to the [Apache Beam Python SDK documentation](https://beam.apache.org/releases/pydoc/current/apache_beam.ml.inference.base.html#apache_beam.ml.inference.base.ModelMetadata).

A common approach to model updates in production is to use a `FileWatchPattern` as a side input:

```python
import apache_beam as beam
from apache_beam.ml.inference.utils import WatchFilePattern
from apache_beam.ml.inference.base import RunInference

tf_model_handler = ... # model handler for the model

with beam.Pipeline() as pipeline:

  file_pattern = '<path_to_model_file>'

  side_input_pcoll = (
    pipeline
    | "FilePatternUpdates" >> WatchFilePattern(file_pattern=file_pattern))

  main_input_pcoll = ... # main input PCollection

  inference_pcoll = (
    main_input_pcoll
    | "RunInference" >> RunInference(
    model_handler=model_handler,
    model_metadata_pcoll=side_input_pcoll))
```

In the provided example, the `model_metadata_pcoll` parameter expects a `PCollection` of `ModelMetadata` compatible with the `AsSingleton` marker. Given that the pipeline employs the `WatchFilePattern` class as a side input, it automatically manages windowing and encapsulates the output into `ModelMetadata`.

For more information, refer to the [Use `WatchFilePattern` to auto-update ML models in RunInference](https://beam.apache.org/documentation/ml/side-input-updates/) section in the Apache Beam documentation.
