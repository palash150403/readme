# K8sGPT

## Install K8sgpt
-   offical Docs

https://k8sgpt.ai/docs/getting-started/installation

```
curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/latest/download/k8sgpt_Linux_x86_64.tar.gz

tar -xzf k8sgpt_Linux_x86_64.tar.gz

sudo mv k8sgpt /usr/local/bin/
```

## Install Ollama

-   Offocal Doc
https://ollama.com/download
```
curl -fsSL https://ollama.com/install.sh | sh
```

## K8sgpt and ollam integration

```
ollama pull llama3
```

```
k8sgpt auth add --backend ollama --model llama3:latest
```

```
k8sgpt auth list
```

Expected result something like
```
Default: 
> openai
Active: 
> ollama

```

### To analyze the error.
```
k8sgpt analyze --explain -b ollama
```

## Azure Open Ai Integration

1. Create an azure open Ai service
2. open the service oprtal
3. go to deploymnets
4. select the model and deploy the model
5. now in your local system run this command to authenticate the ai
```
k8sgpt auth add -b azureopenai   --baseurl <get-it-from-portal->endpoints(in overview)>   --engine <name-of-deployment>   --model <model-name>
```

6. set as default
```
k8s auth default --provider azureopenai
```

### Features

#### 1. Set Filters

You can set filter and search for issues related to your services

```
k8s filters list
```
This will list only issues related to Pods only.

```
k8s analyse --explain --filter Pod
```

#### 2. Make it interactive

```
k8s analyse --explain --filter Pod -i
```