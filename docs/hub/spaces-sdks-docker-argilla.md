# Argilla on Spaces

**Argilla** is an open-source, data labelling tool, for highly efficient human-in-the-loop and MLOps workflows. Argilla is composed of (1) a server and webapp for data labelling, and curation, and (2) a Python library for building data annotation workflows in Python. Argilla nicely integrates with the Hugging Face stack (`datasets`, `transformers`, `hub`, and `setfit`), and now it can also be deployed using the Hub's Docker Spaces. 

Visit the [Argilla documentation](https://docs.argilla.io) to learn about its features and check out the [Deep Dive Guides](https://docs.argilla.io/en/latest/guides/guides.html) and [Tutorials](https://docs.argilla.io/en/latest/tutorials/tutorials.html).

In the next sections, you'll learn to deploy your own Argilla app and use it for data labelling workflows right from the Hub. The diagram below illustrates the complete process.

<div class="flex justify-center">
    <img src="https://www.argilla.io/blog/hf_space/how.svg"/>
</div>

## Your first Argilla Space

In this section, you'll learn to deploy an Argilla Space and use it for data annotation and training a sentiment classifier with [SetFit](https://github.com/huggingface/setfit/), an amazing few-shot learning library.

### Deploy Argilla on Spaces

You can deploy Argilla on Spaces with just a few clicks:

<a  href="https://huggingface.co/new-space?template=argilla/argilla-template-space">
    <img src="https://huggingface.co/datasets/huggingface/badges/raw/main/deploy-to-spaces-lg.svg" />
</a>

You need to define the **Owner** (your personal account or an organization), a **Space name**, and the **Visibility**. To interact with the Argilla app with Python, you need to setup the visibility to `Public`. If you plan to use the Space frequently or handle large datasets for data labeling and feedback collection, upgrading the hardware with a more powerful CPU and increased RAM can enhance performance.

<div class="flex justify-center">
<img src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hub/spaces-argilla-new-space.png"/>
</div>

<Tip>
If you want to customize the title, emojis, and colors of your space, go to "Files and Versions" and edit the metadata of your README.md file.
</Tip>

Once you have created the space, you'll see the `Building` status and once it becomes `Running` your space is ready to go. If you don't see the Argilla login UI refresh the page.

You'll see the login screen where you need to use either argilla or team with the default passwords (1234). Remember you can change the passwords using secret variables. If you get a 500 error when introducing the credentials, make sure you have correctly introduce the password.

<Tip>
For quick experimentation, you can jump directly into the next section. If you want to add access restrictions, go to the "Setting up secret environment variables" at the end of this document. Setting up secret variables is recommended for longer-term usage.
</Tip>

### How to upload data

Once your Argilla Space is running:

1. You need to find the Space Direct URL under the "Embed this Space" option (top right, see screenshot below).
2. This URL gives you access to a full-screen Argilla UI for data labelling. You will use this URL as the `api_url` parameter for connecting the argilla Python client.
3. You are now ready to upload your first dataset into Argilla.

<div class="flex justify-center">
<img src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hub/spaces-argilla-embed-space.png"/>
</div>

<Tip>
Argilla Datasets cannot be uploaded directly from the Argilla UI. Most Argilla users upload datasets programmatically using the argilla Python library but you can also use Argilla Data Manager, a simple Streamlit app.
</Tip>

For uploading Argilla datasets, there are two options:

1. Using `argilla` Python library inside Jupyter, Colab, VS Code, or other Python IDE. You can use `pandas` or `datasets` to read a file (`csv`, `json`, etc.), transform it into Argilla records and log them into Argilla. To do this, we recommend to read the [basics guide](https://docs.argilla.io/en/latest/guides/how_to.html). 
2. For quick experiments, you can use the [no-code data manager app](https://huggingface.co/spaces/argilla/data-manager) to upload your file and log it into Argilla. If you need to transform your dataset before uploading it into Argilla, we recommend the first option.

If you want to follow a complete tutorial with Colab or Jupyter, [check this tutorial](https://docs.argilla.io/en/latest/tutorials/notebooks/training-textclassification-setfit-fewshot.html). If you want a quick step-by-step example using the `argilla` Python library, keep reading.

First, you need to open a Python IDE, we highly recommend using Jupyter notebooks or Colab.

Second, you need to pip install `datasets` and `argilla` on Colab or your local machine:

```bash
pip install datasets argilla
```

Third, you can read the example dataset using the `datasets` library. This dataset is a CSV file uploaded to the Hub using the drag and drop feature. For reading other file types, check the [basics guide](https://docs.argilla.io/en/latest/guides/how_to.html).

```python
from datasets import load_dataset

dataset = load_dataset("dvilasuero/banking_app", split="train").shuffle()
```

Forth, you can upload your first dataset by logging it into Argilla using your Space Direct URL:

```python
import argilla as rg
from datasets import load_dataset

# You can find your Space URL behind the Embed this space button
rg.init(
    api_url="<https://your-direct-space-url.hf.space>", 
    api_key="team.apikey"
)

banking_ds = load_dataset("argilla/banking_sentiment_setfit", split="train")

# Argilla expects labels in the annotation column
# We include labels for demo purposes
banking_ds = banking_ds.rename_column("label", "annotation")

# Build argilla dataset from datasets
argilla_ds = rg.read_datasets(banking_ds, task="TextClassification")

# Create dataset
rg.log(argilla_ds, "bankingapp_sentiment")
```

Congrats! You now have a dataset available from the Argilla UI to start browsing and labelling. In the code above, we've used one of the many integrations with Hugging Face libraries, which let you [read hundreds of datasets](https://docs.argilla.io/en/latest/guides/features/datasets.html#Importing-a-Dataset) available on the Hub.

### How to train a model with labelled data

To train a model using your labeled data, you need to read the labelled dataset and prepare it for training:

```python
# this will read the dataset and turn it into a clean dataset for training
dataset = rg.load("bankingapp_sentiment").prepare_for_training()
```

To train a SetFit model with this dataset:

```python
from sentence_transformers.losses import CosineSimilarityLoss

from setfit import SetFitModel, SetFitTrainer

# Create train test split
dataset = dataset.train_test_split()

# Load SetFit model from Hub
model = SetFitModel.from_pretrained("sentence-transformers/paraphrase-mpnet-base-v2")

# Create trainer
trainer = SetFitTrainer(
    model=model,
    train_dataset=dataset["train"],
    eval_dataset=dataset["test"],
    loss_class=CosineSimilarityLoss,
    batch_size=8,
    num_iterations=20, 
)

# Train and evaluate
trainer.train()
metrics = trainer.evaluate()
```

Optionally, you can push the full dataset to the Hub for later use:

```python
# save full argilla dataset for reproducibility
rg.load("bankingapp_sentiment").to_datasets().push_to_hub("bankingapp_sentiment") 
```

As a next step, check out the [Argilla Tutorials](https://docs.argilla.io/en/latest/tutorials/tutorials.html) section. All the tutorials can be run using Colab or local Jupyter Notebooks, so you can start building datasets with Argilla and Spaces!

## Feedback and support

If you have improvement suggestions or need specific support, please join [Argilla Slack community](https://join.slack.com/t/rubrixworkspace/shared_invite/zt-whigkyjn-a3IUJLD7gDbTZ0rKlvcJ5g) or reach out on [Argilla's GitHub repository](https://github.com/argilla-io/argilla).

## Setting up secret environment variables

The Space template provides a way to set up different **optional settings** focusing on securing your Argilla Space.

To set up these secrets, you can go to the Settings tab on your created Space. Make sure to save these values somewhere for later use.

The template space has two users: `team` and `argilla`. The username `team` corresponds to the root user, who can upload datasets and access any workspace within your Argilla Space. The username `argilla` is a normal user with access to the `team` workspace and its own workspace called `argilla`. 

Currently, these user names can't be configured. The passwords and API keys to upload, read, update, and delete datasets can be configured using the following secrets:

- `ARGILLA_API_KEY`: Argilla provides a Python library to interact with the app (read, write, and update data, log model predictions, etc.). If you don't set this variable, the library and your app will use the default API key. If you want to secure your Space for reading and writing data, we recommend you to set up this variable. The API key you choose can be any string of your choice and you can check an online generator if you like.

- `ARGILLA_PASSWORD`: This sets a custom password for login into the app with the `argilla` username. The default password is `1234`. By setting up a custom password you can use your own password to login into the app.

- `TEAM_API_KEY`: This sets the root user's API key. The API key you choose can be any string of your choice and you can check an online generator if you like.

- `TEAM_PASSWORD`: This sets a custom password for login into the app with the `team` username. The default password is `1234`. By setting up a custom password you can use your own password to login into the app.

The combination of these secret variables gives you the following setup options:

1. *I want to avoid that anyone without the API keys can add, delete, or update datasets using the Python client*: You need to setup `ARGILLA_API_KEY` and `TEAM_API_KEY`. 
2. *Additionally, I want to avoid that the `argilla` username can delete datasets from the UI*: You need to setup `TEAM_PASSWORD` and use `TEAM_API_KEY` with the Python Client. This option might be interesting if you want to control dataset management but want anyone to browse your datasets using the `argilla` user.
3. *Additionally, I want to avoid that anyone without password can browse my datasets with the `argilla` user*: You need to setup `ARGILLA_PASSWORD`. In this case, you can use `ARGILLA_API_KEY` and/or `TEAM_API_KEY` with the Python Client depending on your needs for dataset deletion rights.

Additionally, the `LOAD_DATASETS` will let you configure the sample datasets that will be pre-loaded. The default value is `single` and the supported values for this variable are:
    1. `single`: Load single datasets for TextClassification task.
    2. `full`: Load all the sample datasets for NLP tasks (TokenClassification, TextClassification, Text2Text)
    3. `none`: No datasets being loaded.
