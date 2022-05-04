# DVC ML Ops Example
This repository contains guidelines and recommendations how to use DVC from iterative.ai for MLOps.

The major goal of implementation of MLOPs tools is to make experimenting with models reliable and traceable. More precisely, it means that for each resulted model information on every step, e.g. data import, feature selection, model training, are documented and can be restored at any time. Moreover, pre-defined metrics are calculated automatically and compared to ones of the previous model state. The overview table with paramters and calculated metrics is then generated automatically.

## Workflows

## Initialisation and setting up
(1) Each step of the pipeline should be a separate python script (not jupyter notebook), e.g. data import, data cleaning, data preprocessing, data split, model training etc.
The resulted project structure might look as follows.

    .
    ├── cache                   # Intermediate results (in gitignore)
    ├── data                    # Data used for model training (in gitignore)
    ├── data_processing         # Data import, preprocessing and split scripts
    ├── data_exploration        # Jupyter notebook scripts, plots, images with data exploration
    ├── training                # Scripts to train model, model architecture as well as training results (including training history and resulted model)
    ├── hpo                     # Scripts and results of hyperparameter optimisation as well as e.g. report of hpo
    ├── report                  # Python script that generates the report, calculate metrics, create plots and images with them. Might also contain a jupiter notebook.
    ├── LICENSE
    ├── README.md               # Problem/Model/Repository description
    ├── params.yaml             # Training, model and other parameters that need to be changed during the experimentation routinr
    └── metrics.json            # Metrics' values in a machine-readable format
    
Note 1: In case of model learning where data is imported from an online source, e.g. Dataverse, please consider using versioning of datasets and specify them in the script with data import. Then each time another version of the dataset is used, it must be specified explicitly in the python script where data import happens.    
   
Note 2: Please notice that Data exploration and Hyper-parameter optimisation steps are mostly done manually by the data scientists. Therefore, they are not included in the pipeline and might have a form of Jupyter Notebooks or other interactive tools.

(2) Intermediate results after each step is recommended to store in a separate file, e.g. pickle or joblib. This allows to re-use results of single steps in case the previous steps did not change.
Then python script for each step should end, for example, with:
```
pathlib.Path('cache').mkdir(parents=True, exist_ok=True)
save_path = os.path.join('cache', 'read_data.pkl')
df.to_pickle(save_path)
```
where df is a variable with intermediate results that will be needed in the next step. In this example an entire pandas dataframe with loaded data is cached.
To load intermediate results from the previous step following code can be used:
```
import_path = os.path.join('cache', 'read_data.pkl')
df = pd.read_pickle(import_path)
```
The imported data will be then present in the variable named 'df'.

(3) Create a dvc pipeline that runs a set of pre-defined steps.

First, dvc must be installed and initialised on the computer. The easiest way to install it is just to run:
```
pip install dvc
``` 
More information https://dvc.org/doc/install

To initialise dvc you need to run
```
dvc init
```
Then create a yaml-file named 'dvc.yaml' in the root of the code. In this file a pipeline must be defined. Here is an example:
```
stages:
  get_data:
    cmd: python ./src/read_data.py
    deps:
    - ./src/read_data.py
    - ./res/train.csv
    outs:
    - ./cache/read_data.pkl
  process_data:
    cmd:
      - python ./src/preprocessing.py
      - python ./src/feature_selection.py
      - python ./src/dataset_split.py
    deps:
      - ./cache/read_data.pkl
      - ./src/preprocessing.py
      - ./src/feature_selection.py
      - ./src/dataset_split.py
    outs:
      - ./cache/preprocessing.pkl
      - ./cache/feature_selection.pkl
      - ./cache/dataset_train.pkl
      - ./cache/dataset_test.pkl
  train:
    cmd: python ./src/model_training.py
    deps:
      - ./cache/dataset_train.pkl
      - ./cache/dataset_test.pkl
      - ./src/model_training.py
    outs:
      - ./cache/trained_model.joblib
  report:
    cmd: python ./src/report.py
    deps:
      - ./cache/trained_model.joblib
      - ./src/report.py
    outs:
      - ./cache/confusion_matrix.png
    metrics:
      - ./cache/metrics.json:
          cache: false
```

Note 1: Python scripts must be also be listed as dependencies.

To re-run the pipeline locally after any changes in the code execute the following:
```
dvc repro
```
In case you want DVC to rerun all stages even if there are no changes, use:
```
dvc repro -f
```

The main benefit of using dvc repro is, if there are no changes in the previous steps, then the input data for the next one will be taken from cache (intermediate results). Thus, experimenting will be in general faster.

To output metrics and compare them with metrics' values from another commit locally, use
```
dvc metrics show
```
and
```
dvc metrics diff
```
More information and further features are here: https://dvc.org/doc/command-reference/

### ML experimenting
(1) If you want to try something completely different (e.g. another model architecture), it is recommended to create a new branch. For alteration of model parameters or usage of new preprocessing steps or extended dataset, a new commit would be enough. But you can also create a new branch even for parameter changes.

(2) After changes you need to add new intermediate files to dvc tracker by means of:
```
dvc add folder_or_file_name
```

(3) If dvc pipeline requires to be updated, update the file dvc.yaml correspondingly.

(4) Run dvc pipeline:
```
dvc repro
```

(5) To push intermediate files onto remote, run:
```
dvc push
```
This command will copy your selected files onto dvc remote and from now on you are able to track changes and pull associated versions of files in other commits or even from another computer. However, this computer must have an access to the dvc repository.
Please run this command every time you have a self-containing state, e.g. after a new model is trained and now you want to try another architecture in another branch.

(6) Commit your changes via git. By doing this, git information on linked dvc repository and remotes will be also updated.

Now you are able to compare models between different commits via comparison of metrics.json and report.md files. You can also use DVC Studio for these purposes.


### Something

It is crucial to understand at every point of time for every model, which dataset, preprocessing steps, model parameters were used to get this very model.
That means that so-called ml experiments are to track and resulted models are to associate with certain parameters set.
(1) To run different experiements first spicify parameters you would like to vary. The default file for parameters is params.yaml
Example of parameter file:
```
train:
  optimizer: adam
  batch_size: 125
model:
  no_neurons: 20
  no_layers: 3
```

(2) Integrate the parameters from this file into your code, for example as follows:
```
    with open('params.yaml', 'r') as stream:
        params = yaml.safe_load(stream)
    batch_size = params['training']['batch_size']
```

(3) More information on how to get overview of run experiments, design new experiment without changing the file, queue them and run in parallel can be found here: https://dvc.org/doc/start/experiments

Note 1: It is recommended that every model architecture has its own git branch where results and code are saved. For small changes, commits can be used. For example if you are training a classifier, a classifier based on SVM must be in a separate branch, while different c-values might be stored as commits.

### Continuous machine learning
In order to automate single steps of the pipeline the framework CML from iterative.ai is recommended to use: https://github.com/iterative/cml
This tool allows defining pipelines that will be run by means of Github Actions automatically after any change is pushed on the repository.

In general, a cml pipeline is a set of steps that will be executed automatically after any change in the model and namely from begin till the end of the pipeline. Moreover, it happens in a virtual enviroment (docker-container) ensuring a totally clear execution. Please consider that docker-container provided by iterative.ai has Python 3.6 installed. That means that the code should be also written for the python interpreter of the version 3.6.

In order to enable automated execution, a corresponding github workflow (e.g. Github Action) must be added. The simplest workflow might look like this:
```
name: cml example
on: [push]
jobs:
  run:
    runs-on: [ubuntu-latest]
    container: docker://dvcorg/cml-py3:latest
    steps:
      - uses: actions/checkout@v2
      - name: cml_run
        env:
          repo_token: ${{ secrets.GITHUB_TOKEN }}
        run: |
          pip install -r requirements.txt
          dvc repro
          git fetch --prune
          dvc metrics diff --show-md main > report.md
          # Add figure
          echo "## Confusion matrix"
          cml-publish confusion_matrix.png --md >>report.md
          cml-send-comment report.md
```
If no execution errors occur, your commit will be marked as "passed". After execution the pushed commit will be provided with a comment to the commit. In this comment there will be a report specified in the last part of the cml pipeline. The last part can be modified in a way when further elements in the report (e.g. figures, tables etc.) can be added.

You can get additional information on the run by selecting it on the tab "Actions".

An example can be found here: https://github.com/vkhaydarov/mlops_training

### Dataset upload
(1) Upload the dataset on the Dataset repository according to the guidelines above or
(2) Upload the dataset onto the ML workstation according the following folder structure: 01_datasheet, 02_data, 03_raw_data and 04_data_cleaning_scripts or 
(3) Prepare the dataset locally according the following folder structure: 01_datasheet, 02_data, 03_raw_data and 04_data_cleaning_scripts


### Self-hosted runner
In case a large dataset or when training performance of the essence it is recommended to use self-hosted runner.
The runner might be deployed on your local computer or on the ML workstation of the Process-to-Order Lab.
For the second option please contact vkhaydarov.
More information is here https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners


### Setting up the repository
Pre-requestives: Dataset is on Dataverse or locally stored

(1) Create a git respository for the problem, e.g. on Github

(2) Create at least two braches: main and develop. Use main only for a model that is designed to be in production or deployed.
Development branch is for a model that you currently work on.

(3) Write your code.

(4) On your local machine initialise dvc by means of running this command in the terminal or command line
```
dvc init
```

(5) Specify the data pipeline in dvc.yaml file.

(6) Specify parameters in params.yaml.

(7) Test your pipeline if everything works as intended and without error:
```
dvc repro
```

(8) Before commiting your changes, consider carefully which files must be added to gitignore. Usually it must be your folder with intermediate results and dvc cache folder. You do not want to store your initial data, intermediate results, cache and models on github. Therefore, add them to gitignore.

### ML experimenting
(1) If you want to try something completely different (e.g. another model architecture), it is recommended to create a new branch. For alteration of model parameters or usage of new preprocessing steps or extended dataset, a new commit would be enough. But you can also create a new branch even for parameter changes.

(2) After changes you need to add new intermediate files to dvc tracker by means of:
```
dvc add folder_or_file_name
```

(3) If dvc pipeline requires to be updated, update the file dvc.yaml correspondingly.

(4) Run dvc pipeline:
```
dvc repro
```

(5) To push intermediate files onto remote, run:
```
dvc push
```
This command will copy your selected files onto dvc remote and from now on you are able to track changes and pull associated versions of files in other commits or even from another computer. However, this computer must have an access to the dvc repository.
Please run this command every time you have a self-containing state, e.g. after a new model is trained and now you want to try another architecture in another branch.

(6) Commit your changes via git. By doing this, git information on linked dvc repository and remotes will be also updated.

Now you are able to compare models between different commits via comparison of metrics.json and report.md files. You can also use DVC Studio for these purposes.

### Model deployment
The main git branch might be thought as a model deployed in production or, in other words, a tested and validated model, which performs the best.
Usually, after ML experimenting you select such a model from your working versions and via a pull request merge it to the main branch. To do this please use Pull request functionality of GitHub. Here, by means of direct comparison of metrics.json and report.md files you can decide whether it is worth it to replace the model.

### Continous machine learning (optional)
If you want to use CML then you need to create another file that contains cml pipeline. 
If using GitHub, then create the cml pipeline and place it in './.github/workflows/cml.yml'
If you are using a dataset stored locally (e.g. large image dataset), please skip this.
