# BigNLP HP Tool

This tool searches for the Hyper-Parameters (HPs) that achieve the highest throughput for training 
Large Language Models (LLMs) using NeMo-Megatron. It also searches for the inference HPs that 
achieve the highest throughput and the lowest latency.

## Table of contents
- [1. HP Tool Capabilities](#1-hp-tool-capabilities)


## 1. HP Tool Capabilities

The Hyper-Parameter (HP) tool is intended to quickly iterate over different model configurations, 
to quickly find the best configuration with minimal time and money spending. To achieve that, our 
tool provides several different capabilities, as shown in the table below:

| Feature                              | GPT-3 | T5  | mT5 |
| ------------------------------------ | ----- | --- | --- |
| Model Size Recommendation            | Yes   | Yes | Yes |
| Base Config Generation               | Yes   | Yes | Yes |
| Training HP Search                   | Yes   | Yes | Yes |
| Parallel Training HP Search          | Yes   | Yes | Yes |
| Inference HP Search                  | Yes   | No  | No  |
| Parallel Inference HP Search         | No    | No  | No  |
| SLURM Based Clusters                 | Yes   | Yes | Yes |
| Base Command Platform Based Clusters | No    | No  | No  |

### 1.1. Model Size Recommendation
For users who do not know what model size they wish to train, our tool is capable of recommending 
a model size, given the hardware and training constraints. If the number of GPUs, the TFLOPS per GPU, 
the maximum time to train, and the number of tokens to train for are known, then our tool can 
recommend a model size that can be trained with the specified hardware and time constraints.

For example, if the user has 20 NVIDIA DGX nodes available (80GB GPU memory), and wants to train a 
GPT-3 model for a maximum of 5 days, the tool will recommend using a 5B parameter GPT-3 model. 
The tool will perform a best effort guess using heuristics, so the results might not be perfect.


### 1.2. Base Config Generation
If the model size is provided by the user, or after the model size is generated (as shown in section 1.1), 
the tool will generate a base configuration for the given model. This configuration will be a valid, 
runnable configuration in YAML format, which can be trained using NeMo-Megatron. However, this config 
will not be optimized at this stage.


### 1.3. Training HP Search
Given the input model size (generated in step 1.1) and the base configuration (generated in step 1.2), 
the tool will now search over four different critical Hyper-Parameters, that have great impact on the 
training throughput: Tensor Parallelism (TP), Pipeline Parallelism (PP), Micro Batch Size (MBS), 
and Activation Checkpointing Layers (ActCkpt).

First, the tool will use heuristics to choose good candidates for those four parameters to generate 
the grid of candidate configurations. All the candidate configs will be saved to the results directory, 
and will include YAML files with the corresponding config. NOTE: some of these configs might not work, 
due to high memory usage or for other reasons. The next step will determine which configs are valid.

Once all the candidate configs are generated, the tool will use heuristics to sort the most promising 
candidate configs. Then, the tool will launch the top `limit_search_runs` most promising candidates 
in parallel, to perform a grid search over the four training parameters. This search will launch the 
jobs using NeMo-Megatron, and it will train each config for `max_minutes_per_run` minutes, on the 
target cluster. During this search, the jobs will run in the minimum number of nodes required, using 
Data Parallelism of 1 (DP=1) in most cases.


### 1.4. Inference HP Search
The tool can also search the best HPs for inference purposes. It will empirically measure the 
throughput and latency for each given configuration in the grid search space, and return a comprehensive 
table with all the numbers. The tool will search over three different critical HPs, which have great 
impact on the inference throughput and latency: Tensor Parallelism (TP), Pipeline Parallelism (PP), and 
Batch Size (BS). Technically, the tool is also capable of searching over different input/outpu sequence 
lengths. However, we do not recommend adding multiple different sequence lengths to the same search, 
since the model that uses the shortest sequence lengths will always achieve higher throughput and lower 
latency. Therefore, we recommend performing several different inference searches for different sequence 
lengths.

Once the search space has been defined, the tool will launch a job for each config, and measure the 
throughput and latency.i This search will launch the jobs using NeMo-Megatron on the target cluster. 
Once all the jobs have finished running, the final result will be summarized in a CSV file.


## 2. Usage
In this section, we will explain how to run each of the stages described in section 1. 

### 2.1. General Configuration
First, our configuration setup assumes that the `/opt/bignlp` directory has been copied from the container 
to the local file system. And we assume that bignlp-hp-tool and BigNLP-Inference-Scripts are both 
located inside the `bignlp` directory.

The first parameter that must be set is the `bignlp_hp_tool_path` parameter inside the `conf/config.yaml` 
file. This parameter must point to the absolute path where the `bignlp-hp-tool` repository is stored in 
the file system. Additionally, if using a Slurm based cluster, the config file in the 
`conf/cluster/bcm.yaml` subfolder has the parameters to set the generic cluster related information, 
such as the `partition` or `account` parameters.

The `bignlp_hp_tool_path` parameter will automatically be mounted to the container at the same path as 
in the local file system. Any additional directories that should be mounted must be specified using the
`container_mounts` parameter. If the paths contain the colon character (`:`), the code will assume both 
the source and destination paths are provided. Otherwise, the given paths will be mounted to the same 
path inside the container.

The `bignlp_inference_path` must point to the path where BigNLP-Inference-Scripts is located. The location 
specified in the default config should be valid if `/opt/bignlp` was extracted correctly. Next, the 
`data_dir` value must point to the path where the training dataset is located. Note that the dataset 
for GPT-3, T5 and mT5 values will be different, so modify this parameter accordingly. Follow the data 
preparation steps to learn how to download and preprocess the datasets for each model. The dataset in 
this path does not need to be the full size dataset; only a small representative sample of the dataset 
is needed, since the HP tool does not train the models to convergence. Finally, the `base_results_dir` 
parameter can be modified to point to the location where the results will be stored. See all the 
parameters for the `conf/config.yaml` file below:

```yaml
defaults:
  - _self_
  - cluster: bcm
  - search_config: gpt3/5b
  - override hydra/job_logging: stdout

bignlp_hp_tool_path: ???  # Path should end with bignlp-hp-tool.
bignlp_inference_path: ${bignlp_hp_tool_path}/../BigNLP-Inference-Scripts
data_dir: ${bignlp_hp_tool_path}/../bignlp-scripts/data
base_results_dir: ${bignlp_hp_tool_path}/results

training_container: nvcr.io/ea-bignlp/bignlp-training:22.04-py3
container_mounts:
    - null
```

### 2.1. Running Pre-Defined Configs
The pre-defined configs we provide have been well tested, and the outputs produced by the HP tool 
have been verified manually. Running one of these configs will first generate a base config file for 
the specified model size. Then, it will launch the training and inference grid search jobs. When 
all the jobs have finished, a final recommendation will be produced for both training and inference, 
which will show the optimal hyper-parameters for the given model.

The pre-defined configs can be found in the `conf/search_config` directory. Each YAML file shows one 
model type (GPT-3, T5 or mT5) and one model size (up to 175B parameters for GPT-3 and up to 42B 
parameters for T5/mT5). To run the desired config, we will need to modify the `search_config` 
parameter in the `conf/config.yaml` file. For example, if we wish to run a 5B GPT-3 model, we can 
set this value to `gpt3/5b` (the .yaml ending should not be included). 

The tool will always generate the base configuration for the given model first. Then, the 
`run_training_hp_search` and `run_inference_hp_search` parameters can be set to `True`, 
to run the training and inference HP searches, respectively. If any of these two parameters are set 
to `False`, the corresponding pipeline will not be executed. Once these parameters are set, we can 
run the tool calling `python3 main.py`. 

#### 2.1.1. Model Config
To run the `gpt3/5b` config, we need to set up the `conf/search_config/gpt3/5b.yaml` file correctly.
The config is split in two sections: `train_settings` and `inference_settings`. 

```yaml
train_settings:
  model_size_in_b: 5 # unit in billion parameters
  num_nodes: 20
  gpus_per_node: 8
  max_training_days: 5 # unit in days
  limit_search_runs: 100 # Max number of runs to be launched in parallel for grid search.
  output_top_n: 10  # The result will print the top N fastest training configs.
  max_minutes_per_run: 40 # minutes per run for the grid search.
  tflops_per_gpu: 140  # Estimated tflops per GPU.
  num_tokens_in_b: 300  # Unit in billions, typically 300B for GPT3 models.
  vocab_size: 51200
  logs: ${base_results_dir}/${search_config_value}  # Example base_results_dir/gpt3/126m
  tensor_parallel_sizes: null  # null to use our recommendation, or a list, such as [1, 2, 4, 8]
  pipeline_parallel_sizes: null  # null to use our recommendation, or a list, such as [1, 2, 4, 8, 10]
  micro_batch_sizes: null  # null to use our recommendation, or a list, such as [1, 2, 4, 8, 16]
  act_ckpt_layers: null  # null to use our recommendation, or a list, such as [0, 1, 2, 3]
 
inference_settings:
  vocab_size: 51200
  start_id: 50256
  end_id: 50256
  input_seq_len: 60
  output_seq_len: 20
  top_n: 10
  logs: ${base_results_dir}/${search_config_value}  # Example base_results_dir/gpt3/126m
  tensor_parallel_sizes: [1, 2, 4, 8]
  pipeline_parallel_sizes: [1, 2, 3, 4]
  max_batch_sizes: [1, 2, 8, 16, 32, 64, 256]
```

#### 2.1.1. Base Config Generation
Every time we call `python3 main.py`, a base configuration will be stored for the given model 


#### 2.1.2. Training HP Search
A

#### 2.1.3. Inference HP Search
A

### 2.2. Running Custom Model Size Configs
A

### 2.3. Interpreting Final Results
A

### 2.4. Logging Runs with Weights and Biases (W&B)
Weights and Biases (W&B) can be used to log all the training search runs. To achieve this, the 
`wandb` parameters must be modified in the `conf/config.yaml` file. First, `enable` must be set to 
`True`. Then, the `api_key_file` must be set to point to the path where the file which contains 
the W&B API key. The API key must be in the first line of that file. Finally, the `project` parameter
must have the name of the W&B project where the metrics will be stored. The name of each run does not 
need to be provided. It will be automatically generated by the tool, using the model name, model size, 
and hyper-parameters used for each specific run.

```yaml
wandb:  # Weights and Biases (W&B) logging.
    enable: True 
    api_key_file: null
    project: bignlp-hp-tool
```



