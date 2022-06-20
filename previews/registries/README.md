

### Welcome

* Request access to Registries preview: https://forms.office.com/r/VrX6rE2Ut0
* Template to report bugs, issues, feature requests: https://github.com/Azure/azureml-previews/issues/new?assignees=ManojBableshwar&labels=registries&template=registries-bug.md&title=Registries+bug%3A+  - GitHub issues (not emails) are the preferred medium for reporting issues, getting support, and sharing feedback.


### What are AzureML Registries?  

Registries are org wide repositories of ML assets such as Models, Environments, Components and more. Registries enable seamless MLOps across different IT environments such as dev, test and prod. Using Registries, Ops professionals can promote a ML model from the dev environment in which Data Scientists trained the model, to test environments in which, let’s say AB testing is done, and finally to production environment, while tracking linage across the ML lifecycle. Real-world ML models are complex and come with scaffolding code to run them. For example, a batch inference scenario that needs a ML model to score on, conda environments and custom docker image for the dependencies, and a pipeline of ML tasks to pre-process the data before scoring. Registries let you package all these – the models, pipelines, and environments – into a cohesive collection and deploy across many AzureML Workspaces in different Azure subscriptions in your organization. 

Data science teams get started with Machine Learning on Azure by creating a AzureML Workspace. Workspace offers a central location to organize and track ML activates - experiments, compute, datasets, models, endpoints, environments and more. A workspace is typically used by one or few teams and is associated with an Azure subscription. From an operations standpoint, keeping security, compliance and cost management in mind, customers typically isolate dev environments and prod environments in different Azure subscriptions, with corresponding AzureML workspaces. Today, you cannot deploy a model registered in one workspace into a different workspace or use pipelines published to one workspace in a different one. This makes MLOps hard and fragile because ML assets created by data science teams in dev workspaces in dev subscriptions must be manually copied over to prod workspaces in prod subscriptions. Sure, you can automate the copy tasks using a DevOps systems but more importantly, you lose lineage and traceability when you move assets across workspaces today – What dataset was used to train a model? Which was the experiment and what were the metrics to show this model was a good candidate? Where do I go to retrain this production model because I see that the performance is degrading? 

Registries, much like a Git repository, decouples ML assets from workspaces and hosts them in a central location, making them available to all workspaces in your organization. You start by iterating within a workspace and when you have a good candidate asset, you can publish it to a Registry in the say way you would register a model or a pipeline with a workspace. Meaning, you use the same `az ml` cli commands to publish assets to either a workspace or a Registry. You can even promote an asset already registered in a workspace to a Registry. Just like assets in a workspace are versioned, assets in Registries support versioning too. This means a model in a Registry can have different versions with a newer version deployed for AB testing to an endpoint in a test workspace while an older and stable continues to be deployed to an endpoint in the production workspace. With online endpoints v2, even after you deploy a new version to production, you can gradually shift traffic from the older to the newer version, keeping a margin of safety. Across this entire flow, the Models UI in Registries will visualize the complete lifecycle of this model – the dataset used to train the model, the job that trained the model with metrics, the various dev, test and prod endpoints to which the model is deployed across different workspaces. 
Registries not only enable better MLOps, but also foster great collaboration with your organization. The Registries UI in AzureML Studio offers gallery to discover all ML assets in your organization. It aggregates assets across multiple org level Registries and smartly curates content such as the most popular assets, most used assets, assets featured by the admins and more, opening up creative collaboration opportunities: Has someone already built a utility component to extract and featurize data from the inventory database? Has someone already downloaded and tokenized Enron Email dataset in our org? Is there a component that warps the latest release of TenserFlow 2.0 and are the any jobs to show how this works? In addition to sharing assets within the enterprise, Registries also enables a framework for public sharing of AzureML compatible ML assets. To begin with, Microsoft will publish a set of valuable utilities and popular per-trained models that will be made available to all AzureML users. In future, we will enable our partners and customers to share assets publicly outside their organization. 

To summarize,

* Registry is a collection of AzureML Assets that can be used by one or more Workspaces.
* Registries facilitate sharing of assets among teams working across multiple Workspaces in an organization.
Registries, by virtue of sharing assets, enable MLOps flow of assets across dev -> test -> prod environments.
* Registries can make Workspaces more project centric by decoupling iterative assets in Workspaces and final/prod ready assets in Registries.
* Assets in Registries can be used by Workspaces in any region (specified while creating the a Registry), with the service transparently replicating necessary resources (code snapshots, docker images) in the background.

### Registry Concepts

#### ARM Representation  

AzureML Registries are Azure Resources that can be managed using Azure Resource Manager (ARM), under the `Microsoft.MachineLearningServices` resource provider. This implies:
* A Registry is placed under a Azure Resource Group
* A Registry is associated with a Azure Subscription that is used to bill resources consumed by the Registry
* In addition to the `microsoft.machinelearningservices/registries` core resource type, creating a Registry creates additional Azure resources such as storage accounts and Azure Container Registries.

#### Replication
Registries are supposed to enable org wide (tenant level) sharing of assets. As such, users can create jobs and endpoints using Registry assets in workspaces if different Azure regions. A Registry can replicate assets to multiple regions ensuring that Workspaces in different regions have low latency access to assets content (code, binaries, docker images, etc.) The regions to which a Registry replicates assets is defined when the Registry is created and can also be updated at a later point. Using assets from a Registry in a Workspace that is in a Region to which the Registry has not been configured to replicate assets will result in an error. 

#### Access control 
Registry permissions will be governed by RBAC, allowing teams to have flexibility between highly curated (tight permissions) vs everyone can contribute (lose permissions). 
* Write access that allows users to create assets in Registry and Manage access that allows users to edit Registry properties is always configured through RBAC.
* Read access allows users to create jobs and endpoints in Workspaces that use assets from a Registry can either be set to i) Tenant level access - all users in a tenant can access the Registry ii) RBAC level access - users granted access to RBAC will be able to use assets from the Registry. 

Currently, a Workspace does not need any permission grant or configuration to use assets from a Registry as long as the user submitting the job as permissions to read assets from the Registry. We may revisit this in future if we discover a need for users to lock down a Workspace and prevent it from using assets outside the Workspace.

For Registry specific RBAC permissions, see https://docs.microsoft.com/en-us/azure/role-based-access-control/resource-provider-operations#microsoftmachinelearningservices. 

![Registry RBAC](./images/rbac-permissions.png)


#### Comparision to Workspace
* Assets created in a Workspace can be used only in that specific Workspace but assets created in a Registry can be used in any Workspace in that tenant. 
* Assets created in a Workspace are referenced as `azureml:<assetname>:<assetversion>` in AzureML CLI V2 YAML. Assets created in a Registry are referenced as `azureml://registries/<registry_name>/<asset_type>/<asset_name>/versions/<asset_version>` when they need to be used in a Workspace. 
* You can browse Assets created in a Workspace in a the AzureML Studio Workspace UI (Components, Environments, Models, Datasets hubs) where as Assets created in a Registry are available for browsing in the AzureML Studio Registries hub in the global home page (ml.azure.com/home).



### Preview Pre-requisites


Step 1: Setup your CLI V2 environment as explained here: https://docs.microsoft.com/en-us/azure/machine-learning/how-to-configure-cli?tabs=public (az cli, az ml cli, create and set default workspace, etc.)

Make sure you have set your Azure subscription and defaults for location, resource group and workspace for the Az cli

```
az account set -s <subscription_id> 
az configure --defaults group=<RG> workspace=<WORKSPACE> location=<LOCATION>
```


Step 2: Uninstall the V2 CLI (you will install private version in next step)

```
az extension remove -n ml
```

Step 3: Install the private CLI as shown below (or the specific private version that was shared with you)

```
az extension add --source https://azuremlsdktestpypi.blob.core.windows.net/wheels/azureml-v2-cli-e2e-test/65104501/ml-0.0.65104501-py3-none-any.whl --pip-extra-index-urls https://azuremlsdktestpypi.azureedge.net/azureml-v2-cli-e2e-test/65104501 --yes
```

Step 4: Enable Private Preview Features
  
Linux
```
export AZURE_ML_CLI_PRIVATE_FEATURES_ENABLED=true
```
Windows Cmd
```
set AZURE_ML_CLI_PRIVATE_FEATURES_ENABLED=true
```
Powershell
```
$env:AZURE_ML_CLI_PRIVATE_FEATURES_ENABLED = 'true'
```


### How to create an AzureML Registry?

Currently, you need to use an [ARM template](./arm/README.md) to create a Registry. We will support creating Registries using CLI and UI in future. 

### What assets can I create in a Registry?

Today
* Environments
* Components
* Models

In future:
* Datasets 

### How to create assets in a Registry? 

You use the same `az ml <asset_type> create` commands available with AzureML CLI V2 with an additional parameter `--registry-name`.

We will add support for Python SDK and UI in future. 

#### Example - Command component + inline/local environment with public docker image

component.yml

```yaml
name: awesome_component
version: 101
type: command
command: echo 'I live in Registry and run in Workspace'
environment:
  image: docker.io/python
```

```
az ml component create --file component.yml --registry-name <registry_name_placeholder>
```

### How to view assets created in Registries? 

#### View with CLI

You can list assets using the CLI

```
az ml <asset_type> list --registry-name <registry_name_placeholder>
```

You can show a specific asset using the CLI

```
az ml <asset_type> show --name <asset_name> --version <asset_version> --registry-name <registry_name_placeholder>
```

#### View in AzureML Studio 
The Registries UI hub is located in the AzureML global homepage: ml.azure.com/home. For the time-being you may need to use append a flight to the URL: https://ml.azure.com/registries?flight=GlobalRegistries 


![Registry hub](./images/registry-hub.png)

### How to use assets from a Registry?

You use assets from Registries in Jobs and Endpoints created in Workspaces. To do so, the asset name needs a Registry scope qualifier, hence the name of assets living in Workspaces looks like this: 

```
azureml://registries/<registry_name>/<asset_type>/<asset_name>/versions/<asset_version>
```

You can find the fully formed name string on the asset details page in the UI

![Component detail page](./images/component-detail.png)

#### Example - simple pipeline job that uses the Command Component created above

pipeline.yml
```yaml
type: pipeline
jobs:
  awesome_job:
    type: command
    component: azureml://registries/<registry_name>/components/awesome_component/versions/101
    compute: azureml:cpu-cluster
```

```
az ml job create --file pipeline.yml
```

![Sample job using Component from Registry](./images/sample-job.png)

#### Use with CLI - Models
<todo: model deployment using CLI>

#### Use in AzureML Studio - Jobs in Designer 

Open the Designer in AzureML Studio with flight=registryAsset appended to the url. 

You can then filter assets from a specific Registry and then drag and drop them to the Designer canvas to submit jobs.

![Registry in Designer](./images/registry-designer.gif)

#### Use in AzureML Studio - Models 

<todo: model deployment>

#### Use in Python SDK 
<todo: jobs and models in Python SDK>

### I get the concepts, show me more examples...

You can try the samples in AzureML examples repo: https://github.com/Azure/azureml-examples.git


Clone repo and navigate to sample registry
```
git clone https://github.com/Azure/azureml-examples.git

cd azureml-examples/cli/jobs/pipelines-with-components/nyc_taxi_data_regression

```

Create components in Registry. Replace ContosoMLjun14 with your registry name in below commands. Note that we are setting the environment on the cli from different Registry because curated environments are not yet supported in Registry. This limitation will be addressed shortly. 

```
 az ml component create --file prep.yml --query name -o tsv --registry-name ContosoMLjun14 --set environment=azureml://registries/CuratedRegistry/environments/AzureML-sklearn-0.24-ubuntu18.04-py37-cpu/versions/35 

 az ml component create --file transform.yml --registry-name ContosoMLjun14 --set environment=azureml://registries/CuratedRegistry/environments/AzureML-sklearn-0.24-ubuntu18.04-py37-cpu/versions/35

 az ml component create --file train.yml --registry-name ContosoMLjun14 --set environment=azureml://registries/CuratedRegistry/environments/AzureML-sklearn-0.24-ubuntu18.04-py37-cpu/versions/35 

 az ml component create --file score.yml --registry-name ContosoMLjun14 --set environment=azureml://registries/CuratedRegistry/environments/AzureML-sklearn-0.24-ubuntu18.04-py37-cpu/versions/35 

 az ml component create --file predict.yml --registry-name ContosoMLjun14 --set environment=azureml://registries/CuratedRegistry/environments/AzureML-sklearn-0.24-ubuntu18.04-py37-cpu/versions/35 

```

Query the Registry to check for the registered Components. 
```
# az ml component list --registry-name ContosoMLjun14 --query [].id
[
  "azureml://registries/ContosoMLjun14/components/predict_taxi_fares",
  "azureml://registries/ContosoMLjun14/components/score_model",
  "azureml://registries/ContosoMLjun14/components/train_linear_regression_model",
  "azureml://registries/ContosoMLjun14/components/taxi_feature_engineering",
  "azureml://registries/ContosoMLjun14/components/prep_taxi_data",
  "azureml://registries/ContosoMLjun14/components/awesome_component",
  "azureml://registries/ContosoMLjun14/components/hello_python_world"
]

```

Edit the pipeline.yml with components names pointing to assets created in Registry 

Before editing the components use local files:

```
# cat pipeline.yml | grep "component:"
    component: file:./prep.yml
    component: file:./transform.yml
    component: file:./train.yml
    component: file:./predict.yml
    component: file:./score.yml
```

After editing the components refer to registry:

```
# cat pipeline.yml | grep "component:"
    component: azureml://registries/ContosoMLjun14/components/prep_taxi_data
    component: azureml://registries/ContosoMLjun14/components/taxi_feature_engineering
    component: azureml://registries/ContosoMLjun14/components/train_linear_regression_model
    component: azureml://registries/ContosoMLjun14/components/predict_taxi_fares
    component: azureml://registries/ContosoMLjun14/components/score_model
```

Submit the pipeline Job
```
az ml job create --file pipeline.yml 
```

You must be able to verify that the child jobs in the pipeline are using components from Registries in both the output of `az ml job show` and in the UI as explained in the screenshot above.

![NYC pipeline job](./images/pipeline-nyc.png)


### Coming soon...
* Update to this doc to show how to create and use Models from Registry
* Python SDK support for assets from Registry
* Curated environment support
* MLflow model support
* Batch Endpoint support


### Appendix

Old preview bits

```
az extension add --source https://azuremlsdktestpypi.blob.core.windows.net/wheels/azureml-v2-cli-e2e-test/64774736/ml-0.0.64774736-py3-none-any.whl --pip-extra-index-urls https://azuremlsdktestpypi.azureedge.net/azureml-v2-cli-e2e-test/64774736 --yes 
```

















