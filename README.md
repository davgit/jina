<p align="center">
<br><br><br>
<a href="https://docs.jina.ai"><img src="https://github.com/jina-ai/jina/blob/master/docs/_static/logo-light.svg?raw=true" alt="Jina logo: Build cross-modal and multi-modal applications on the cloud" width="150px"></a>
<br><br><br>
</p>

<p align="center">
<b>Build cross-modal and multi-modal applications on the cloud</b>
</p>


<p align=center>
<a href="https://github.com/jina-ai/jina/actions/workflows/cd.yml"><img alt="Github CD status" src="https://github.com/jina-ai/jina/actions/workflows/cd.yml/badge.svg"></a>
<a href="https://pypi.org/project/jina/"><img alt="PyPI" src="https://img.shields.io/pypi/v/jina?label=PyPI&logo=pypi&logoColor=white&style=flat-square"></a>
<a href="https://codecov.io/gh/jina-ai/jina"><img alt="Codecov branch" src="https://img.shields.io/codecov/c/github/jina-ai/jina/master?logo=Codecov&logoColor=white&style=flat-square"></a>
<a href="https://slack.jina.ai"><img src="https://img.shields.io/badge/Slack-3.0k-blueviolet?logo=slack&amp;logoColor=white&style=flat-square"></a>
</p>

<!-- start jina-description -->

Jina is a framework that empowers anyone to build cross-modal and multi-modal<sup><a href="#example-application">[*]</a></sup> applications on the cloud. It uplifts a PoC into a production-ready service in just minutes. Jina handles the infrastructure complexity, making advanced solution engineering and cloud-native technologies accessible to every developer. 

Applications built with Jina enjoy the following features out-of-the-box:

🌌 **Universal**
  - Build applications that deliver fresh insights from multiple data types such as text, image, audio, video, 3D mesh, PDF with [Jina AI's DocArray](https://docarray.jina.ai).
  - Support all mainstream deep learning frameworks.
  - Polyglot gateway that supports gRPC, Websockets, HTTP, GraphQL protocols with TLS.

⚡ **Performance**
  - Intuitive design pattern for high-performance microservices.
  - Scaling at ease: set replicas, sharding in one line. 
  - Duplex streaming between client and server.
  - Async and non-blocking data processing over dynamic flows.

☁️ **Cloud-native**
  - Seamless Docker integration: sharing, exploring, sandboxing, versioning and dependency control via [Jina Hub](https://hub.jina.ai).
  - Fast deployment to Kubernetes, Docker Compose and Jina Cloud.
  - Full observability via Prometheus and Grafana.

🍱 **Ecosystem**
  - Improved engineering efficiency thanks to the Jina AI ecosystem, so you can focus on innovating with the data applications you build.

<sup><a id="example-application">[*]</a> Example cross-modal application: <a href="https://github.com/jina-ai/dalle-flow/">DALL·E Flow</a>; example multi-modal services: <a href="https://github.com/jina-ai/clip-as-service/">CLIP-as-service</a>, <a href="https://github.com/jina-ai/now/">Jina Now</a>.</sup>

<!-- end jina-description -->

## [Documentation](https://docs.jina.ai)

## Install 

```bash
pip install jina
```

[More install options can be found in the docs](https://docs.jina.ai/get-started/install/).


## Get Started


### Basic Concepts

Document, Executor and Flow are three fundamental concepts in Jina.

- [**Document**](https://docarray.jina.ai/) is the fundamental data structure.
- [**Executor**](https://docs.jina.ai/fundamentals/executor/) is a group of functions with Documents as IO.
- [**Flow**](https://docs.jina.ai/fundamentals/flow/) ties Executors together into a pipeline and exposes it with an API gateway.


### Hello world example

Leveraging these three concepts, let's look at a simple example below:

```python
from jina import DocumentArray, Executor, Flow, requests


class MyExec(Executor):
    @requests
    async def foo(self, docs: DocumentArray, **kwargs):
        for d in docs:
            d.text += 'hello, world!'


f = Flow().add(uses=MyExec).add(uses=MyExec)

with f:
    r = f.post('/', DocumentArray.empty(2))
    print(r.texts)
```

- The first line imports three concepts we just introduced;
- `MyExec` defines an async function `foo` that receives `DocumentArray` from network requests and appends `"hello, world"` to `.text`;
- `f` defines a Flow streamlined two Executors in a chain;
- The `with` block opens the Flow, sends an empty DocumentArray to the Flow, and prints the result.

Running it gives you:

<p align="center">
<a href="#"><img src="https://github.com/jina-ai/jina/blob/master/.github/readme/run-hello-world.gif?raw=true" alt="Running a simple hello-world program" width="70%"></a>
</p>

At the last line we see its output `['hello, world!hello, world!', 'hello, world!hello, world!']`.


While one could use standard Python with the same number of lines and get the same output, Jina accelerate time to market of your application by making it more scalable and cloud-native. Jina handles the infrastructure complexity in production and other Day-2 operations so that you can focus on the data application itself.  

<p align="center">
<a href="https://docs.jina.ai"><img src="https://github.com/jina-ai/jina/blob/master/.github/readme/no-complexity-banner.png?raw=true" alt="Jina: No Infrastructure Complexity, High Engineering Efficiency" width="100%"></a>
</p>

Leveraging these three concepts, let's build a simple image search service, as a "productization" of [DocArray README](https://github.com/jina-ai/docarray#a-complete-workflow-of-visual-search). 


<p align="center">
<a href="https://docs.jina.ai"><img src="https://github.com/jina-ai/jina/blob/master/.github/images/readme-banner1.svg?raw=true" alt="Get started with Jina to build production-ready neural search solution via ResNet in less than 20 minutes" width="100%"></a>
</p>

### Build a service from scratch

<sup>
Preliminaries: <a href="https://pytorch.org/get-started/locally/">install PyTorch & Torchvision</a>
</sup>

1. Import what we need.
    ```python
    from docarray import Document, DocumentArray
    from jina import Executor, Flow, requests
    ```
2. Copy-paste the preprocessing step and wrap it via `Executor`:
    ```python
    class PreprocImg(Executor):
        @requests
        async def foo(self, docs: DocumentArray, **kwargs):
            for d in docs:
                (
                    d.load_uri_to_image_tensor(200, 200)  # load
                    .set_image_tensor_normalization()  # normalize color
                    .set_image_tensor_channel_axis(
                        -1, 0
                    )  # switch color axis for the PyTorch model later
                )
    ```
3. Copy-paste the embedding step and wrap it via `Executor`:
    
    ```python   
    class EmbedImg(Executor):
        def __init__(self, **kwargs):
            super().__init__(**kwargs)
            import torchvision
            self.model = torchvision.models.resnet50(pretrained=True)        
   
        @requests
        async def foo(self, docs: DocumentArray, **kwargs):
            docs.embed(self.model)
    ```
4. Wrap the matching step into an `Executor`:
    ```python
    class MatchImg(Executor):
        _da = DocumentArray()

        @requests(on='/index')
        async def index(self, docs: DocumentArray, **kwargs):
            self._da.extend(docs)
            docs.clear()  # clear content to save bandwidth

        @requests(on='/search')
        async def foo(self, docs: DocumentArray, **kwargs):
            docs.match(self._da, limit=9)
            del docs[...][:, ('embedding', 'tensor')]  # save bandwidth as it is not needed
    ```
5. Connect all `Executor`s in a `Flow`, scale embedding to 3:
    ```python
    f = (
        Flow(port=12345)
        .add(uses=PreprocImg)
        .add(uses=EmbedImg, replicas=3)
        .add(uses=MatchImg)
    )
    ```
    Plot it via `f.plot('flow.svg')` and you get:
    
    <p align="center">
    <img src="https://github.com/jina-ai/jina/blob/master/.github/images/readme-flow-plot.svg?raw=true" title="Jina Flow.plot visualization" width="65%">
    </p>
    
6. Download the image dataset.


<table>
<tr>
<th> Pull from Cloud </th> 
<th> Manually download, unzip and load </th>
</tr>
<tr>
<td> 

```python
index_data = DocumentArray.pull('demo-leftda', show_progress=True)
```
     
</td>
<td>

1. Download `left.zip` from [Google Drive](https://sites.google.com/view/totally-looks-like-dataset)
2. Unzip all images to `./left/`
3. Load into DocumentArray
    ```python
    index_data = DocumentArray.from_files('left/*.jpg')
    ```

</td>
</tr>
</table>

    
7. Index image data:
    ```python
    with f:
        f.post(
            '/index',
            index_data,
            show_progress=True,
            request_size=8,
        )
        f.block()
    ```

The full indexing on 6,000 images should take ~8 minutes on a MacBook Air 2020.

Now you can use a Python client to access the service:

```python
from jina import Client

c = Client(port=12345)  # connect to localhost:12345
print(c.post('/search', index_data[0])['@m'])  # '@m' is the matches-selector
```

To switch from gRPC interface to REST API, you can simply set `protocol = 'http'`:

```python
with f:
    ...
    f.protocol = 'http'
    f.block()
```

Now you can query it via `curl`:

<p align="center">
<a href="https://docs.jina.ai"><img src="https://github.com/jina-ai/jina/blob/master/.github/images/readme-curl.png?raw=true" alt="Use curl to query image search service built by Jina & ResNet50" width="80%"></a>
</p>

Or go to `http://0.0.0.0:12345/docs` and test requests via a Swagger UI:

<p align="center">
<a href="https://docs.jina.ai"><img src="https://github.com/jina-ai/jina/blob/master/.github/images/readme-swagger-ui.gif?raw=true" alt="Visualize visual similar images in Jina using ResNet50" width="60%"></a>
</p>




<p align="center">
<a href="https://docs.jina.ai"><img src="https://github.com/jina-ai/jina/blob/master/.github/images/readme-banner2.svg?raw=true" alt="Get started with Jina to build production-ready neural search solution via ResNet in less than 20 minutes" width="100%"></a>
</p>

### Play with Containerized Executors

You can containerize the Executors and use them in a sandbox thanks to [Hub](https://hub.jina.ai).

1. Move each `Executor` class to a separate folder with one Python file in each:
   - `PreprocImg` -> 📁 `preproc_img/exec.py`
   - `EmbedImg` -> 📁 `embed_img/exec.py`
   - `MatchImg` -> 📁 `match_img/exec.py`
2. Create a `requirements.txt` in `embed_img` as it requires `torchvision`.

    ```text
    .
    ├── embed_img
    │     ├── exec.py  # copy-paste codes of ImageEmbeddingExecutor
    │     └── requirements.txt  # add the requirement `torchvision`
    └── match_img
          └── exec.py  # copy-paste codes of IndexExecutor
    └── preproc_img
          └── exec.py  # copy-paste codes of IndexExecutor
    ```
3. Push all Executors to the [Hub](https://hub.jina.ai):
    ```bash
    jina hub push preproc_img
    jina hub push embed_img
    jina hub push match_img
    ```
   You will get three Hub Executors that can be used via Sandbox, Docker container or source code. 

<p align="center">
<a href="https://docs.jina.ai"><img src="https://github.com/jina-ai/jina/blob/master/.github/images/readme-hub-push.png?raw=true" alt="Jina hub push gives you the sandbox" width="70%"></a>
</p>

4. In particular, Sandbox hosts your Executor on Jina Cloud and allows you to use it from your local machine:
    ```python
    from docarray import DocumentArray
    from jina import Flow

    index_data = DocumentArray.pull(
        'demo-leftda', show_progress=True
    )  # Download the dataset as shown in the tutorial above

    f = Flow().add(uses='jinahub+sandbox://2k7gsejl')

    with f:
        print(f.post('/', index_data[:10]))
    ```

<p align="center">
<a href="https://docs.jina.ai"><img alt="Shell outputs running docker-compose" src="https://github.com/jina-ai/jina/blob/master/.github/images/readme-sandbox-play.png?raw=ture" title="outputs of docker-compose" width="90%"></a>
</p>


<p align="center">
<a href="https://docs.jina.ai"><img src="https://github.com/jina-ai/jina/blob/master/.github/images/readme-banner3.svg?raw=true" alt="Containerize, share and play in one-place like a pro" width="100%"></a>
</p>


### Deploy the service via Docker Compose

1. Now that all Executors are in containers, we can easily use Docker Compose to orchestrate the Flow:

    ```python
    f = (
        Flow(port=12345)
        .add(uses='jinahub+docker://1ylut0gf')
        .add(uses='jinahub+docker://258lzh3c')
    )
    f.to_docker_compose_yaml()  # By default, stored at `docker-compose.yml`
    ```

2. Now in the console run:

    ```shell
    docker-compose up
    ```

<p align="center">
<a href="https://docs.jina.ai"><img alt="Shell outputs running docker-compose" src="https://github.com/jina-ai/jina/blob/master/.github/images/readme-docker-compose.png?raw=ture" title="She;; outputs of docker-compose"  width="85%"></a>
</p>

### Deploy the service via Kubernetes

1. Create a Kubernetes cluster and get credentials (example in GCP, [more K8s providers here](https://docs.jina.ai/advanced/experimental/kubernetes/#preliminaries)):
    ```bash
    gcloud container clusters create test --machine-type e2-highmem-2  --num-nodes 1 --zone europe-west3-a
    gcloud container clusters get-credentials test --zone europe-west3-a --project jina-showcase
    ```

2. Create a namespace `flow-k8s-namespace` for demonstration purpose:
    ```bash
    kubectl create namespace flow-k8s-namespace
    ```

3. Generate the kubernetes configuration files using one line of code:
    ```python
    f.to_kubernetes_yaml('./k8s_config', k8s_namespace='flow-k8s-namespace')
    ```
    
4. Your `k8s_config` folder will look like the following:
    ```shell
    k8s_config
    ├── executor0
    │     ├── executor0-head.yml
    │     └── executor0.yml
    ├── executor1
    │     ├── executor1-head.yml
    │     └── executor1.yml
    └── gateway
          └── gateway.yml
    ```

5. Use `kubectl` to deploy your neural search application: 

    ```shell
    kubectl apply -R -f ./k8s_config
    ```

<p align="center">
<a href="https://docs.jina.ai"><img alt="Shell outputs running k8s" src="https://github.com/jina-ai/jina/blob/master/.github/images/readme-k8s.png?raw=ture" title="kubernetes outputs" width="70%"></a>
</p>

6. Run port forwarding so that you can send requests to your Kubernetes application from local CLI : 

    ```shell
    kubectl port-forward svc/gateway -n flow-k8s-namespace 12345:12345
    ```

Now we have the service up running in Kubernetes!



<!-- start support-pitch -->

## Support

- Check out the [Learning Bootcamp](https://learn.jina.ai) to get started with Jina.
- Join our [Slack community](https://slack.jina.ai) to chat to our engineers about your use cases, questions, and
  support queries.
- Join our [Engineering All Hands](https://youtube.com/playlist?list=PL3UBBWOUVhFYRUa_gpYYKBqEAkO4sxmne) meet-up to
  discuss your use case and learn Jina's new features.
    - **When?** The second Tuesday of every month
    - **Where?**
      Zoom ([see our public calendar](https://calendar.google.com/calendar/embed?src=c_1t5ogfp2d45v8fit981j08mcm4%40group.calendar.google.com&ctz=Europe%2FBerlin)/[.ical](https://calendar.google.com/calendar/ical/c_1t5ogfp2d45v8fit981j08mcm4%40group.calendar.google.com/public/basic.ics)/[Meetup
      group](https://www.meetup.com/jina-community-meetup/))
      and [live stream on YouTube](https://youtube.com/c/jina-ai)
- Subscribe to the latest video tutorials on our [YouTube channel](https://youtube.com/c/jina-ai)

## Join Us

Jina is backed by [Jina AI](https://jina.ai) and licensed under [Apache-2.0](./LICENSE).
[We are actively hiring](https://jobs.jina.ai) AI engineers, solution engineers to build the next neural search
ecosystem in open source.

<!-- end support-pitch -->

## Contribute

We welcome all kinds of contributions from the open-source community, individuals and partners. We owe our success to
your active involvement.

- [Release cycles and development stages](RELEASE.md)
- [Contributing guidelines](CONTRIBUTING.md)
- [Code of conduct](https://github.com/jina-ai/jina/blob/master/.github/CODE_OF_CONDUCT.md)
