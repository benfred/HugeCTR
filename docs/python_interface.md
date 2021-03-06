# About HugeCTR Python Interface
 
HugeCTR Python Interface includes low-level training API and inference API. Currently, both APIs rely on the configuration file in JSON format and we will introduce our high-level JSON-free training API in the next release. Please refer to [Configuration File Setup](./configuration_file_setup.md) if you want to get detailed information on how to configure the JSON file. 

## Table of Contents
* [Low-level Training API ](#low-level-training-api)
* [Inference API](#inference-api)
* [Sample Code](#sample-code)

## Low-level Training API ##
For HugeCTR low-level training API, the core data structures are `SolverParser`, `LearningRateScheduler`, `DataReader`, `ModelOversubscriber` and `Session`. HugeCTR currently supports both epoch mode training and non-epoch mode training for dataset in Norm and Raw formats, and only supports non-epoch mode training for dataset in Parquet format. While introducing the API usage, we will elaborate how to employ these two modes of training.
 
### SolverParser ###
**solver_parser_helper method**
```bash
hugectr.solver_parser_helper()
```
`solver_parser_helper` returns an `SolverParser` object according to the custom argument values，which specify the training resource and task items.

**Arguments**
* `seed`: A random seed to be specified. The default value is 0.

* `max_eval_batches`: Maximum number of batches used in evaluation. It is recommended that the number is equal to or bigger than the actual number of bathces in the evaluation dataset. The default value is 100.

* `batchsize_eval`: Minibatch size used in evaluation. The default value is 2048.

* `batchsize`: Minibatch size used in training. The default value is 2048.

* `model_file`: Trained dense model file to be loaded. If you train a model from scratch, it is not necessary.

* `dense_opt_states_file`: Dense otimizer states file to be loaded. If your optimizer doesn't have any dynamic states to be saved, an empty file is created. Make sure that you use the same type of optimizer used to generate this file. If you train a model from scratch or you don't want to use it, it is unnecessary.

* `embedding_files`: A trained embeding table (or sparse model) or their list to be loaded. If you train a model from scratch, it is not necessary.

* `sparse_opt_states_file`: Sparse otimizer states file(s) to be loaded. The behavior is as the same as `dense_opt_states_file`.

* `vvgpu`: GPU indices used in the training process, which has two levels. For example: [[0,1],[1,2]] indicates that two nodes are used. In the first node, GPUs 0 and 1 are used while GPUs 1 and 2 are used for the second node. It is also possible to specify non-continuous GPU indices such as [0, 2, 4, 7]. The default value is [[0]].

* `use_mixed_precision`: Whether to enable mixed precision training. The default value is `False`.

* `enable_tf32_compute`: If you want to accelerate FP32 matrix multiplications within the FullyConnectedLayer and InteractionLayer, set this value to `True`. The default value is `False`.

* `scaler`: The scaler to be used when mixed precision training is enabled. Only 128, 256, 512, and 1024 scalers are supported for mixed precision training. The default value is 1.0, which corresponds to no mixed precision training.

* `i64_input_key`: If your dataset format is `Norm`, you can choose the data type of each input key. For the `Parquet` format dataset generated by NVTabular, only I64 is allowed. For the `Raw` dataset format, only I32 is allowed. Set this value to `True` when you need to use I64 input key. The default value is `False`.

* `use_algorithm_search`: Whether to use algorithm search for cublasGemmEx within the FullyConnectedLayer. The default value is `True`.

* `use_cuda_graph`: Whether to enable cuda graph for dense network forward and backward propagation. The default value is `True`.

* `repeat_dataset`: Whether to repeat the dataset for training. If the value is `True`, non-epoch mode training will be employed. Otherwise, epoch mode training will be adopted. The default value is `True`.

### LearningRateScheduler ###
**get_learning_rate_scheduler method**
```bash
hugectr.get_learning_rate_scheduler()
```
`get_learning_rate_scheduler` generates and returns a LearningRateScheduler object based on the configuration JSON file. When the `SGD` optimizer is adopted for training, the returned object can obtain the dynamically changing learning rate according to the `warmup_steps`, `decay_start`和`decay_steps` configured in the JSON file。Please refer to [SGD Optimizer and Learning Rate Scheduling
](./hugectr_user_guide.md#sgd-optimizer-and-learning-rate-scheduling) if you want to get detailed information about LearningRateScheduler.

**Arguments**
* `configure_file`: The JOSN format configuration file.
***
**get_next method**
```bash
hugectr.LearningRateScheduler.get_next()
```
This method takes no extra arguments and returns the learning rate to be used for the next iteration.

### DataReader ###
**set_source method**
```bash
hugectr.DataReader32.set_source()
hugectr.DataReader64.set_source()
```
The `set_source` method of DataReader currently supports the dataset in Norm and Raw formats, and should be used in epoch mode training. When the data reader reaches the end of file for the current training data or evaluation data, this method can be used to re-specify the training data file or evaluation data file.

**Arguments**
* `file_name`: The file name of the new training source or evaluation source. For Norm format dataset, it takes the form of `file_list.txt`. For Raw format dataset, it appears as `data.bin`. The default value is `''`, which means that the data reader will reset to the beginning of the current data file.

### ModelOversubscriber ###
**update method**
```bash
hugectr.ModelOversubscriber.update()
```
The `update` method of ModelOversubscriber currently supports Norm format datasets. Using this method requires that a series of file lists and the corresponding keyset files are generated at the same time when preprocessing the original data to Norm format. This method gives you the ability to load a subset of an embedding table into the GPU in a coarse grained, on-demand manner during the training stage. Please refer to [Model Oversubscription](./hugectr_user_guide.md#model-oversubscription) if you want to get detailed information about ModelOversubscriber.

**Arguments**
* `keyset_file` or `keyset_file_list`: This method is an overloaded method that can accept str or List[str] as an argument. For the model with multiple embedding tables, if the keyset of each embedding table is not separated when generating the keyset files, then pass in the `keyset_file`. If the keyset of each embedding table has been separated when generating keyset files, you need to pass in the `keyset_file_list`, the size of which should equal to the number of embedding tables.

### Session ###
**Session class**
```bash
hugectr.Session()
```
`Session` groups embeddings and dense network into an object with traning features. The construction of `Session` requires a configuration JSON file to parse the `optimizer` and `layers` clauses. Please refer to [Optimizer](./configuration_file_setup.md#optimizer) and [Layers](./configuration_file_setup.md#layers) to know the correct usage of the configuration file.

**Arguments**
* `solver_config`: A hugectr.SolverParser object, the solver configuration of the session.

* `config_file`: String, the configuration file in JSON format.

* `use_model_oversubscriber`: Boolean, whether to employ the features of ModelOversubscriber. The default value is `False`.

* `temp_embedding_dir:`: String，where to store the temporary embedding table files. The path needs to have write permission to support the features of ModelOversubscriber. The default value is `''`.
***

**get_model_oversubscriber method**
```bash
hugectr.Session.get_model_oversubscriber()
```
This method takes no extra arguments and returns the ModelOversubscriber object.
***

**get_data_reader_train method**
```bash
hugectr.Session.get_data_reader_train()
```
This method takes no extra arguments and returns the DataReader object that reads the training data.
***

**get_data_reader_eval method**
```bash
hugectr.Session.get_data_reader_eval()
```
This method takes no extra arguments and returns the DataReader object that reads the evaluation data.
***

**start_data_reading method**
```bash
hugectr.Session.start_data_reading()
```
This method takes no extra arguments and should be used if and only if it is under the non-epoch mode training. The method starts the `train_data_reader` and `eval_data_reader` before entering the training loop.
***

**set_learning_rate method**
```bash
hugectr.Session.set_learning_rate()
```
This method is used together with the `get_next` method of `LearningRateScheduler` and sets the learning rate for the next training iteration.

**Arguments**
* `lr`: Float, the learning rate to be set。
***

**train method**
```bash
hugectr.Session.train()
```
This method takes no extra arguments and executes one iteration of the model weights based on one minibatch of training data.
***

**get_current_loss method**
```bash
hugectr.Session.get_current_loss()
```
This method takes no extra arguments and returns the loss value for the current iteration.
***

**check_overflow method**
```bash
hugectr.Session.check_overflow()
```
This method takes no extra arguments and checks whether any embedding has encountered overflow.
***

**copy_weights_for_evaluation method**
```bash
hugectr.Session.copy_weights_for_evaluation()
```
This method takes no extra arguments and copies the weights of the dense network from training layers to evaluation layers.
***

**eval method**
```bash
hugectr.Session.eval()
```
This method takes no arguments and calculates the evaluation metrics based on one minibatch of evaluation data.
***

**get_eval_metrics method**
```bash
hugectr.Session.get_eval_metrics()
```
This method takes no extra arguments and returns the average evaluation metrics of several minibatches of evaluation data.
***

**evaluation method**
```bash
hugectr.Session.evaluation()
```
This method returns the average evaluation metrics of several minibatches of evaluation data. You can export predictions and labels to files by passing arguments into this method.

**Arguments**
* `export_predictions_out_file`: Optinal. If passed, the evaluation prediction results will be writen to the file specified by this argument. The order of the prediction results are the same as that of the labels, but may be different with the order of the samples in the dataset.

* `export_labels_out_file:`: Optinal. If passed, the evaluation labels will be writen to the file specified by this argument. The order of the labels are the same as that of the prediction results, but may be different with the order of the samples in the dataset.

## Inference API ##
For HugeCTR inference API, the core data structures are `ParameterServer`, `EmbeddingCache` and `InferenceSession`. Please refer to [Inference Framework](https://gitlab-master.nvidia.com/dl/hugectr/hugectr_inference_backend/-/blob/main/docs/user_guide.md#inference-framework) to get informed of the hierarchy of HugeCTR inference implementation.

Please **NOTE** that Inference API requires a configuration JSON file which is slightly different from the training JSON file. We need `inference` and `layers` clauses in the inference JSON file. The paths of the stored dense model and sparse model(s) should be specified at `dense_model_file` and `sparse_model_file` within the `inference` clause. Some modifications need to be made to `data` within the `layers` clause and the last layer should be replaced by `SigmoidLayer`. Please refer to [HugeCTR Inference Notebook](../notebooks/hugectr_inference.ipynb) for detailed information of the inference JSON file.

### ParameterServer ###
**CreateParameterServer method**
```bash
hugectr.inference.CreateParameterServer()
```
`ParameterServer` resides on the CPU and stores all the embedding tables of multiple models. This factory method `CreateParameterServer` creates and returns a `ParameterServer` object.

**Arguments**
* `model_config_path`: List[str], the list of configuration JSON files for different models.

* `model_name`: List[str], the list of different model names.
 
* `i64_input_key`: Boolean, whether to use I64 input key for the parameter server.

Please **NOTE** that the order of the configuration files within `model_config_path` and that of the model names within `model_name` should be consistent.

### EmbeddingCache ###
**CreateEmbeddingCache method**
```bash
hugectr.inference.CreateEmbeddingCache()
```
`EmbeddingCache` resides on the GPU and stores a portion of embeddings for a specific model. This factory method `CreateEmbeddingCache` creates and returns an `EmbeddingCache` object.

**Arguments**
* `parameter_server`: A hugectr.inference.ParameterServerBase object, the parameter server that accommodates the embedding tables to be cached on GPU.

* `cuda_dev_id`: Integer, the GPU device index.

* `use_gpu_embedding_cache`: Boolean, whether to employ the features of GPU embedding cache. If the value is `True`, the embedding vector look up will go to GPU embedding cache. Otherwise, it will reach out to the CPU parameter server directly.

* `cache_size_percentage`: Float, the percentage of cached embeddings on GPU relative to all the embedding tables on CPU.

* `model_config_path`: String, the configuration file of the model whose embeddings will be on the GPU embedding cache.

* `model_name`: String, the name of the model whose embeddings will be on the GPU embedding cache.

* `i64_input_key`: Boolean, whether to use I64 input key for the embedding cache.

Please **NOTE** that the `parameter_server` specified here should contain the model whose embedding tables are supposed to be cached on GPU. Besides, `i64_input_key` should be consistent with that of the `parameter_server`.

### InferenceSession ###
**InferenceSession class**
```bash
hugectr.inference.InferenceSession()
```
`InferenceSession` groups embedding cache and dense network into an object with inference features. The construction of `InferenceSession` requires an inference configuration JSON file. 

**Arguments**
* `config_file`: String, the inference configuration file.

* `device_id`: Integer, GPU device index.

* `embedding_cache`: A hugectr.inference.EmbeddingCacheInterface object, the embedding cache that loads a portion of embeddings on GPU for the model to be used.

Please **NOTE** that the `device_id` specified here should be the same as that of the `embedding_cache`.
***

**predict method**
```bash
hugectr.inference.InferenceSession.predict()
```
The `predict` method of InferenceSession currently supports model with only one embedding table. This method makes predictions for the samples in the inference inputs. Please refer to [HugeCTR Inference Notebook](../notebooks/hugectr_inference.ipynb) for detailed usage information.

**Arguments**
* `dense_feature`: List[float], the dense features of the samples.

* `embeddingcolumns`: List[int], the embedding keys of the samples.

* `row_ptrs`: List[int], the row pointers that indicate which embedding keys belong to the same slot.

* `i64_input_key`: Boolean, whether to use I64 input key for the inference session.
***
Taking Deep and Cross Model on Criteo dataset for example, if the inference request includes two samples, then `dense_feature` will be of the length 2\*13, `embeddingcolumns` will be of the length 2\*26, and `row_ptrs` will be like [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52].

## Sample Code ##
The sample code for non-epoch mode training, epoch mode training and inference will be given here. Please make sure the JSON files and the datasets are ready to successfully run the sample code. Please refer to [HugeCTR Python Interface Notebook](../notebooks/python_interface.ipynb) and [HugeCTR Inference Notebook](../notebooks/hugectr_inference.ipynb) to get familiar with the workflow of HugeCTR training and inference.

### Non-epoch Mode Training ###
```bash
from hugectr import Session, solver_parser_helper,get_learning_rate_scheduler
from mpi4py import MPI
json_file = "your_config.json"
solver_config = solver_parser_helper(seed = 0,
                                    batchsize = 16384,
                                    batchsize_eval = 16384,
                                    model_file = "",
                                    embedding_files = [],
                                    vvgpu = [[0,1,2,3,4,5,6,7]],
                                    use_mixed_precision = True,
                                    scaler = 1024,
                                    i64_input_key = False,
                                    use_algorithm_search = True,
                                    use_cuda_graph = True,
                                    repeat_dataset = True)
lr_sch = get_learning_rate_scheduler(json_file)
sess = Session(solver_config, json_file)
sess.start_data_reading()
for i in range(10000):
    lr = lr_sch.get_next()
    sess.set_learning_rate(lr)
    sess.train()
    if (i%100 == 0):
        loss = sess.get_current_loss()
        print("[HUGECTR][INFO] iter: {}; loss: {}".format(i, loss))
    if (i%1000 == 0 and i != 0):
        sess.check_overflow()
        sess.copy_weights_for_evaluation()
        for _ in range(solver_config.max_eval_batches):
            sess.eval()
        metrics = sess.get_eval_metrics()
        print("[HUGECTR][INFO] iter: {}, {}".format(i, metrics))
sess.download_params_to_files("./", 10000)
```

### Epoch Mode Training ###
**NOTE** This sample employs the epoch mode training and enables the feature of ModelOversubscriber. Please prepare the JSON file, Norm dataset with file lists together with keyset files and a temporary write-enabled directory before running this script. See [HugeCTR Python Interface Notebook](../notebooks/python_interface.ipynb) for help.
```bash
from hugectr import Session, solver_parser_helper, get_learning_rate_scheduler
from mpi4py import MPI
json_file = "your_config.json"
temp_dir = "./your_temp_embedding_dir"
dataset = [("file_list."+str(i)+".txt", "file_list."+str(i)+".keyset") for i in range(5)]
solver_config = solver_parser_helper(seed = 0,
                                    batchsize = 16384,
                                    batchsize_eval =16384,
                                    model_file = "",
                                    embedding_files = [],
                                    vvgpu = [[0]],
                                    use_mixed_precision = False,
                                    scaler = 1.0,
                                    i64_input_key = False,
                                    use_algorithm_search = True,
                                    use_cuda_graph = True,
                                    repeat_dataset = False)
lr_sch = get_learning_rate_scheduler(json_file)
sess = Session(solver_config, json_file, True, temp_dir)
data_reader_train = sess.get_data_reader_train()
data_reader_eval = sess.get_data_reader_eval()
data_reader_eval.set_source("file_list.5.txt")
model_oversubscriber = sess.get_model_oversubscriber()
iteration = 0
for file_list, keyset_file in dataset:
    data_reader_train.set_source(file_list)
    model_oversubscriber.update(keyset_file)
    while True:
        lr = lr_sch.get_next()
        sess.set_learning_rate(lr)
        good = sess.train()
        if good == False:
            break
        if iteration % 100 == 0:
            sess.check_overflow()
            sess.copy_weights_for_evaluation()
            data_reader_eval = sess.get_data_reader_eval()
            good_eval = True
            j = 0
            while good_eval:
                if j >= solver_config.max_eval_batches:
                    break
                good_eval = sess.eval()
                j += 1
            if good_eval == False:
                data_reader_eval.set_source()
            metrics = sess.get_eval_metrics()
            print("[HUGECTR][INFO] iter: {}, metrics: {}".format(iteration, metrics))
        iteration += 1
    print("[HUGECTR][INFO] trained with data in {}".format(file_list))
sess.download_params_to_files("./", iteration)
```

### Inference ###
**NOTE** Please prepare the inference JSON file, the dense model file, the sparse model file and inference data file in the right format before running this script. See [HugeCTR Inference Notebook](../notebooks/hugectr_inference.ipynb) for help.
```bash
from hugectr.inference import CreateParameterServer, CreateEmbeddingCache, InferenceSession
from mpi4py import MPI
config_file = "your_inference_config.json"
model_name = "your_model_name"
data_path = "your_inference_inputs_data.txt"
use_embedding_cache = True
# read data from file
data_file = open(data_path)
labels = [int(item) for item in data_file.readline().split(' ')]
dense_features = [float(item) for item in data_file.readline().split(' ')]
embedding_columns = [int(item) for item in data_file.readline().split(' ')]
row_ptrs = [int(item) for item in data_file.readline().split(' ')]
# create parameter server, embedding cache and inference session
parameter_server = CreateParameterServer([config_file], [model_name], False)
embedding_cache = CreateEmbeddingCache(parameter_server, 0, use_gpu_embedding_cache, 0.2, config_file, model_name, False)
inference_session = InferenceSession(config_file, 0, embedding_cache)
# make prediction and calculate accuracy
output = inference_session.predict(dense_features, embedding_columns, row_ptrs)
```
