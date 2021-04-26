---
layout: post
title: The implications of pickling ML models
feature_image: "/images/aysegul-yahsi-4rPoNLW_3rs-unsplash.jpg"
---

When you have trained a machine learning model (pipeline), you will make predictions directly afterwards to assess its quality. When using the model actually for something useful, we also want to make predictions with it at a later point in time. This forces us to store the model to disk and think of a way to serialize it.

If you have built your pipeline with Python, the most common and probably the easiest way is to store your model using the `pickle` module. In most cases, this is the least effort solution that will bring you the most cost-efficient solution if you are aware of the implications behind the mechanics of `pickle`.

In the case that the mechanics of `pickle` prohibit its stable usage or makes it too complicated, there are various alternatives that are either limited in their scope or require a fair amount of additional work. Depending on your use case, one of them might be better suited for you.

## What is pickling and how is it used in ML?

Pickle or pickling (as a verb) is a (binary) protocol to transform Python objects into a stream of bytes. This stream of bytes can then be later deserialised in another process, possibly on a different host. By being built-in into Python itself, you can serialise nearly all Python objects. Exceptions include things like lambda functions but some libraries (like `cloudpickle`) extend the pickling process to support also these.

The beauty of pickle is that you can serialise arbitrary complex Python object hierarchies without the need to write custom serialisation code nor having to modify the code of the serialised objects/classes.

In the case of machine learning pipelines, we can use them to store fitted models. As the machine learning pipelines themselves are also Python objects, pickling them is no different than pickling any other Python object. This can be seen in the scikit-learn example:

```python
import pickle

from sklearn import svm
from sklearn import datasets

clf = svm.SVC()
X, y = datasets.load_iris(return_X_y=True)
clf.fit(X, y)
with open('model.pkl', 'w') as f:
    pickle.dump(clf, f)
```

We can reload this fitted model later using the following snippet. Note that `pickle` also takes care of loading the correct Python modules in the background.

```python
import pickle

with open('model.pkl', 'r') as f:
    clf = pickle.load(f)

from sklearn import datasets
X, _ = datasets.load_iris(return_X_y=True)
y = clf2.predict(X[0:1])
print(y[0])
# Outputs: 0
```

## Alternatives to pickling

While pickling is a very convenient way to store fitted models, there are also negative sides to it. They may be a cause why you maybe don't want to use it.

There was a recent blogpost by [Ned Batchelder about pickles's nine flaws](https://nedbatchelder.com/blog/202006/pickles_nine_flaws.html). One of the issues is that `pickle` doesn't provide you stability over the changes third-party libraries make in their code. Pickle stores the attributes of each object that you pass to it and then restores the objects with these attributes again. Thus a pickle already turns unusable if you simply rename an instance attribute of one of your classes and try to load a pickle that was generated with the code of the old name. As pickle is missing the information that the variable was renamed, it doesn't fill this variable. See the following example:

**example.py:**
```python
class A:
    def __init__(self, a):
        self.a = a
```

```python
import example
import pickle
obj = example.A(2)
vars(obj)
# {'a': 2}
pickle.dumps(obj)
# b'\x80\x03cexample\nA\nq\x00)\x81q\x01}q\x02X\x01\x00\x00\x00aq\x03K\x02sb.'
```

A slight modification to `example.py` to make the single attribute private:

```python
class A:
    def __init__(self, a):
        self._a = a
```

Still leads to the unpickled version of the above pickle to an instance of `A` with an `a` but no `_a` attribute. Thus any method that would expect `_a` to be set will fail with an `AttributeError`.

```python
import pickle
a_object = pickle.loads(b'\x80\x03cexample\nA\nq\x00)\x81q\x01}q\x02X\x01\x00\x00\x00aq\x03K\x02sb.')
vars(a_object)
# {'a': 2}
```

Another aspect of pickling is that it is bound to the Python language as it aims to reconstruct Python objects. Thus when you want to serve your models in a completely different stack, you will not be able to use there to restore your fitted model.

Additionally, you should only use pickle as a persistence format if you trust all actors in your model development chain. From a security perspective, it is critical that you only load pickles that come from a trusted source. It is possible to craft pickles that [execute arbitrary code on load](https://checkoway.net/musings/pickle/).

## Selecting an example

To walk through the different approaches of serialising fitted machine learning models, we are going to pick an example of a simple ML pipeline. One caveat that we build into that pipeline is that we have scikit-learn, pandas and LightGBM as a dependency. This should increase the complexity of the object hierarchy sufficient to reflect real-world issues.

You can see the full example in my [nyc-taxi-fare-prediciton-deployment](https://github.com/xhochy/nyc-taxi-fare-prediction-deployment-example/blob/a37ec165e8f624ed8338067f7b315c663448ef50/src/nyc_taxi_fare/pipeline.py) repository. The basic pipline, accepting a DataFrame as an input, looks as follows (pseudo-Python code!):

```python
pipeline = Pipeline(
    CoulmnTransformer(
        (FunctionTransformer(), ["passenger_count"]),
        (FunctionTransformer(func=split_pickup_datetime), […],),
        (FunctionTransformer(func=haversine_distance_from_df), […]),
    ),
    LGBMRegressor(objective="regression_l1")
)
```

We train and save this pipleline using the typical `scikit-learn` interface. We use `pickle.HIGHEST_PROTOCOL` in the call to `dump` to let `pickle` use its most efficient form of serialization. This won't be human-readable at all but it will be a smaller dump and faster to load than the older protocol versions.

```python
df = pd.read_parquet(…)[relevant_coluns]
pipeline = lgbm_pipeline()
pipeline.fit(df, y)

with open(model_location, "wb") as f:
    pickle.dump(pipeline, f, protocol=pickle.HIGHEST_PROTOCOL)
```

To predict using the trained model, we can reload the stored pickle file and call `predict` on the restored object. As long as we have the dependencies installed in the right version, this is going to work. We don't even need to import anything, the `pickle` code is taking care of all of this.

```python
with open("model.pkl", "rb") as f:
    pipeline = pickle.load(f)
    
df = …
predictions = pipeline.predict(df)
```

In the example case here, we have trained a model on a single month of New York Taxi trip data for the prediction of fare prices based on time and distance. The resulting `model.pkl` file was 318K in size. Pickle files are written in an efficient binary format but aren't compressed at all. Using `zstd -19 model.pkl -o model.pkl.zst`, we can compress it down to 82K (took 0,18s).

## Alternative 1: Roll your own serialization protocol

Instead of relying on `pickle`, you could instead roll your own serialization protocol. This means that you would have to write serialization code for all the steps in your pipeline and store them as JSON, Protobuf or a similar generic serialization protocol. As long as your pipeline structure is simple enough, this will allow you to serialise your models without the need to explicitly track the versions of your packages. You still need to take care that future versions of your code load the old models correctly though. This is much more complicated than it sounds and in most cases, you will only realise this when you have written out several versions of your serialisation specification.

If you plan to go the route of writing your own serialization protocol, you can take insipration from [Camel](https://eev.ee/release/2015/10/15/dont-use-pickle-use-camel/), a library that allows you to do exactly that into YAML files in a safe fashion. For machine learning models you should though keep in mind that you will quite often save complex features matrices and thus a human-readable format like YAML is most likely to inefficient for that.

To illustrate how much work this is and which difficulties can occur, we are using the above example pipeline and implement serialisation for it using `camel`. For this, we need to implement loaders that transform the specific Python objects into standard YAML-serialisable objects.

We start the implementation by instantiating a `CamelRegistry` where we register all our custom loader and dumper functions. As a simple start, we implement a dumper/loader combination for `scikit-learn`'s `Pipeline`. This is the most straight-forward of all the parts of the overall pipelines. We only need to return the list of steps and pass that back into the constructor on load.

```python
from camel import CamelRegistry
from sklearn.pipeline import Pipeline

type_registry = CamelRegistry()

@type_registry.dumper(Pipeline, 'pipeline', version=1)
def dump_pipeline(pipeline):
    return pipeline.steps

@type_registry.loader('pipeline', version=1)
def load_pipeline(pipeline, version):
    return Pipeline(pipeline)
```

The next class we need to implement is the `ColumnTransformer`. While at first sight, we would have expected that we also only need the arguments that were passed into the original constructor, we actually need to serialise the full internal state. This is due to the fact that the transformer observes the data that passes through it during fit and checks against that during `predict`. While the code below doesn't look that lengthy, it actually doesn't provide much benefit over a `pickle`. It serialises the internal structure of the transformer and deserialises the exact same structure again during load. Thus this code implicitly introduces the constraint that we have the same `scikit-learn` version in both cases. Due to the limited public API of the `ColumnTransformer`, it is nearly impossible to get the state of the `ColumnTransformer` that doesn't make an assumption of its internal implementation.

Another point during its serialisation is that one of its attributes is a `pandas.Index` object. While we could invest in writing serialisation code that handles all kindes of instantiations of this class, we limit ourselves here to the assumption that it is a string-list based index, making the dump just a single line. This limitation is encoded in the version number.


```python
from pandas import Index
from sklearn.compose import ColumnTransformer

@type_registry.dumper(ColumnTransformer, 'column_transformer', version=1)
def dump_column_transformer(transformer):
    # We need to seriliaze the internal state of the ColumnTransformer as
    # otherwise it will complain that it wasn't yet fitted.
    attrs = {k: v for k, v in vars(transformer).items() if not k.startswith("__")}
    attrs['_feature_names_in'] = list(attrs['_feature_names_in'])
    return attrs

@type_registry.loader('column_transformer', version=1)
def load_column_transformer(transformer, version):
    ct = ColumnTransformer.__new__(ColumnTransformer)
    ct.__dict__.update(transformer)
    return ct

@type_registry.dumper(Index, 'index', version=1)
def dump_index(index):
    return list(index.values)

@type_registry.loader('index', version=1)
def load_index(index, version):
    return pd.Index(index)
```

The `FunctionTransfomer` in contrast provides again a sufficiently large public API so that we can write serialisation code without using internal implementation details. We limit ourselves to only using fully-qualified functions (i.e. no `lambda`) but that puts us on par with `pickle`. To improve on `pickle`, we would actually need to implement something that qualifies the version of the function on dump and dispatch to the correct (dated/versioned) implementation of that function during load.

```python
from sklearn.preprocessing import FunctionTransformer

import importlib

@type_registry.dumper(FunctionTransformer, 'function_transformer', version=1)
def dump_function_transformer(transformer):
    # Identity transform
    if transformer.func is None:
        return None, None
    return transformer.func.__module__, transformer.func.__name__

@type_registry.loader('function_transformer', version=1)
def load_function_transformer(transformer, version):
    if transformer[0] is None:
        return FunctionTransformer()
    mod = importlib.import_module(transformer[0])
    return FunctionTransformer(getattr(mod, transformer[1]))
```

Finally, we need to write serialisation code for the LightGBM model. Here we are faced with the complications again that we need to store the internal state of LightGBM's `sklearn` wrapper as well as need to write serialisation code for its `Booster` class. The latter is  more in the spirit of our pickle-free-and-version-stable approach we want to implement in this example. The `Booster` class provides a function `model_to_string` that exports its internal state as a string with a stable interpretation that can be loaded by different LightGBM versions. Sadly, as we did need to serialise the internal state of the `sklearn` wrapper, we are still bound to using the same version of the `lightgbm` package in the dump and load sides.

```python
from camel import Camel

from lightgbm.sklearn import LGBMRegressor
from lightgbm import Booster

from collections import defaultdict, OrderedDict

@type_registry.dumper(LGBMRegressor, 'lgbm_regressor', version=1)
def dump_lgbm_regressor(lgbm_regressor):
    dct = lgbm_regressor.__dict__.copy()
    # Restore later with defaultdict(collections.OrderedDict)
    del dct['_best_score']
    return dct

@type_registry.loader('lgbm_regressor', version=1)
def load_lgbm_regressor(lgbm_regressor, version=1):
    regressor = LGBMRegressor.__new__(LGBMRegressor)
    regressor.__dict__ = lgbm_regressor.copy()
    regressor.__dict__['_best_score'] = defaultdict(OrderedDict)
    return regressor

@type_registry.dumper(Booster, 'lgbm_booster', version=1)
def dump_lgbm_booster(lgbm_booster):
    return lgbm_booster.model_to_string()

@type_registry.loader('lgbm_booster', version=1)
def load_lgbm_booster(lgbm_booster, version):
    return Booster(model_str=lgbm_booster)
```

Once all we have defined all dumper/loader combinations, we pass the registry in a `Camel` object and can serialise our pipeline into Yaml and load it back again. Predicting using that restored pipeline produces then the exact same predictions as the original pipeline. The uncompressed `model.yaml` file is 348K in size, being in line with the `model.pkl`. Compressing it with `zstd` also shrinks it down to 84K.

```python
c = Camel([type_registry])
model_yaml = c.dump(pipeline)
restored_pipeline = c.load(model_yaml)
preds = restored_pipeline.predict(df)
```

In conclusion, using `camel` as a way to serialise the pipeline looked promising at first sight as it allowed you to easily define simple and versioned Python functions for serialisation. Sadly, in a lot of cases it was not possible to write serialisation code that is independent of the used library versions. The public API provided by `scikit-learn` as well as `LightGBM` didn't cover enough of the internals that you could write version-independent code. 

One approach that you could use would be to write library-version specific serialisation code for these pipeline components but implement the predict part of them fully without them. Dependending on the exposure of the internal state of your pipeline steps, this is either simpler or harder than the approach taken here. In _Alternative 3_, we are actually doing this approach by taking a `scikit-learn` based pipeline and do the prediction via ONNX with no dependency on `scikit-learn` or our custom code.

## Alternative 2: Tensorflow / Keras

Tensorflow and Keras are two popular machine learning libraries that work together and thus also share their approach to saving models. Both provide [their own](https://keras.io/getting_started/faq/#what-are-my-options-for-saving-models) [help sections](https://www.tensorflow.org/tutorials/keras/save_and_load#save_the_entire_model) on what are the possible options. 

Historically, they were using HDF5 files to store the configuration and the weights of the neural networks but were limited to the built-in types. If you had custom layers or subclassed any components, you needed to do an extra step to [persist your custom objects](https://www.tensorflow.org/tutorials/keras/save_and_load#saving_custom_objects).

Nowadays Tensorflow provides its custom serialization protocol named [`SavedModel`](https://www.tensorflow.org/guide/saved_model). This allows one to saved arbitrary complex and custom Tensorflow execution graphs. While this supports the flexibility of customisation in contrast to the previous approach, it is interesting to note that it suffers a similar problem with the recreation as `pickle` does. As outlined in the section ["Saving a custom model"](https://www.tensorflow.org/guide/saved_model#saving_a_custom_model), no Python code or even attributes are stored. Instead `SavedModel` relies on the cache of traced `tensorflow.function` calls. In contrast to `pickle` this is though a bit better as this allows much more complex excution graphs to be restored without having the full Python code that was used for training available.

*As I personally have not been using Tensorflow much and the documentation around BoostedTrees in Tensorflow was quite sparse, I did skip applying this to my example pipeline. If a reader of this article though has the sufficient knowledge to replicate it in Tensorflow, I would be really grateful for a code snippet.*

## Alternative 3: Framework-neutral model specifications PMML / ONNX

With `pickle` and a custom serialisation protocol like the `Camel` approach, we were still bound to Python as a language. In the Tensorflow setting, we aren't bound to Python anymore but to anything that Tensorflow supports. Still, we have the constraint that our model must be a Tensorflow execution graph, thus we are now bound to a specific machine learning framework.

There are several alternatives available that promise to export a machine learning model / pipeline to a framework neutral format. The most prominent ones are [ONNX](https://onnx.ai/) and [PMML](http://dmg.org/). We are going to take ONNX as an example for this category here.

ONNX is standard / exchange format that defines a set of operators to represent the output of a model fit with the scope to run the prediction step using a different technology and possibly on a different hardware. While it has "neural network (NN)" in its name, it also supports other machine learning models like boosted trees. With the given set of operators, you can also reconstruct nearly all elements of a machine learning pipeline (for inference). In the case that there is not a predefined component / translator for a step of your pipeline, you should be able to build your own.

To show how this works, we can take the example pipeline from above and run our feature engineering using the standard `sklearn` code and only convert the boosted tree to ONNX. This is a lot simpler than the whole example pipeline as for that part of the pipeline, we already have pre-existing converters.

```python
used_columns = [
    "fare_amount",
    "pickup_latitude",
    "pickup_longitude",
    "dropoff_latitude",
    "dropoff_longitude",
    "tpep_pickup_datetime",
    "passenger_count",
]
df = pd.read_parquet("data/yellow_tripdata_2016-01.parquet", columns=used_columns)
y = df.pop("fare_amount")

# feature_enginering is the example pipeline without the Regressor
features_as_array = feature_enginering().fit_transform(df)

# Take a sklearn built-in regressor as we have a converter
# for that directly in skl2onnx
from sklearn.ensemble import GradientBoostingRegressor
from skl2onnx.common.data_types import FloatTensorType

gbr = GradientBoostingRegressor(loss="lad")
gbr.fit(features_as_array)

# Convert to ONNX
from skl2onnx import convert_sklearn

initial_type = [('float_input', FloatTensorType([None, 5]))]
onnx = convert_sklearn(gbr, initial_types=initial_type)
(model_dir / "gbr.onnx").write_bytes(onnx.SerializeToString())

# Load the ONNX model to compute the predictions with onnxruntime
import onnxruntime as rt
import numpy
sess = rt.InferenceSession(str(model_dir / "gbr.onnx"))
input_name = sess.get_inputs()[0].name
label_name = sess.get_outputs()[0].name

pred_onnx = sess.run([label_name], {input_name: features_as_array.astype(np.float32)})[0]
```

In the above code sample, we train the model using the normal `sklearn` API but then use the `skl2onnx` to convert the model to ONNX as a representation. In the following line we then use the `onnxruntime` package to load that model again. The `onnxruntime` does only pull the information about the pipeline from the `gbr.onnx` file and doesn't reference the `sklearn` pipeline at all. 

Note that as `onnxruntime` doesn't support `pandas.DataFrame` as an input, we fallback here to `numpy.ndarray`. With a bit more knowledge of ONNX, you should be able though to pass in DataFrames as the specification support heterogenously typed inputs.

With the `onnxruntime` itself being independent of Python, we should be able to get rid of some of the overhead that is introduced by using Python as a high-level glue and a bit by having in some steps intermediate results in a NumPy array in main memory where it could also be passed via a CPU register.

For that we can run benchmarks on a whole month of NYC taxi trip data as well as on a single row of data.

```python
# Extract inputs outside of the benchmark loop
features_as_array_32 = features_as_array.astype(numpy.float32)
first_line_32 = features_as_array_32[0, :].reshape(1, -1)
first_line = features_as_array[0, :].reshape(1, -1)

# ONNX batch predictions
%timeit sess.run([label_name], {input_name: features_as_array_32})
# 3.42 s ± 153 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)

# scikit-learn batch predictions
%timeit gbr.predict(features_as_array.astype(numpy.float32))
# 10.9 s ± 41 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)

# ONNX single row prediction
%timeit sess.run([label_name], {input_name: first_line_32})
# 14.4 µs ± 466 ns per loop (mean ± std. dev. of 7 runs, 10000 loops each)

# scikit-learn single row prediction
%timeit gbr.predict(first_line)
# 113 µs ± 2.54 µs per loop (mean ± std. dev. of 7 runs, 10000 loops each)
```

We can see a ~3x speedup in the batch prediction case. This is partly due to ONNX storing less results in main memory but also as it is using all four cores of the CPU while `sklearn` is only using a single one.

In the single row case, we see a speedup of 100µs. Here we look at the absolute time as this is the latency offset between using the Python based code and the native ONNX execution. Python-based libraries like NumPy or `scikit-learn` add a bit of overhead through their  checks and Python calls that are normally irrelevant in the case of batch predictions but can be seen in the latency-heavy single-row case. `onnxruntime` does get rid of them here.

With this reduced example running, we can now take the example pipeline and convert it to ONNX. One important caveat is that ONNX normally expects a matrix as an input. There are Python-based runtimes like [`mlprodict`](http://www.xavierdupre.fr/app/mlprodict/helpsphinx/index.html) that support ONNX with `pandas.DataFrame` as an input but we wanted to keep it as portable as possible and thus converted the pipeline to work with purely NumPy arrays as an input.

In the following, we will show only the result of the conversion. In future, there might be a blog post though about how this pipeline was converted to this form and what roadblocks were hit during that.

The first step is to rewrite the feature engineering based on `numpy` matrices:

```python

df_for_matrix = df[
    [
        "passenger_count",
        "tpep_pickup_datetime",
        "pickup_latitude",
        "pickup_longitude",
        "dropoff_latitude",
        "dropoff_longitude",
    ]
]
df_for_matrix["tpep_pickup_datetime"] = df_for_matrix[
    "tpep_pickup_datetime"
].astype(int)

# Get a mapping of column name -> index
df_for_matrix_cols = {x: i for i, x in enumerate(df_for_matrix.columns)}
df_for_matrix_cols

def haversine_from_matrix(x):
    return haversine_distance(x[:, 0], x[:, 1], x[:, 2], x[:, 3]).reshape(
        x.shape[0], -1
    )


def split_pickup_datetime(x):
    # Day of week, hour, minute
    return np.stack(
        [
            (x[:, 0] // (24 * 60 * 60 * 1000 * 1000 * 1000) + 3) % 7,
            (x[:, 0] // (60 * 60 * 1000 * 1000 * 1000)) % 24,
            (x[:, 0] // (60 * 1000 * 1000 * 1000)) % 60,
        ],
        axis=1,
    )

pipeline_with_haversine = make_pipeline(
    ColumnTransformer(
        [
            (
                "haversine_distance",
                FunctionTransformer(haversine_from_matrix),
                [
                    df_for_haversine_cols["pickup_latitude"],
                    df_for_haversine_cols["pickup_longitude"],
                    df_for_haversine_cols["dropoff_latitude"],
                    df_for_haversine_cols["dropoff_longitude"],
                ],
            ),
            (
                "split_pickup_datetime",
                FunctionTransformer(split_pickup_datetime),
                [df_for_haversine_cols["tpep_pickup_datetime"]],
            ),
        ],
        remainder="passthrough",
    ),
    LGBMRegressor(objective="regression_l1"),
)
```

We then fit the pipeline as usual using the `sklearn` API:

```python
x_for_matrix = df_for_matrix.astype(float).values
pipeline.fit(x_for_matrix, y)
```

Converting it to ONNX fails though as there no converters registered for our custom transformations:

```python
initial_type = [("float_input", FloatTensorType([None, 8]))]
onnx_func_id = convert_sklearn(pipeline_with_haversine, initial_types=initial_type)

# This raises:
# RuntimeError: FunctionTransformer is not supported unless the transform function is None (= identity).
```

To register a conversion, we need to provide two functions. One fast and simple that computes the output shapes and one that does the full conversion. As the simpler one we start with the shape calculation:

```python
def calculate_custom_function_transformer_output_shapes(operator):
    if operator.raw_operator.func is haversine_from_matrix:
        N = operator.inputs[0].type.shape[0]
        operator.outputs[0].type = copy.deepcopy(operator.inputs[0].type)
        # The haversine functions returns a single new column
        operator.outputs[0].type.shape = [N, 1]
    elif operator.raw_operator.func is split_pickup_datetime:
        N = operator.inputs[0].type.shape[0]
        operator.outputs[0].type = copy.deepcopy(operator.inputs[0].type)
        # The datetime function returns (dayofweek, hour, minute)
        operator.outputs[0].type.shape = [N, 3]
    else:
        # Otherwise fallback to original FunctionTransformer converter
        calculate_sklearn_function_transformer_output_shapes(operator)
```

As the next step, we write the function that returns the ONNX representation of the transformers. The code is here is quite verbose as we need to describe the haversine distance function with ONNX operators. These operators are all represented by their respective classes and thus need more characters than their Python operator equivalents.

```python
def convert_custom_function_transformer(scope, operator, container):
    op = operator.raw_operator
    if op.func is haversine_from_matrix:

        op_in = operator.inputs[0]
        op_out = operator.outputs[0].full_name

        # Map to radians
        double_in = OnnxCast(op_in, to=onnx_proto.TensorProto.DOUBLE)
        as_radians = OnnxMul(double_in, np.pi / 180)
        # Select local variables
        lat1 = OnnxArrayFeatureExtractor(as_radians, 0)
        lng1 = OnnxArrayFeatureExtractor(as_radians, 1)
        lat2 = OnnxArrayFeatureExtractor(as_radians, 2)
        lng2 = OnnxArrayFeatureExtractor(as_radians, 3)
        # d_1 = np.sin(lng2 / 2 - lng1 / 2) ** 2
        d_1 = OnnxPow(OnnxSub(OnnxDiv(lng2, 2.0), OnnxDiv(lng1, 2.0)), 2)
        # d = (
        # np.sin(lat2 / 2 - lat1 / 2) ** 2
        #  + np.cos(lat1) * np.cos(lat2) * d_1
        # )
        d = OnnxAdd(
            OnnxPow(OnnxSin(OnnxSub(OnnxDiv(lat2, 2.0), OnnxDiv(lat1, 2.0))), 2),
            OnnxMul(
                OnnxCast(
                    OnnxMul(
                        OnnxCos(OnnxCast(lat1, to=onnx_proto.TensorProto.FLOAT)),
                        OnnxCos(OnnxCast(lat2, to=onnx_proto.TensorProto.FLOAT)),
                        output_names="lat1_lat2_mul",
                    ),
                    to=onnx_proto.TensorProto.DOUBLE,
                ),
                d_1,
                output_names="lat_d1_mul",
            ),
        )
        # return 2 * 6371 * np.arcsin(np.sqrt(d))  # 6,371 km is the earth radius
        r = OnnxMul(
            2 * 6371.0,
            OnnxCast(
                OnnxAsin(OnnxCast(OnnxSqrt(d), to=onnx_proto.TensorProto.FLOAT)),
                to=onnx_proto.TensorProto.DOUBLE,
            ),
            output_names=op_out,
        )
        result = OnnxCast(r, to=onnx_proto.TensorProto.FLOAT)
        result.add_to(scope, container)
    elif op.func is split_pickup_datetime:
        op_in = operator.inputs[0]
        op_out = operator.outputs[0].full_name
        op_in64 = OnnxArrayFeatureExtractor(
            OnnxCast(op_in, to=onnx_proto.TensorProto.INT64), 0
        )
        dayofweek = OnnxMod(
            OnnxAdd(OnnxDiv(op_in64, 24 * 60 * 60 * 1000 * 1000 * 1000), 3), 7
        )
        hour = OnnxMod(OnnxDiv(op_in64, 60 * 60 * 1000 * 1000 * 1000), 24)
        minute = OnnxMod(OnnxDiv(op_in64, 60 * 1000 * 1000 * 1000), 60)
        r = OnnxConcat(dayofweek, hour, minute, axis=1)
        result = OnnxCast(r, to=onnx_proto.TensorProto.FLOAT, output_names=op_out)
        result.add_to(scope, container)
    else:
        return convert_sklearn_function_transformer(scope, operator, container)
```

One drawback by the above implementation and the used `onnxruntime` was that not all operators supported `float` and `double` as input data types. Thus we needed to cast the intermediate results in some cases to the other precision.

Finally, we register our conversion functions with `skl2onnx` so that it gets picked up during conversion. We use the same conversion code as in the simpler example above.

```python
skl2onnx.update_registered_converter(
    FunctionTransformer,
    "FunctionTransformer",
    calculate_custom_function_transformer_output_shapes,
    convert_custom_function_transformer,
    True,
)


initial_type = [("float_input", FloatTensorType([None, 6]))]
onnx_with_features = convert_sklearn(pipeline, initial_types=initial_type)
(model_dir / "with_features.onnx").write_bytes(onnx_with_features.SerializeToString())

sess_with_features = rt.InferenceSession(str(model_dir / "with_features.onnx"))
input_name = sess_with_features.get_inputs()[0].name
label_name = sess_with_features.get_outputs()[0].name
```

With ONNX we have ensured that we can export the trained model without having any further dependencies. Additionally, through the ONNX specification we also have the model in a format where we expect it to be runnable in a reproducible fashion for a long time.

Finally, we will have also a look at the performance between the converted and the original pipeline.

```python
%timeit lgbm_pipeline.predict(x_for_matrix)
# 15.6 s ± 110 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)

%timeit sess_lgbm.run([label_name], {input_name: x_for_matrix.astype(np.float32)})
# 9.92 s ± 321 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)

first_line_32 = x_for_matrix.astype(np.float32)[0, :].reshape(1, -1)
first_line = x_for_matrix[0, :].reshape(1, -1)

%timeit lgbm_pipeline.predict(first_line)
# 777 µs ± 26.1 µs per loop (mean ± std. dev. of 7 runs, 1000 loops each)

%timeit sess_lgbm.run([label_name], {input_name: first_line_32})
# 73 µs ± 1.81 µs per loop (mean ± std. dev. of 7 runs, 10000 loops each)
```

In the batch case where we pass a full month of data, we see a 36% speedup. This is already remarkable given that LightGBM is also quite optimized and uses multiple CPUs for its prediction computation. For the single row case though, we see a 10x speedup as in the example above.

## Conclusion

Overall, we see that `pickle` is awesome as it stores arbitrary Python object hierachies. You can use this as your default for storing models. But be careful to use the exact same package versions when restoring the pickle.

In the case that you are a Tensorflow user and can represent your whole pipeline with it, you should use its `SavedModel` approach. This will not introduce a new technology into your stack but it will remove the dependency on your custom code during the deployment.

If you want to share your model with someone else, transfer it to a different technology or simply want to be sure that updating a single dependency in your stack doesn't lead to silently different predictions, you should convert it to a format that is totally independent of your libraries and simply a representation of the transformation of the machine learning model. 

In the last case, you should use ONNX or PMML if they suffice for your use case. They support a wide range of operations and there are a lot runtimes available for them. If they suffice for your problem, they are a near perfect solution. Otherwise, if you cannot / don't want to use pickle at all cost, roll your own serialisation. Still, you need to be aware that this will get complicated and expensive as you will need individual serialisers for all your classes and also handle version migrations when your code changes.

## Improving the Stats Quo

It would be easy but also a bit boring to just report on the status quo. Thus during the writing of the blog post, I made the following pull requests to improve the status quo a bit:

* [Add `camel` to conda-forge](https://github.com/conda-forge/staged-recipes/pull/14239)
* [Add `skl2onnx` to conda-forge](https://github.com/conda-forge/staged-recipes/pull/14220)
* [Add `mlprodict` to conda-forge](https://github.com/conda-forge/staged-recipes/pull/14249)
* [Update `onnxconverter-common` to 1.7.0 on conda-forge](https://github.com/conda-forge/onnxconverter-common-feedstock/pull/1)
* [Retroactively add `onnxconverter-common` 1.6.1 on conda-forge](https://github.com/conda-forge/onnxconverter-common-feedstock/pull/2)
* [Retroactively add `onnxconverter-common` 1.6.0 on conda-forge](https://github.com/conda-forge/onnxconverter-common-feedstock/pull/3)
* [Retroactively add `onnxconverter-common` 1.5.5 on conda-forge](https://github.com/conda-forge/onnxconverter-common-feedstock/pull/4)
* [Update `keras2onnx` to 1.6.1 on conda-forge](https://github.com/conda-forge/keras2onnx-feedstock/pull/1)
* [Update `keras2onnx` to 1.6.5 on conda-forge](https://github.com/conda-forge/keras2onnx-feedstock/pull/2)

We also closed one issue and opened two new ones:

* [https://github.com/microsoft/hummingbird/issues/314](https://github.com/microsoft/hummingbird/issues/314)
* [https://github.com/onnx/sklearn-onnx/issues/630](https://github.com/onnx/sklearn-onnx/issues/630)
* [https://github.com/onnx/onnxmltools/issues/453](https://github.com/onnx/onnxmltools/issues/453)


*Title picture: Photo by [Aysegul Yahsi](https://unsplash.com/@aysegulyahsi?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)*

