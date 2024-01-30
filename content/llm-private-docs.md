Title: Experiments with local LLM for local documents
Author: Ferdinando Simonetti
Tags: LLM, Ollama, Llamaindex, ChromaDB, Qdrant, Python
Category: Programming
Date: 2024-01-30
Modified: 2024-01-30

Here I've tried to describe my journey through LLMs for local knowledge base documents lookup.

Let's start warning my three readers that I'm running these tools on my local MiniPC (Intel i5-8259u, 16Gb RAM, integrated GPU) so everything is kinda *sssssssssssslllllllllllooooooooowwwwww*... and that forces me to avoid using more powerful models and/or relying on heavy quantizations.

**Additional warning**: this is intended as a journal, so it would be a work-in-progress document.

## Basic building bricks / support tools

### Ollama

This software ([product homepage](https://ollama.io)) allows you to run **Large Language Models** of your choice on *commodity* hardware (like your desktop PC).

Ollama leverages **Nvidia** (and **AMD**) GPUs if present and **CUDA** drivers are installed.

If you run the installer oneliner

```
curl https://ollama.ai/install.sh | sh
```

you'll end up with an already running **systemd** service that, however, listens only on **localhost**.

If you want Ollama to listen on **all interfaces** of your PC / server, you'll have to modify the service file */etc/systemd/system/ollama.service* adding this line

```
Environment="OLLAMA_HOST=0.0.0.0"
```

, reloading the services, and restarting Ollama.

```
sudo systemctl daemon-reload
sudo systemctl stop ollama
sudo systemctl start ollama
```

To **upgrade** Ollama, you have to only re-run the installation script (and, probably, fix the *service* file again).

You can also use a Dockerized Ollama version instead, as demonstrated below

### Ollama Web UI

It presents itself as a **User-Friendly Web Interface for Chat Interactions**, you can find it [here](https://github.com/ollama-webui/ollama-webui).

Its *docker-compose.yml* file starts both a container-based Ollama instance and its Web companion that enables the user to download and customize (operating parameters and prompts) new LLM models from Ollama's [model library](https://ollama.ai/library) and/or other sources, as well as loading personal documents to use as **query context** within a query.

There are other *docker-compose.xxx* files, meant for Ollama server behavior's customization:
- **docker-compose.api.yaml**: exposes Ollama API port to the host
- **docker-compose.gpu.yaml**: enables GPU support
- **docker-compose.data.yaml**: uses host system's directory to persist Ollama data, instead of relying on Docker volumes

It's worth noting that the *document-based* query works only if you explicitly references them using `#` notation or load them directly when querying.

To interact with the Dockerized Ollama instance, you can use ```docker compose exec ollama ollama ...``` or your local **ollama** client, after having stopped and disabled the systemd-based instance. 

### Ollama Chrome extension

There is an [in-browser](https://chromewebstore.google.com/detail/ollama-ui/cmgdpmlhgjhoadnonobjeekmfcehffco?pli=1) interface for Ollama that allows you to query local and remote Ollama instances using the models of your choice (between those already installed).

### Models

On [Huggingface](https://huggingface.co/models) model library I've found [a model](https://huggingface.co/mymaia/Magiq-M0) whose primary purpose is *to enhance interactions in English, French, and Italian, each with unique linguistic peculiarities and nuances*; given that I'm Italian, and my end goal is building tools for:
- loading (as chunks) all sort of corporate documents (mostly written in Italian for an Italian audience, otherwise in English) on a vector database running on a corporate VMs, taking care of updates as well
- allowing work colleagues to query the above built *knowledge base* using natural language, via a command line and/or web-based tool

the vast majority of the models in the wild are English-centric, so quite poorly fitted for it.

#### Model download and quantize

Let's follow the instructions found [here](https://github.com/ollama/ollama/blob/main/docs/import.md) to get the previously mentioned **Magiq-M0** model for local use.

```
git lfs install
git clone https://huggingface.co/mymaia/Magiq-M0.git
cd Magiq-m0
```

Convert and quantize, will output two files: **f16.bin** (the model in GGUF format) and **q4_0.bin** (a *quantized*, aka reduced size) version of the model that we'll use inside our Modelfile

```
docker run --rm -v .:/model ollama/quantize -q q4_0 /model
```

The first line of said **Modelfile** will be:

```
FROM ./q4_0.bin
```

#### Model tuning

Each model could be customized via a **Modelfile** (described in detail [here](https://github.com/ollama/ollama/blob/main/docs/modelfile.md)).
You can look up existing model configurations via ```ollama show ...``` or from within Web UI.

### Ollama Hub

On [Ollama Hub](https://ollamahub.com/) you can search within a huge catalog of (user-contributed) **Modelfiles** and **Prompts**

### Pretrained models for sentence transformers

This [website](https://www.sbert.net/docs/pretrained_models.html) has a collection of **sentence transformers**, used to overlap the (vectorized) user query with the space of vectorized knowledge base sentences.

### Llama Hub

A [Repository](https://llamahub.ai/) of *Data Loaders* for specific source documents' types (Word, Excel, PDF, Powerpoint and so on), Connectors (*Agent Tools*) for online services (Google Docs, Slack...) and *Packs* (quite all-in-one, more vertical, solutions)

## Existing projects

Scanning the Internet for useful hints, I've found [this project](https://github.com/PromptEngineer48/Ollama.git) that seems to be a good starting point for what I need.

