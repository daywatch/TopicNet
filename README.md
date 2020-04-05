<h1 align="center">TopicNet: A Python NLP Library for Many Human Languages</h2>

<div align="center">
    <a href="https://pypi.org/project/topicnet">
        <img alt="PyPI Version" src="https://img.shields.io/pypi/v/topicnet?color=blue">
    </a>
    <a href="https://www.python.org/downloads/release/python-360/">
        <img alt="Python Version" src="https://img.shields.io/pypi/pyversions/TopicNet">
    </a>
    <a href="https://travis-ci.com/machine-intelligence-laboratory/TopicNet">
        <img alt="Travis Build Status" src="https://travis-ci.com/machine-intelligence-laboratory/TopicNet.svg?branch=master">
    </a>
    <a href="https://codecov.io/gh/machine-intelligence-laboratory/TopicNet">
        <img alt="Code Coverage" src="https://codecov.io/gh/machine-intelligence-laboratory/TopicNet/branch/master/graph/badge.svg">
    </a>
    <a href="https://github.com/machine-intelligence-laboratory/TopicNet/blob/master/LICENSE.txt">
        <img alt="License" src="https://img.shields.io/pypi/l/TopicNet?color=Black">
    </a>
    <a href="https://github.com/machine-intelligence-laboratory/TopicNet/commits/master">
        <img alt="GitHub last commit" src="https://img.shields.io/github/last-commit/machine-intelligence-laboratory/TopicNet">
    </a>
</div>


### What is TopicNet?

TopicNet is a high-level interface developed by [Machine Intelligence Laboratory](https://mipt.ai/en) for [BigARTM](https://github.com/bigartm/bigartm) library. 

```TopicNet```  library was created to assist in the task of building topic models. It aims at automating model training routine freeing more time for artistic process of constructing a target functional for the task at hand.

Consider using TopicNet if:

* you want to explore BigARTM functionality without writing an overhead.
* you need help with rapid solution prototyping.
* you want to build a good topic model quickly (out-of-box, with default parameters).
* you have an ARTM model at hand and you want to explore it's topics.

`TopicNet` provides an infrastructure for your prototyping (`Experiment` class) and helps to observe results of your actions via `viewers` module.

<p>
    <div align="center">
        <img src="./docs/readme_images/training_scheme_example.png" width="60%" alt/>
    </div>
    <em>
        Example of the two-stage experiment scheme.
        At the first stage, regularizer with parameter <img src="https://render.githubusercontent.com/render/math?math=\tau"> taking values in some range <img src="https://render.githubusercontent.com/render/math?math=\{\tau_1, \tau_2, \tau_3\}"> is applied.
        Best models after the first stage are <em>Model 1</em> and <em>Model 2</em> — so <em>Model 3</em> is not taking part in the training process anymore.
        The second stage is connected with another regularizer with parameter <img src="https://render.githubusercontent.com/render/math?math=\xi"> taking values in range <img src="https://render.githubusercontent.com/render/math?math=\{\xi_1, \xi_2\}">.
        As a result of this stage, two descendant models of <em>Model 1</em> and two descendant models of <em>Model 2</em> are obtained.
    </em>
</p>


### How to start?

Define `TopicModel` from an ARTM model at hand or with help from `model_constructor` module, where you can set models main parameters. Then create an `Experiment`, assigning a root position to this model and path to store your experiment. Further, you can define a set of training stages by the functionality provided by the `cooking_machine.cubes` module.

Further you can read documentation [here](https://machine-intelligence-laboratory.github.io/TopicNet/). Currently we are in the process of imporving it. 

## How to install TopicNet

**Core library functionality is based on BigARTM library** which required manual installation on all systems.  
Currently we have working solution for Linux users: 
```
pip install topicnet
```
as it is currently awailiable to install BigARTM on linux systems via `pip`. We hoping to bring `pip` installation support to other systems, hovewer right now you may find the following guide useful.

To avoid installing BigARTM you can use [docker images](https://hub.docker.com/r/xtonev/bigartm/tags) with preinstalled different versions of BigARTM library in them. 

#### Using docker image
```
docker pull xtonev/bigartm:v0.10.0
docker run -t -i xtonev/bigartm:v0.10.0
```
#### Check if import is sucessfull
```
python3
import artm
artm.version()
```

Alternatively, you can follow [BigARTM installation manual](https://bigartm.readthedocs.io/en/stable/installation/index.html).
After setting up the environment you can fork this repository or use ```pip install topicnet``` to install the library.  

## How to use TopicNet

Let's say you have a handful of raw text mined from some source and you want to perform some topic modelling on them. Where should you start? 
### Data Preparation
Every ML problem starts with data preprocess step. TopicNet does not perform data preprocessing itself. Instead, it demands data being prepared by the user and loaded via [Dataset class.](topicnet/cooking_machine/dataset.py)
Here is a basic example of how one can achieve that: [rtl_wiki_preprocessing](topicnet/demos/RTL-WIKI-PREPROCESSING.ipynb).

### Training topic model
Here we can finally get on the main part: making your own, best of them all, manually crafted Topic Model
#### Get your data
We need to load our data prepared previously with Dataset:
```
data = Dataset('/Wiki_raw_set/wiki_data.csv')
```
#### Make initial model
In case you want to start from a fresh model we suggest you use this code:
```
from topicnet.cooking_machine.model_constructor import init_simple_default_model

model_artm = init_simple_default_model(
    dataset=data,
    modalities_to_use={'@lemmatized': 1.0, '@bigram':0.5},
    main_modality='@lemmatized',
    specific_topics=14,
    background_topics=1,
)
```
Note that here we have model with two modalities: `'@lemmatized'` and `'@bigram'`.  
Further, if needed, one can define a custom score to be calculated during the model training.
```
from topicnet.cooking_machine.models.base_score import BaseScore

class CustomScore(BaseScore):
    def __init__(self):
        super().__init__()

    def call(self, model,
             eps=1e-5,
             n_specific_topics=14):
        phi = model.get_phi().values[:,:n_specific_topics]
        specific_sparsity = np.sum(phi < eps) / np.sum(phi < 1)
        return specific_sparsity
```
Now, `TopicModel` with custom score can be defined:
```
from topicnet.cooking_machine.models.topic_model import TopicModel

custom_score_dict = {'SpecificSparsity': CustomScore()}
tm = TopicModel(model_artm, model_id='Groot', custom_scores=custom_score_dict)
```
#### Define experiment
For further model training and tuning `Experiment` is necessary:
```
from topicnet.cooking_machine.experiment import Experiment
experiment = Experiment(experiment_id="simple_experiment", save_path="experiments", topic_model=tm)
```
#### Toy with the cubes
Defining a next stage of the model training to select a decorrelator parameter:
```
from topicnet.cooking_machine.cubes import RegularizersModifierCube

my_first_cube = RegularizersModifierCube(
    num_iter=5,
    tracked_score_function='PerplexityScore@lemmatized',
    regularizer_parameters={
        'regularizer': artm.DecorrelatorPhiRegularizer(name='decorrelation_phi', tau=1),
        'tau_grid': [0,1,2,3,4,5],
    },
    reg_search='grid',
    verbose=True
)
my_first_cube(tm, demo_data)
```
Selecting a model with best perplexity score:
```
perplexity_select = 'PerplexityScore@lemmatized -> min COLLECT 1'
best_model = experiment.select(perplexity_select)
```
#### View the results
Browsing the model is easy: create a viewer and call its `view()` method:
```
thresh = 1e-5
top_tok = TopTokensViewer(best_model, num_top_tokens=10, method='phi')
top_tok_html =  top_tok.to_html(top_tok.view(),thresh=thresh)
for line in first_model_html:
    display_html(line, raw=True)
```

## FAQ

#### In the example we used to write vw modality like **@modality**, is it a VowpallWabbit format?

It is a convention to write data designating modalities with @ sign taken by TopicNet from BigARTM.

#### CubeCreator helps to perform a grid search over initial model parameters. How can I do it with modalities?

Modality search space can be defined using standart library logic like:
```
class_ids_cube = CubeCreator(
    num_iter=5,
    parameters: [
        name: 'class_ids',
        values: {
        '@text': [1, 2, 3],
        '@ngrams': [4, 5, 6],
        },
    ]
    reg_search='grid',
    verbose=True
)

```
However for the case of modalities a couple of slightly more convenient methods are availiable:

```
parameters : [
    {
        'name': 'class_ids@text',
        'values': [1, 2, 3]
    },
    {
        'name': 'class_ids@ngrams',
        'values': [4, 5, 6]
    }
]
parameters:[
    {
        'class_ids@text'  : [1, 2, 3],
        'class_ids@ngrams': [4, 5, 6]
    }
]
```
