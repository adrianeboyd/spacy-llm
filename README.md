<a href="https://explosion.ai"><img src="https://explosion.ai/assets/img/logo.svg" width="125" height="125" align="right" /></a>

# spacy-llm: Integrating LLMs into structured NLP pipelines

![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/explosion/spacy-llm/external.yml)
[![pypi Version](https://img.shields.io/pypi/v/spacy-llm.svg?style=flat-square&logo=pypi&logoColor=white)](https://pypi.org/project/spacy-llm/)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg?style=flat-square)](https://github.com/ambv/black)

This package integrates Large Language Models (LLMs) into [spaCy](https://spacy.io), featuring a modular system for **fast prototyping** and **prompting**, and turning unstructured responses into **robust outputs** for various NLP tasks, **no training data** required.

- Serializable `llm` **component** to integrate prompts into your pipeline
- **Modular functions** to define the [**task**](#Tasks) (prompting and parsing) and [**backend**](#Backends) (model to use)
- Support for **hosted APIs** and self-hosted **open-source models**
- Integration with [`MiniChain`](https://github.com/srush/MiniChain) and [`LangChain`](https://github.com/hwchase17/langchain)
- Access to **[OpenAI API](https://platform.openai.com/docs/api-reference/introduction)**, including GPT-4 and various GPT-3 models
- Built-in support for **open-source [Dolly](https://huggingface.co/databricks)** models hosted on Hugging Face
- Usage examples for **Named Entity Recognition** and **Text Classification**
- Easy implementation of **your own functions** via [spaCy's registry](https://spacy.io/api/top-level#registry) for custom prompting, parsing and model integrations

## 🧠 Motivation

Large Language Models (LLMs) feature powerful natural language understanding capabilities. With only a few (and sometimes no) examples, an LLM can be prompted to perform custom NLP tasks such as text categorization, named entity recognition, coreference resolution, information extraction and more.

[spaCy](https://spacy.io) is a well-established library for building systems that need to work with language in various ways. spaCy's built-in components are generally powered by supervised learning or rule-based approaches.

Supervised learning is much worse than LLM prompting for prototyping, but for many tasks it's much better for production. A transformer model that runs comfortably on a single GPU is extremely powerful, and it's likely to be a better choice for any task for which you have a well-defined output. You train the model with anything from a few hundred to a few thousand labelled examples, and it will learn to do exactly that. Efficiency, reliability and control are all better with supervised learning, and accuracy will generally be higher than LLM prompting as well.

`spacy-llm` lets you have **the best of both worlds**. You can quickly initialize a pipeline with components powered by LLM prompts, and freely mix in components powered by other approaches. As your project progresses, you can look at replacing some or all of the LLM-powered components as you require.

Of course, there can be components in your system for which the power of an LLM is fully justified. If you want a system that can synthesize information from multiple documents in subtle ways and generate a nuanced summary for you, bigger is better. However, even if your production system needs an LLM for some of the task, that doesn't mean you need an LLM for all of it. Maybe you want to use a cheap text classification model to help you find the texts to summarize, or maybe you want to add a rule-based system to sanity check the output of the summary. These before-and-after tasks are much easier with a mature and well-thought-out library, which is exactly what spaCy provides.

## ⏳ Install

`spacy-llm` will be installed automatically in future spaCy versions. For now, you can run the following in the same virtual environment where you already have `spacy` [installed](https://spacy.io/usage).

```bash
python -m pip install spacy-llm
```

> ⚠️ This package is still experimental and it is possible that changes made to the interface will be breaking in minor version updates.

## 🐍 Usage

The task and the backend have to be supplied to the `llm` pipeline component using [spaCy's config
system](https://spacy.io/api/data-formats#config). This package provides various built-in
functionality, as detailed in the [API](#-api) documentation.

### Example 1: Add a text classifier using a GPT-3 model from OpenAI

Create a new API key from openai.com or fetch an existing one, and ensure the keys are set as environmental variables.
For more background information, see the [OpenAI](#OpenAI) section.

Create a config file `config.cfg` containing at least the following
(or see the full example [here](usage_examples/textcat_openai)):

```ini
[nlp]
lang = "en"
pipeline = ["llm"]

[components]

[components.llm]
factory = "llm"

[components.llm.task]
@llm_tasks = "spacy.TextCat.v1"
labels = COMPLIMENT,INSULT

[components.llm.backend]
@llm_backends = "spacy.REST.v1"
api = "OpenAI"
config = {"model": "text-davinci-003", "temperature": 0.3}
```

Now run:

```python
from spacy import util

config = util.load_config("config.cfg")
nlp = util.load_model_from_config(config, auto_fill=True)
doc = nlp("You look gorgeous!")
print(doc.cats)
```

### Example 2: Add NER using an open-source model through Hugging Face

To run this example, ensure that you have a GPU enabled, and `transformers`, `torch` and CUDA installed.
For more background information, see the [DollyHF](#spacydollyhfv1) section.

Create a config file `config.cfg` containing at least the following
(or see the full example [here](usage_examples/ner_dolly)):

```ini
[nlp]
lang = "en"
pipeline = ["llm"]

[components]

[components.llm]
factory = "llm"

[components.llm.task]
@llm_tasks = "spacy.NER.v1"
labels = PERSON,ORGANISATION,LOCATION

[components.llm.backend]
@llm_backends = "spacy.DollyHF.v1"
# For better performance, use databricks/dolly-v2-12b instead
model = "databricks/dolly-v2-3b"
```

Now run:

```python
from spacy import util

config = util.load_config("config.cfg")
nlp = util.load_model_from_config(config, auto_fill=True)
doc = nlp("Jack and Jill rode up the hill in Les Deux Alpes")
print([(ent.text, ent.label_) for ent in doc.ents])
```

Note that Hugging Face will download the `"databricks/dolly-v2-3b"` model the first time you use it. You can
[define the cached directory](https://huggingface.co/docs/huggingface_hub/main/en/guides/manage-cache)
by setting the environmental variable `HF_HOME`.
Also, you can upgrade the model to be `"databricks/dolly-v2-12b"` for better performance.

### Example 3: Create the component directly in Python

The `llm` component behaves as any other spaCy component does, so adding it to an existing pipeline follows the same
pattern:

```python
import spacy

nlp = spacy.blank("en")
nlp.add_pipe(
    "llm",
    config={
        "task": {
            "@llm_tasks": "spacy.NER.v1",
            "labels": "PERSON,ORGANISATION,LOCATION"
        },
        "backend": {
            "@llm_backends": "spacy.REST.v1",
            "api": "OpenAI",
            "config": {"model": "text-davinci-003"},
        },
    },
)
doc = nlp("Jack and Jill rode up the hill in Les Deux Alpes")
print([(ent.text, ent.label_) for ent in doc.ents])
```

Note that for efficient usage of resources, typically you would use [`nlp.pipe(docs)`](https://spacy.io/api/language#pipe)
with a batch, instead of calling `nlp(doc)` with a single document.

### Example 4: Implement your own custom task

To write a
[`task`](#tasks), you
need to implement two functions: `generate_prompts` that takes a list of spaCy [`Doc`](https://spacy.io/api/doc) objects and transforms
them into a list of prompts, and `parse_responses` that transforms the LLM outputs into annotations on the [`Doc`](https://spacy.io/api/doc), e.g. entity spans, text categories and more.

> 📖 For more details, see the [**usage example on writing your own task**](usage_examples/README.md#writing-your-own-task)

```python
from spacy_llm.registry import registry

@registry.llm_tasks("spacy.MyTask.v1")
class MyTask:
    def __init__(self, labels: str):
        ...

    def generate_prompts(self, docs: Iterable[Doc]) -> Iterable[str]:
        ...

    def parse_responses(
        self, docs: Iterable[Doc], responses: Iterable[str]
    ) -> Iterable[Doc]:
        ...
```

```ini
# config.cfg (excerpt)
[components.llm.task]
@llm_tasks = "spacy.MyTask.v1"
labels = LABEL1,LABEL2,LABEL3
```

## 📓 API

Each `llm` component is defined by two main settings:

- A [**task**](#Tasks), defining the prompt to send to the LLM as well as the functionality to parse the resulting response
  back into structured fields on spaCy's [Doc](https://spacy.io/api/doc) objects.
- A [**backend**](#Backends) defining the model to use and how to connect to it. Note that `spacy-llm` supports both access to external
  APIs (such as OpenAI) as well as access to self-hosted open-source LLMs (such as using Dolly through Hugging Face).

### Tasks

A _task_ defines an NLP problem or question, that will be sent to the LLM via a prompt. Further, the task defines
how to parse the LLM's responses back into structured information. All tasks are registered in spaCy's `llm_tasks` registry.

Practically speaking, a task should adhere to the `Protocol` `LLMTask` defined in [`ty.py`](spacy_llm/ty.py).
It needs to define a `generate_prompts` function and a `parse_responses` function.

#### <kbd>function</kbd> `task.generate_prompts`

Takes a collection of documents, and returns a collection of "prompts", which can be of type `Any`.
Often, prompts are of type `str` - but this is not enforced to allow for maximum flexibility in the framework.

| Argument    | Type            | Description            |
| ----------- | --------------- | ---------------------- |
| `docs`      | `Iterable[Doc]` | The input documents.   |
| **RETURNS** | `Iterable[Any]` | The generated prompts. |

#### <kbd>function</kbd> `task.parse_responses`

Takes a collection of LLM responses and the original documents, parses the responses into structured information,
and sets the annotations on the documents. The `parse_responses` function is free to set the annotations in any way,
including `Doc` fields like `ents`, `spans` or `cats`, or using custom defined fields.

The `responses` are of type `Iterable[Any]`, though they will often be `str` objects. This depends on the
return type of the [backend](#backends).

| Argument    | Type            | Description              |
| ----------- | --------------- | ------------------------ |
| `docs`      | `Iterable[Doc]` | The input documents.     |
| `responses` | `Iterable[Any]` | The generated prompts.   |
| **RETURNS** | `Iterable[Doc]` | The annotated documents. |

#### spacy.NER.v1

The built-in NER task supports both zero-shot and few-shot prompting.

```ini
[components.llm.task]
@llm_tasks = "spacy.NER.v1"
labels = PERSON,ORGANISATION,LOCATION
examples = null
```

| Argument                  | Type                                    | Default      | Description                                                                                                                                  |
| ------------------------- | --------------------------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `labels`                  | `str`                                   |              | Comma-separated list of labels.                                                                                                              |
| `examples`                | `Optional[Callable[[], Iterable[Any]]]` | `None`       | Optional function that generates examples for few-shot learning.                                                                             |
| `normalizer`              | `Optional[Callable[[str], str]]`        | `None`       | Function that normalizes the labels as returned by the LLM. If `None`, defaults to `spacy.LowercaseNormalizer.v1`.                           |
| `alignment_mode`          | `str`                                   | `"contract"` | Alignment mode in case the LLM returns entities that do not align with token boundaries. Options are `"strict"`, `"contract"` or `"expand"`. |
| `case_sensitive_matching` | `bool`                                  | `False`      | Whether to search without case sensitivity.                                                                                                  |
| `single_match`            | `bool`                                  | `False`      | Whether to match an entity in the LLM's response only once (the first hit) or multiple times.                                                |

The NER task implementation doesn't currently ask the LLM for specific offsets, but simply expects a list of strings that represent the enties in the document.
This means that a form of string matching is required. This can be configured by the following parameters:

- The `single_match` parameter is typically set to `False` to allow for multiple matches. For instance, the response from the LLM might only mention the entity "Paris" once, but you'd still
  want to mark it every time it occurs in the document.
- The case-sensitive matching is typically set to `False` to be robust against case variances in the LLM's output.
- The `alignment_mode` argument is used to match entities as returned by the LLM to the tokens from the original `Doc` - specifically it's used as argument
  in the call to [`doc.char_span()`](https://spacy.io/api/doc#char_span). The `"strict"` mode will only keep spans that strictly adhere to the given token boundaries.
  `"contract"` will only keep those tokens that are fully within the given range, e.g. reducing `"New Y"` to `"New"`.
  Finally, `"expand"` will expand the span to the next token boundaries, e.g. expanding `"New Y"` out to `"New York"`.

To perform few-shot learning, you can write down a few examples in a separate file, and provide these to be injected into the prompt to the LLM.
The default reader `spacy.FewShotReader.v1` supports `.yml`, `.yaml`, `.json` and `.jsonl`.

```yaml
- text: Jack and Jill went up the hill.
  entities:
    PERSON:
      - Jack
      - Jill
    LOCATION:
      - hill
- text: Jack fell down and broke his crown.
  entities:
    PERSON:
      - Jack
```

```ini
[components.llm.task]
@llm_tasks = "spacy.NER.v1"
labels = PERSON,ORGANISATION,LOCATION
[components.llm.task.examples]
@misc = "spacy.FewShotReader.v1"
path = "ner_examples.yml"
```

#### spacy.TextCat.v1

The built-in TextCat task supports both zero-shot and few-shot prompting.

```ini
[components.llm.task]
@llm_tasks = "spacy.TextCat.v1"
labels = COMPLIMENT,INSULT
examples = null
```

| Argument            | Type                                    | Default | Description                                                                                                                                      |
| ------------------- | --------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `labels`            | str                                     |         | Comma-separated list of labels.                                                                                                                  |
| `examples`          | `Optional[Callable[[], Iterable[Any]]]` | `None`  | Optional function that generates examples for few-shot learning.                                                                                 |
| `normalizer`        | `Optional[Callable[[str], str]]`        | `None`  | Function that normalizes the labels as returned by the LLM. If `None`, falls back to `spacy.LowercaseNormalizer.v1`.                             |
| `exclusive_classes` | `bool`                                  | `False` | If set to `True`, only one label per document should be valid. If set to `False`, one document can have multiple labels.                         |
| `allow_none`        | `bool`                                  | `True`  | When set to `True`, allows the LLM to not return any of the given label. The resulting dict in `doc.cats` will have `0.0` scores for all labels. |
| `verbose`           | `bool`                                  | `False` | If set to `True`, warnings will be generated when the LLM returns invalid responses.                                                             |

To perform few-shot learning, you can write down a few examples in a separate file, and provide these to be injected into the prompt to the LLM.
The default reader `spacy.FewShotReader.v1` supports `.yml`, `.yaml`, `.json` and `.jsonl`.

```json
[
  {
    "text": "You look great!",
    "answer": "Compliment"
  },
  {
    "text": "You are not very clever at all.",
    "answer": "Insult"
  }
]
```

```ini
[components.llm.task]
@llm_tasks = "spacy.TextCat.v1"
labels = COMPLIMENT,INSULT
[components.llm.task.examples]
@misc = "spacy.FewShotReader.v1"
path = "textcat_examples.json"
```

#### spacy.NoOp.v1

This task is only useful for testing - it tells the LLM to do nothing, and does not set any fields on the `docs`.

```ini
[components.llm.task]
@llm_tasks = "spacy.NoOp.v1"
```

### Backends

A _backend_ defines which LLM model to query, and how to query it. It can be a simple function taking a collection
of prompts (consistent with the output type of `task.generate_prompts()`) and returning a collection of responses
(consistent with the expected input of `parse_responses`). Generally speaking, it's a function of type `Callable[[Iterable[Any]], Iterable[Any]]`,
but specific implementations can have other signatures, like `Callable[[Iterable[str]], Iterable[str]]`.

All built-in backends are registered in `llm_backends`. If no backend is specified, the repo currently connects to the [`OpenAI` API](#openai) by default,
using the built-in REST protocol, and accesses the `"text-davinci-003"` model.

#### OpenAI

When the backend uses OpenAI, you have to get an API key from openai.com, and ensure that the keys are set as
environmental variables:

```shell
export OPENAI_API_KEY="sk-..."
export OPENAI_API_ORG="org-..."
```

#### spacy.REST.v1

This default backend uses `requests` and a simple retry mechanism to access an API.

```ini
[components.llm.backend]
@llm_backends = "spacy.REST.v1"
api = "OpenAI"
config = {"model": "text-davinci-003", "temperature": 0.3}
```

| Argument    | Type             | Default | Description                                                                                                          |
| ----------- | ---------------- | ------- | -------------------------------------------------------------------------------------------------------------------- |
| `api`       | `str`            |         | The name of a supported API. In v.0.1.0, only "OpenAI" is supported.                                                 |
| `config`    | `Dict[Any, Any]` | `{}`    | Further configuration passed on to the backend.                                                                      |
| `strict`    | `bool`           | `True`  | If `True`, raises an error if the LLM API returns a malformed response. Otherwise, return the error responses as is. |
| `max_tries` | `int`            | `3`     | Max. number of tries for API request.                                                                                |
| `timeout`   | `int`            | `30`    | Timeout for API request in seconds.                                                                                  |

When `api` is set to `OpenAI`, the following settings can be defined in the `config` dictionary:

- `model`: one of the following list of supported models:
  - `"gpt-4"`
  - `"gpt-4-0314"`
  - `"gpt-4-32k"`
  - `"gpt-4-32k-0314"`
  - `"gpt-3.5-turbo"`
  - `"gpt-3.5-turbo-0301"`
  - `"text-davinci-003"`
  - `"text-davinci-002"`
  - `"text-curie-001"`
  - `"text-babbage-001"`
  - `"text-ada-001"`
  - `"davinci"`
  - `"curie"`
  - `"babbage"`
  - `"ada"`
- `url`: By default, this is `https://api.openai.com/v1/completions`. For models requiring the chat endpoint, use `https://api.openai.com/v1/chat/completions`.

#### spacy.MiniChain.v1

To use [MiniChain](https://github.com/srush/MiniChain) for the API retrieval part, make sure you have installed it first:

```shell
python -m pip install "minichain>=0.3,<0.4"
```

Note that MiniChain currently only supports Python 3.8, 3.9 and 3.10.

Example config blocks:

```ini
[components.llm.backend]
@llm_backends = "spacy.MiniChain.v1"
api = "OpenAI"

[components.llm.backend.query]
@llm_queries = "spacy.RunMiniChain.v1"
```

| Argument | Type                                                                              | Default | Description                                                                         |
| -------- | --------------------------------------------------------------------------------- | ------- | ----------------------------------------------------------------------------------- |
| `api`    | `str`                                                                             |         | The name of an API supported by MiniChain, e.g. "OpenAI".                           |
| `config` | `Dict[Any, Any]`                                                                  | `{}`    | Further configuration passed on to the backend.                                     |
| `query`  | `Optional[Callable[["minichain.backend.Backend", Iterable[str]], Iterable[str]]]` | `None`  | Function that executes the prompts. If `None`, defaults to `spacy.RunMiniChain.v1`. |

The default `query` (`spacy.RunMiniChain.v1`) executes the prompts by running `model(text).run()` for each given textual prompt.

#### spacy.LangChain.v1

To use [LangChain](https://github.com/hwchase17/langchain) for the API retrieval part, make sure you have installed it first:

```shell
python -m pip install "langchain>=0.0.144,<0.1"
```

Note that LangChain currently only supports Python 3.9 and beyond.

Example config block:

```ini
[components.llm.backend]
@llm_backends = "spacy.LangChain.v1"
api = "OpenAI"
query = {"@llm_queries": "spacy.CallLangChain.v1"}
config = {"temperature": 0.3}
```

| Argument | Type                                                                           | Default | Description                                                                          |
| -------- | ------------------------------------------------------------------------------ | ------- | ------------------------------------------------------------------------------------ |
| `api`    | `str`                                                                          |         | The name of an API supported by LangChain, e.g. "OpenAI".                            |
| `config` | `Dict[Any, Any]`                                                               | `{}`    | Further configuration passed on to the backend.                                      |
| `query`  | `Optional[Callable[["langchain.llms.BaseLLM", Iterable[Any]], Iterable[Any]]]` | `None`  | Function that executes the prompts. If `None`, defaults to `spacy.CallLangChain.v1`. |

The default `query` (`spacy.CallLangChain.v1`) executes the prompts by running `model(text)` for each given textual prompt.

#### spacy.DollyHF.v1

To use this backend, ideally you have a GPU enabled and have installed `transformers`, `torch` and CUDA in your virtual environment.
This allows you to have the setting `device=cuda:0` in your config, which ensures that the model is loaded entirely on the GPU (and fails otherwise).

```shell
python -m pip install "cupy-cuda11x"
python -m pip install "torch>=1.13.1,<2.0"
python -m pip install "transformers>=4.28.1,<5.0"
```

If you don't have access to a GPU, you can install `accelerate` and set`device_map=auto` instead, but be aware that this may result in some layers getting distributed to the CPU or even the hard drive,
which may ultimately result in extremely slow queries.

```shell
python -m pip install "accelerate>=0.16.0,<1.0"
```

Example config block:

```ini
[components.llm.backend]
@llm_backends = "spacy.DollyHF.v1"
model = "databricks/dolly-v2-3b"
```

| Argument | Type             | Default | Description                                                                                      |
| -------- | ---------------- | ------- | ------------------------------------------------------------------------------------------------ |
| `model`  | `str`            |         | The name of a Dolly model that is supported.                                                     |
| `config` | `Dict[Any, Any]` | `{}`    | Further configuration passed on to the construction of the model with `transformers.pipeline()`. |

Supported models (see the [Databricks models page](https://huggingface.co/databricks) on Hugging Face for details):

- `"databricks/dolly-v2-3b"`
- `"databricks/dolly-v2-7b"`
- `"databricks/dolly-v2-12b"`

Note that Hugging Face will download this model the first time you use it - you can
[define the cached directory](https://huggingface.co/docs/huggingface_hub/main/en/guides/manage-cache)
by setting the environmental variable `HF_HOME`.

### Various functions

#### spacy.FewShotReader.v1

This function is registered in spaCy's `misc` registry, and reads in examples from a `.yml`, `.yaml`, `.json` or `.jsonl` file.
It uses [`srsly`](https://github.com/explosion/srsly) to read in these files and parses them depending on the file extension.

```ini
[components.llm.task.examples]
@misc = "spacy.FewShotReader.v1"
path = "ner_examples.yml"
```

| Argument | Type               | Description                                                                |
| -------- | ------------------ | -------------------------------------------------------------------------- |
| `path`   | `Union[str, Path]` | Path to an examples file with suffix `.yml`, `.yaml`, `.json` or `.jsonl`. |

#### Normalizer functions

These functions provide simple normalizations for string comparisons, e.g. between a list of specified labels
and a label given in the raw text of the LLM response. They are registered in spaCy's `misc` registry
and have the signature `Callable[[str], str]`.

- `spacy.StripNormalizer.v1`: only apply `text.strip()`
- `spacy.LowercaseNormalizer.v1`: applies `text.strip().lower()` to compare strings in a case-insensitive way.

## 🚀 Ongoing work

In the near future, we will

- Add more example tasks
- Support a broader range of models
- Provide more example use-cases and tutorials
- Make the built-in tasks easier to customize via Jinja templates to define the instructions & examples

PRs are always welcome!

## 📝️ Reporting issues

If you have questions regarding the usage of `spacy-llm`, or want to give us feedback after giving it a spin, please use the
[discussion board](https://github.com/explosion/spaCy/discussions).
Bug reports can be filed on the [spaCy issue tracker](https://github.com/explosion/spaCy/issues). Thank you!
