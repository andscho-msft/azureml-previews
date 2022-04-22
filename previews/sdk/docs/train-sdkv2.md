# Train models with the Azure ML Python SDK (v2) (preview)

In this article, you learn how to configure and submit Azure Machine Learning jobs to train your models. Snippets of code explain the key parts of configuration and submission of a training job. Then use one of the [example notebooks](https://github.com/Azure/azureml-examples/tree/sdk-preview/sdk) to find the full end-to-end working examples.

## Prerequisites

* If you don't have an Azure subscription, create a free account before you begin. Try the [free or paid version of Azure Machine Learning](https://azure.microsoft.com/free/) today
* The Azure Machine Learning SDK v2 for Python - [install SDKv2](install-sdkv2.md)
* An Azure Machine Learning workspace

### Clone examples repository

To run the training examples, first clone the examples repository and change into the `sdk` directory:

```bash
git clone --depth 1 https://github.com/Azure/azureml-examples --branch sdk-preview
cd azureml-examples/sdk
```

Using `--depth 1` clones only the latest commit to the repository, which reduces time to complete the operation.

## Start on your local machine

To start with let us run a script which trains a model using `lightgbm`. The script file is available [here](https://github.com/Azure/azureml-examples/blob/sdk-preview/sdk/jobs/single-step/lightgbm/iris/src/main.py). The script needs 3 inputs

* _input data_: We will use data from a web location for our run - [web location](https://azuremlexamples.blob.core.windows.net/datasets/iris.csv). In this example, we are using a file in a remote location for brevity, but you can use a local file as well.
* _learning-rate_: We will use a learning rate of _0.9_
* _boosting_: We will use the Gradient Boosting _gdbt_

We run this script file as follows

```bash
cd jobs/single-step/lightgbm/iris

python src/main.py --iris-csv https://azuremlexamples.blob.core.windows.net/datasets/iris.csv  --learning-rate 0.9 --boosting gbdt
```

The output expected is as follows:

```terminal
2022/04/21 15:02:44 INFO mlflow.tracking.fluent: Autologging successfully enabled for lightgbm.
2022/04/21 15:02:44 INFO mlflow.tracking.fluent: Autologging successfully enabled for sklearn.
2022/04/21 15:02:45 INFO mlflow.utils.autologging_utils: Created MLflow autologging run with ID 'a1d5f652796e4d88961176166de52253', which will track hyperparameters, performance metrics, model artifacts, and lineage information for the current lightgbm workflow
lightgbm\engine.py:177: UserWarning: Found `num_iterations` in params. Will use it instead of argument
[LightGBM] [Warning] Auto-choosing col-wise multi-threading, the overhead of testing was 0.000164 seconds.
You can set `force_col_wise=true` to remove the overhead.
[LightGBM] [Warning] No further splits with positive gain, best gain: -inf
[LightGBM] [Warning] No further splits with positive gain, best gain: -inf
[LightGBM] [Warning] No further splits with positive gain, best gain: -inf
```

## Move to the cloud

Now that the local run works, we will move this run to an Azure Machine Learning workspace. To run this on Azure ML we need the following:

1. A workspace to run
1. A compute on which to run it
1. An environment on the compute to ensure we have the required packages to run our script

Let us tackle these steps below

### 1. Connect to the workspace

To connect to the workspace, we need identifier parameters - a subscription, resource group and workspace name. We will use these details in the `MLClient` from `azure.ml` to get a handle to the required Azure Machine Learning workspace. To authenticate, we use the [default azure authentication](https://docs.microsoft.com/python/api/azure-identity/azure.identity.defaultazurecredential?view=azure-python). Check this [example](https://github.com/Azure/azureml-examples/blob/sdk-preview/sdk/jobs/configuration.ipynb) for more details on how to configure credentials and connect to a workspace.

<!--[!notebook-python[] (~/azureml-examples/blob/sdk-preview/sdk/jobs/single-step/lightgbm/iris/lightgbm-iris-sweep.ipynb?name=connect-workspace)]-->

```python
#import required libraries
from azure.ml import MLClient
from azure.identity import DefaultAzureCredential

#Enter details of your AML workspace
subscription_id = '<SUBSCRIPTION_ID>'
resource_group = '<RESOURCE_GROUP>'
workspace = '<AML_WORKSPACE_NAME>'

#connect to the workspace
ml_client = MLClient(DefaultAzureCredential(), subscription_id, resource_group, workspace)
```

### 2. Create compute

We will create a compute called `cpu-cluster` for our job. This is done as follows:

<!--[!notebook-python[] (~/azureml-examples/blob/sdk-preview/sdk/jobs/configuration.ipynb?name=create-cpu-compute)]-->

```python
from azure.ml.entities import AmlCompute

# specify aml compute name.
cpu_compute_target = 'cpu-cluster'

try:
    ml_client.compute.get(cpu_compute_target)
except Exception:
    print('Creating a new cpu compute target...')
    compute = AmlCompute(name=cpu_compute_target, size="STANDARD_D2_V2", min_instances=0, max_instances=4)
    ml_client.compute.begin_create_or_update(compute)
```

### 3. Environment to run the script

To run our script on `cpu-cluster`, we need an environment which has the required packages and dependencies to run our script. There are a few options available for environments:

1. Use a curated environment in your workspace - Azure ML offers several curated [environments](https://ml.azure.com/environments) which cater to various needs.
1. Use a custom environment - Azure ML allows you to create your own environment using
   * A docker image
   * A base docker image with a conda YAML to customize further
   * A docker build context

   Check this [example](https://github.com/Azure/azureml-examples/blob/sdk-preview/sdk/assets/environment/environment.ipynb) on how to create custom environments.

For our case we will use a curated environment provided by Azure ML for `lightgbm` called `AzureML-lightgbm-3.2-ubuntu18.04-py37-cpu`

### 4. Submit a job to run the script

To run this script we will use a `command`. The command will be run by submitting it as a `job` to Azure ML.

<!--[!notebook-python[] (~/azureml-examples/blob/sdk-preview/sdk/jobs/single-step/lightgbm/iris/lightgbm-iris-sweep.ipynb?name=create-command)]-->

```python
from azure.ml import command, Input
#define the command
command_job=command(
    code='./src',
    inputs={'iris_csv':Input(type='uri_file', path='https://azuremlexamples.blob.core.windows.net/datasets/iris.csv')},
    command = 'python main.py --iris-csv ${{inputs.iris_csv}}',
    environment='AzureML-lightgbm-3.2-ubuntu18.04-py37-cpu@latest',
    compute='cpu-cluster'
)
```

<!--[!notebook-python[] (~/azureml-examples/blob/sdk-preview/sdk/jobs/single-step/lightgbm/iris/lightgbm-iris-sweep.ipynb?name=run-command)]-->

```python
# submit the command
returned_job = ml_client.jobs.create_or_update(command_job)
# get a URL for the status of the job
returned_job.services["Studio"].endpoint
```

In the above we have configured the following:

* `code` - This is the path where the code to run the command is located. In our case the python file is located inside the src folder
* `command` - This is the command that needs to be run.
* `inputs` - This is the dictionary of inputs using name value pairs to the command. The key is a name for the input within the context of the job and the value is the input value. Inputs are referenced in the `command` using the `${{inputs.<input_name>}}` expression. To use files or folders as inputs, we can use the `Input` class.

For more details refer to the reference documentation [here](https://review.docs.microsoft.com/python/api/azure-ml/azure.ml?view=azure-ml-py&branch=sdk-cli-v2-preview-master#azure-ml-command)

## Improve the model using hyperparameter sweep

Now that we have run a job on Azure, let us make it better using Hyperparameter tuning. Also called hyperparameter optimization, this is the process of finding the configuration of hyperparameters that results in the best performance. Azure Machine Learning provides a `sweep` function on the `command` to do hyperparameter tuning. 

To perform a sweep, there needs to be input(s) against which the sweep needs to be performed. These inputs can have a discrete or continuos value. The `sweep` function will run the `command` multiple times using different combination of input values specified. Each input is a dictionary of name value pairs. The key is the name of the hyperparameter and the value is the parameter expression. 

Let us improve our model by sweeping on `learning_rate` and `boosting` inputs to the script. In the previous step, we used a specific value for these, but now we will use a range or  choice of values.

<!--[!notebook-python[] (~/azureml-examples/blob/sdk-preview/sdk/jobs/single-step/lightgbm/iris/lightgbm-iris-sweep.ipynb?name=search-space)]-->

```python
# we will reuse the command_job created before. we call it as a function so that we can apply inputs
# we do not apply the 'iris_csv' input again -- we will just use what was already defined earlier
from azure.ml.sweep import Choice, Uniform
command_job_for_sweep = command_job(
    learning_rate=Uniform(min_value=0.01, max_value=0.9),
    boosting=Choice(values=["gbdt", "dart"]),
)
```

Now that we have defined the parameters, let us run the sweep

<!--[!notebook-python[] (~/azureml-examples/blob/sdk-preview/sdk/jobs/single-step/lightgbm/iris/lightgbm-iris-sweep.ipynb?name=configure-sweep)]-->

```python
# apply the sweep parameter to obtain the sweep_job
sweep_job = command_job_for_sweep.sweep(
    compute='cpu-cluster',
    sampling_algorithm='random',
    primary_metric='test-multi_logloss',
    goal='Minimize'
)

#define the limits for this sweep
sweep_job.set_limits(max_total_trials=20, max_concurrent_trials=10, timeout=7200)
```

<!--[!notebook-python[] (~/azureml-examples/blob/sdk-preview/sdk/jobs/single-step/lightgbm/iris/lightgbm-iris-sweep.ipynb?name=run-sweep)]-->

```python
# submit the sweep
returned_sweep_job = ml_client.create_or_update(sweep_job)
# get a URL for the status of the job
returned_sweep_job.services["Studio"].endpoint
```

As seen above, the `sweep` function allows user to configure the following key aspects:

* `sampling_algorithm`- The hyperparameter sampling algorithm to use over the search_space. Allowed values are `random`, `grid` and `bayesian`.
* `objective` - the objective of the sweep
  * `primary_metric` - The name of the primary metric reported by each trial job. The metric must be logged in the user's training script using `mlflow.log_metric()` with the same corresponding metric name.
  * `goal` - The optimization goal of the objective.primary_metric. The allowed values are `maximize` and `minimize`.
* `compute` - Name of the compute target to execute the job on.
* `limits` - Limits for the sweep job

Once this job completes, you can look at the metrics and the job details in the [Azure ML Portal](https://ml.azure.com/). The job details page will identify the best performing child run.

:::image type="content" source="media/sweep-best-run.jpg" alt-text="Best Run of Sweep":::

## Distributed training

Azure Machine Learning supports PyTorch, TensorFlow, and MPI-based distributed training. Let us look at how to configure a command for distribution for the `command_job` we created earlier

```python
# Distribute using PyTorch
from azure.ml import PyTorchDistribution
command_job.distribution = PyTorchDistribution(process_count_per_instance=4)

# Distribute using TensorFlow
from azure.ml import TensorFlowDistribution
command_job.distribution = TensorFlowDistribution(parameter_server_count=1, worker_count=2)

# Distribute using MPI
from azure.ml import MpiDistribution
job.distribution = MpiDistribution(process_count_per_instance=3)
```

Once the distribution is configured, then the job can be submitted as usual.

## Next steps

Try these next steps to learn how to use the Azure Machine Learning SDK (v2) for Python:

1. [Use pipelines with the Azure ML Python SDK (v2)](./pipeline-sdkv2.md)
