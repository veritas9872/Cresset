# 범용 PyTorch 소스 빌드 Docker 템플릿

## Preamble
최근 몇 년 동안 더 작고 효율적인 장치에서 계속 증가하는 데이터 양에 대처하기 위해
효율적인 신경망을 설계하고 구현하는 데 엄청난 학문적 노력이 투입되었습니다.
그러나 이 글을 쓰는 시점에서 대부분의 딥 러닝 실무자들은 가장 기본적인 GPU 가속 기술조차 모르고 있습니다.

특히 학계에서는 메모리 요구 사항을 1/4로 줄이고,
속도를 4~5배 높일 수 있는 AMP(Automatic Mixed Precision)조차 사용하지 않는 경우가 많습니다.
[HuggingFace Accelerate](https://github.com/huggingface/accelerate) 또는 [PyTorch Lightning](https://github.com/PyTorchLightning/pytorch-lightning)를 사용하여 큰 번거로움 없이 AMP를 활성화할 수 있음에도 마찬가지입니다.
특히 Accelerate 라이브러리는 몇 줄의 코드만으로 기존 PyTorch 프로젝트에 통합할 수 있습니다.

딥 러닝의 신비에 발을 담그기 시작한 초보자라도 더 많은 컴퓨팅이 성공의 핵심 요소라는 것을 알고 있습니다.
과학자가 아무리 똑똑하더라도 10배 더 많은 컴퓨팅으로 경쟁자를 능가하는 것은 결코 대단한 일이 아닙니다.

이 템플릿은 GPU, CUDA, Docker 등에 대한 지식이 많지 않은 연구원과 엔지니어가 __*동일한 하드웨어와 신경망을 사용하여*__ GPU의 성능을 최대한 끌어낼 수 있도록 하기 위해 만들어졌습니다.

PyTorch 소스 빌드가 포함된 Docker 이미지는 이미 공식 [PyTorch Docker Hub](https://hub.docker.com/r/pytorch/pytorch)리포지토리와 [NVIDIA NGC](https://ngc.nvidia.com/catalog/containers/nvidia:pytorch) 리포지토리에서 사용할 수 있지만 이러한 이미지에는 다른 패키지가 많이 설치되어 있어 기존 프로젝트에 통합하기 어렵습니다.
또한 많은 실무자는 Docker 이미지보다 로컬 환경을 사용하는 것을 선호합니다.

여기에 제시된 프로젝트는 다릅니다.
사용자가 설치한 라이브러리를 제외하고 작업할 추가 라이브러리가 없습니다.
빌드에서 생성된 휠은 Docker 사용법을 배울 필요 없이 모든 환경에서 사용하기 위해 추출할 수 있습니다.
 (이 프로젝트의 두 번째 부분에서는 Docker를 훨씬 쉽게 사용할 수 있도록 `docker-compose.yaml` 파일도 제공합니다.)

만약 당신이 더 빨리 끝내고 싶어하는 사람이라면 Tensor board를 응시하면서 오랜 시간을 견디는것이 끝났습니다.
이 프로젝트가 바로 정답일 수 있습니다.
AMP와 결합된 최신 버전의 CUDA와 함께 PyTorch의 소스 빌드를 사용할 때,
순수한 PyTorch 환경보다 학습/추론시간을 10배 빠르게 달성할 수 있습니다.

제 프로젝트가 학계와 산업계의 실무자들에게 도움이 되기를 진심으로 바랍니다.
제 작업이 유익하다가 생각하시는 사용자는 이 저장소에 star을 해주셔서 감사를 표시해주시는 것을 환영합니다.


## Warning
__*이 템플릿을 사용하기 전에 먼저 GPU를 실제로 사용하고 있는지 확인하세요!*__

대부분의 시나리오에서, 느린 학습은 비효율적인 파이프라인 ETL(추출, 변환, 로드)에 의해 발생합니다.
데이터가 GPU가 느리게 실행되기 때문이 아니라 데이터가 GPU에  충분히 빠르게 도달하지 않아서 학습이 느립니다.
GPU 사용률이 컴퓨팅 최적화를 할 만큼 충분히 높은지 확인하려면 `watch nvidia-smi`를 실행하세요.
GPU 사용률이 낮거나 돌발적으로 최고조에 달하는 경우, 이 템플릿을 사용하기 전에 효율적인 ETL 파이프라인을 설계하세요.
그렇지 않으면, 더 빠른 컴퓨팅이 병목 현상이 되지 않으므로 별로 도움이 되지 않습니다.

효율적인 ETL 파이프라인 설계에 대한 가이드는 https://www.tensorflow.org/guide/data_performance 를 참조하세요.

[NVIDIA DALI](https://github.com/NVIDIA/DALI) 라이브러리 또한 도움이 될 수도 있습니다. 
The [DALI PyTorch plugin](https://docs.nvidia.com/deeplearning/dali/user-guide/docs/plugins/pytorch_tutorials.html)
은 PyTorch에서 효율적인 ETL 파이프라인을 위한 API를 제공합니다.


## Introduction
PyTorch/CUDA/cuDNN의 __*모든 버전*__ 에서의 __*소스로부터*__ PyTorch를 빌드하기 위한 템플릿 리포지토리.

소스에서 빌드한 PyTorch는 `pip`/`conda`에서 설치된 PyTorch보다 훨씬 빠르지만(일부 벤치마크에서는 x4배, x2가 더 일반적입니다.
소스에서 빌드하는 것은 힘들고 버그가 발생하기 쉬운 프로세스입니다.

이 리포지토리는 모든 버전의 CUDA에서 의 소스로부터 모든 버전의 PyTorch를 빌드하기 위한 고도로 모듈화된 템플릿입니다.
Linux 기반 이미지 또는 프로젝트에 통합할 수 있는 사용하기 쉬운 Dockerfile을 제공합니다.

Docker에 익숙하지 않은 연구원을 위해 생성된 휠 파일을 추출하여 로컬 환경에 PyTorch를 설치할 수 있습니다.

Windows 사용자는 WSL을 통해 이 프로젝트를 사용할 수도 있습니다. 아래 지침을 참조하십시오.

'Makefile'은 쉬운 사용을 위한 인터페이스와 맞춤형 이미지 구축을 위한 튜토리얼로 제공됩니다.

Docker를 사용한 간단한 대화형 개발 환경을 위해 `docker-compose.yaml` 파일도 제공됩니다.

이 템플릿의 속도 향상은 다음 요소에서 비롯됩니다.
1. 최신 버전의 CUDA 및 관련 라이브러리 사용 (cuDNN, cuBLAS, etc.).
2. 다른 하드웨어 및 소프트웨어 환경과 호환되어야 하는 빌드 대신 최신 소프트웨어 사용자 정의가 포함된 대상 시스템을 위해 특별히 만들어진 소스 빌드를 사용합니다.
3. 최신 버전의 PyTorch 및 보조 라이브러리 사용. 
많은 사용자가 기존 환경과의 호환성 문제로 인해 PyTorch 버전을 업데이트하지 않습니다.
4. 사용자에게 속도 문제에 대한 해답을 어디서 찾는지를 알려줍니다
   (이것이 가장 중요한 요소일 수 있음).

AMP 및 cuDNN 벤치마킹과 같은 기술과 결합하면 __*동일한 하드웨어에서*__
계산 처리량이 극적으로(예: x10) 증가할 수 있습니다.

프로젝트에서 Docker를 사용하지 않으려는 경우에도,
이 템플릿이 유용할 수 있습니다.

**_빌드에서 생성된 휠 파일은 Docker에 의존하지 않고 모든 Python 환경에서 사용할 수 있습니다._**

따라서 이 프로젝트를 사용하여 원하는 환경(`conda`, `pip` 등)에 대해 학습 및 추론 속도를 극적으로 개선하여 
사용자 지정 휠 파일을 생성할 수 있습니다.


## Quickstart
__*사용자는 `Dockerfile`의 `train` 단계를 원하는 대로 자유롭게 사용자 지정할 수 있습니다. 
그러나 절대적으로 필요한 경우가 아니면 '빌드' 단계를 변경하지 마세요.
새 패키지를 빌드해야 하는 경우, 새 `build` 계층을 추가하세요.*__

이 프로젝트는 템플릿이며, 사용자는 필요에 맞게 사용자 정의해야 합니다.

코드는 필요한 NVIDIA 드라이버와 최신 버전의 Docker 및 Docker Compose가 사전 설치된 Linux 호스트에서 실행되는 것으로 가정합니다.
그렇지 않은 경우, 먼저 설치하십시오.

학습 이미지를 빌드하려면,
먼저 `apt`/`conda`/`pip`에서 원하는 패키지를 포함하도록 Dockerfile `train` 단계를 편집합니다.

그런 다음 https://developer.nvidia.com/cuda-gpus를 방문하여
대상 GPU 장치의 컴퓨팅 기능(CC)을 찾으세요.

마지막으로, `make all CC=TARGET_CC(s)`를 실행하세요.


### Examples 
(1) RTX 3090용 `make all CC="8.6"`, 
(2) RTX 2080Ti 와 RTX 3090용 `make all CC="7.5 8.6"`
(많은 GPU CC용으로 빌드하면 빌드 시간이 늘어남).

그러면 학습에 사용할 수 있는 `pytorch_source:train` 이미지가 생성됩니다.

빌드 중에 사용할 수 없는 장치용 CC는 이미지를 빌드하는 데 사용할 수 있다는것을 주의하세요.
예를 들어, 이미지를 RTX 2080Ti 시스템에서 사용해야 하지만 사용자에게 RTX 3090만 있는 경우, 
사용자는 이미지가 RTX 2080Ti GPU에서 작동하도록 `CC="7.5"`를 설정할 수 있습니다.
`Makefile`에서 `CC`로 지정되는 `TORCH_CUDA_ARCH_LIST`를 설정하는 방법에 대한 자세한 가이드는
https://pytorch.org/docs/stable/cpp_extension.html 를 참조하세요.


### Makefile Explanation
`Makefile` 은 이 패키지를 간단하고 모듈화하여 사용할 수 있도록 설계되었습니다.

생성될 첫 번째 이미지는 빌드에 필요한 모든 패키지가 포함된 `pytorch_source:build_install`입니다.
설치 이미지는 다운로드를 캐시하기 위해 별도로 생성됩니다.

두 번째 이미지는 `pytorch_source:build_torch-v1.9.1`(기본값)입니다.
여기에는 Python 3.8, CUDA 11.3.1 그리고 cuDNN 8이 포함된 
Ubuntu 20.04 LTS의 PyTorch 1.9.1 설정과 함께 PyTorch, TorchVision, TorchText 및 TorchAudio용 휠이 포함되어 있습니다.
두 번째 이미지는 빌드 프로세스의 결과를 캐시하기 위해 존재합니다.

Docker를 사용하지 않고 환경에 pip 설치를 위한 `.whl` 휠 파일만 추출하려는 경우,
생성된 휠 파일은 `/tmp/dist` 디렉토리에서 찾을 수 있습니다.

빌드 결과를 저장하면 다른 PyTorch 버전(다른 CUDA 버전, 다른 라이브러리 버전 등)이 필요한 경우보다 편리한 버전 전환이 가능합니다.

최종 이미지는 실제 학습에 사용할 이미지인 `pytorch_source:train`입니다.
빌드 아티팩트(휠 등)에 대해서만 이전 단계에 의존하고 다른 것은 전혀 사용하지 않습니다.
이를 통해 다양한 환경 및 GPU 장치에 최적화된 다양한 훈련 이미지를 매우 간단하게 생성할 수 있습니다.

PyTorch가 이미 빌드되었기 때문에,
학습 이미지는 나머지 `apt`/`conda`/`pip` 패키지만 다운로드하면 됩니다.
이 프로세스의 속도를 높이기 위해 캐싱도 구현됩니다.


### Timezone Settings
해외 사용자는 이 섹션이 도움이 될 수 있습니다.

`학습` 이미지에는 `tzdata` 패키지를 사용하여 `TZ` 변수에 의해 설정된 시간대가 있습니다.
기본 시간대는 `Asia/Seoul` 이지만 `make` 호출 시 `TZ` 변수를 지정하여 변경할 수 있습니다.
[IANA](https://www.iana.org/time-zones) 시간대 이름을 사용하여 원하는 시간대를 지정합니다.

예시: `make all CC="8.6" TZ=America/Los_Angeles` 는 학습 이미지에서 L.A.시간을 사용합니다.

참고: 학습 이미지에만 시간대 설정이 있습니다.
설치 및 이미지 빌드는 시간대 정보를 사용하지 않습니다.

또한, 학습 이미지에는 한국어 사용자를 위해 업데이트된 'apt' 및 'pip' 설치 URL이 있습니다.
설치 속도를 높이려면,
설치 캐시로 인해 불필요할 수 있지만,
해당 위치에 최적화된 URL을 찾으십시오.


## Specific PyTorch Version
__*PyTorch 보조 라이브러리는 일치하는 PyTorch 버전에서만 작동합니다.*__

To change the version of PyTorch,
set the [`PYTORCH_VERSION_TAG`](https://github.com/pytorch/pytorch), 
[`TORCHVISION_VERSION_TAG`](https://github.com/pytorch/vision), 
[`TORCHTEXT_VERSION_TAG`](https://github.com/pytorch/text), and 
[`TORCHAUDIO_VERSION_TAG`](https://github.com/pytorch/audio) 
variables to matching versions.

The `*_TAG` variables must be GitHub tags or branch names of those repositories.
Visit the GitHub repositories of each library to find the appropriate tags.

Example: To build on an RTX 3090 GPU with PyTorch 1.9.1, use the following command:

`make all CC="8.6" 
PYTORCH_VERSION_TAG=v1.9.1 
TORCHVISION_VERSION_TAG=v0.10.1 
TORCHTEXT_VERSION_TAG=v0.10.1
TORCHAUDIO_VERSION_TAG=v0.9.1`.

The resulting image, `pytorch_source:train`, can be used 
for training with PyTorch 1.9.1 on GPUs with Compute Capability 8.6.


## Multiple Training Images
To use multiple training images on the same host, 
give a different name to `TRAIN_NAME`, 
which has a default value of `train`.

New training images can be created without having to rebuild PyTorch
if the same build image is used for different training images.
Creating new training images takes only a few minutes at most.

This is useful for the following use cases.
1. Allowing different users, who have different UID/GIDs, 
to use separate training images.
2. Using different versions of the final training image with 
different library installations and configurations.
3. Using this template for multiple PyTorch projects,
each with different libraries and settings.

For example, if `pytorch_source:build_torch-v1.9.1` has already been built,
Alice and Bob would use the following commands to create separate images.

Alice:
`make build-train 
CC="8.6"
TORCH_NAME=build_torch-v1.9.1
PYTORCH_VERSION_TAG=v1.9.1
TORCHVISION_VERSION_TAG=v0.10.1
TORCHTEXT_VERSION_TAG=v0.10.1
TORCHAUDIO_VERSION_TAG=v0.9.1
TRAIN_NAME=train_alice`

Bob:
`make build-train 
CC="8.6"
TORCH_NAME=build_torch-v1.9.1
PYTORCH_VERSION_TAG=v1.9.1
TORCHVISION_VERSION_TAG=v0.10.1
TORCHTEXT_VERSION_TAG=v0.10.1
TORCHAUDIO_VERSION_TAG=v0.9.1
TRAIN_NAME=train_bob` 

This way, Alice's image would have her UID/GID while Bob's image would have his UID/GID.
This procedure is necessary because training images have their users set during the build.
Also, different users may install different libraries in their training images.
Their environment variables and other settings may also be different.


### Word of Caution
When using build images such as `pytorch_source:build_torch-v1.9.1` as a build cache 
for creating new training images, the user must re-specify all build arguments 
(variables specified by ARG and ENV using --build-arg) of all previous layers.

Otherwise, the default values for these arguments will be given to the Dockerfile
and a cache miss will occur because of the different input values.

This will both waste time rebuilding previous layers and, more importantly,
cause inconsistency in the training images due to environment mismatch.

This includes the `docker-compose.yaml` file as well. 
All arguments given to the `Dockerfile` during the build must be respecified.
This includes default values present in the `Makefile` 
but not present in the `Dockerfile` such as the version tags.

__*If Docker starts to rebuild layers that you have already built, 
suspect that build arguments have been given incorrectly.*__ 

See https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#leverage-build-cache
for more information.

The `BUILDKIT_INLINE_CACHE` must also be given to an image to use it as a cache later. See 
https://docs.docker.com/engine/reference/commandline/build/#specifying-external-cache-sources
for more information.


## Advanced Usage
The `Makefile` provides the `*-full` commands for advanced usage.

`make all-full CC=YOUR_GPU_CC TRAIN_NAME=train_cu102` will create 
`pytorch_source:build_install-ubuntu18.04-cuda10.2-cudnn8-py3.9`,
`pytorch_source:build_torch-v1.9.1-ubuntu18.04-cuda10.2-cudnn8-py3.9`, 
and `pytorch_source:train_cu102` by default.

These images can be used for training/deployment on CUDA 10 devices such as the GTX 1080Ti.

Also, the `*-clean` commands are provided to check for cache reliance on previous builds.


### Specific CUDA Version
Set `CUDA_VERSION`, `CUDNN_VERSION`, and `MAGMA_VERSION` to change CUDA versions.
`PYTHON_VERSION` may also be changed if necessary.

This will create a build image that can be used as a cache 
to create training images with the `build-train` command.

Also, the extensive use of caching in the project means that 
the second build is much faster than the first build.
This may be advantageous if many images must be created for multiple PyTorch/CUDA versions.

### Specific Linux Distro
CentOS and UBI images can be created with only minor edits to the `Dockerfile`.
Read the `Dockerfile` for full instructions.

Set the `LINUX_DISTRO` and `DISTRO_VERSION` arguments afterwards.

### Windows
Windows users may use this template by updating to Windows 11 and installing 
Windows Subsystem for Linux (WSL).
WSL on Windows 11 gives a similar experience to using native Linux.

This project has been tested on WSL on Windows 11 
with the WSL CUDA driver and Docker Desktop for Windows.


# Interactive Development with Docker Compose

## _Raison D'être_
The purpose of this section is to introduce 
a new paradigm for deep learning development. 
I hope that using Docker Compose for deep learning projects
will eventually become best practice, 
improving the reproducibility of ML experiments 
and freeing ordinary researchers from the burden 
of managing their development environments.

Developing in local environments with `conda` or `pip` 
is commonplace in the deep learning community.
However, this risks making the development environment, 
and the code meant to run on it, unreproducible.
This is a serious detriment to scientific progress 
that many readers of this article 
will have experienced at first-hand.

Docker containers are the standard method for
providing reproducible programs 
across different computing environments. 
They create isolated environments where programs 
can run without interference from the host or from one another.
See https://www.docker.com/resources/what-container for details.

But in practice, Docker containers are often misused. 
Containers are meant to be transient and best practice dictates that 
a new container be created for each run.
But this is very inconvenient for development, 
especially for deep learning applications, 
where new libraries must constantly be installed and 
bugs are often only evident at runtime.
This leads many researchers to develop inside interactive containers.
Docker users often have `run.sh` files with commands such as
`docker run -v my_data:/mnt/data -p 8080:22 -t my_container my_image:latest /bin/bash`
(does this look familiar to anyone?) and use SSH to connect to running containers.
VSCode also provides a remote development mode that can be used 
to code inside containers.

The problem with this approach is that these interactive containers 
become just as unreproducible as local development environments.
A running container cannot connect to a new port or attach a new volume.
But if the computing environment within the container was created over several months 
of installs and builds, the only way to keep it 
is to save it as an image and create a new container from the saved image.
After a few iterations of this process, 
the resulting image becomes bloated and completely unreproducible.

To alleviate this problem, a `docker-compose.yaml` 
file is provided for easy management of containers.
Docker Compose is part of Docker and is already 
a popular tool for both development and production.
For unknown reasons, it has not taken off in the deep learning community yet, 
though anyone who knows how to use Docker will find Compose fairly simple.
This may be because Compose is often advertised as a multi-container solution,
though it can also be used for single-container development just as well.

The `docker-compose.yaml` file allows the user to specify settings 
for both build and run.
Connecting a new volume is as simple as removing the current container,
adding a line in the `docker-compose.yaml`/`Dockerfile` file, 
then creating a new container from the same image. 
Build caches allow new images to be built very quickly,
removing another barrier to Docker adoption,
the long initial build time.

The instructions below allow interactive development on the terminal,
making the transition from local development to 
Docker and Docker Compose much smoother.

With luck, the deep learning community will be able to 
"_code once, train anywhere_" with this technique.
But even if I fail in persuading the majority of users 
of the merits of my method,
I may still spare many a hapless grad student from the 
sisyphean labor of setting up their `conda` environment,
only to have it crash and burn right before their paper submission is due.


## Usage
__*Docker images created by the `Makefile` 
are fully compatible with the `docker-compose.yaml` file.
There is no need to erase them to use Docker Compose.*__

Using Docker Compose V2 (see https://docs.docker.com/compose/cli-command),
run the following two commands, where `train` is the default service name 
in the provided `docker-compose.yaml` file.

0. Read `docker-compose.yaml` and set variables in the `.env` file (first time only).
1. `docker compose up -d train`
2. `docker compose exec train /bin/bash`

This will open an interactive shell with settings specified by the `train` service 
in the `docker-compose.yaml` file. 
Environment variables can be saved in a `.env` file placed on the project root,
removing the need to type in variables such as UID/GID values with each run.
To create a basic `.env` file, run `make env`.

This is extremely convenient for managing reproducible development environments.
For example, if a new `pip` or `apt` package must be installed for the project,
users can simply edit the `train` layer of the 
`Dockerfile` by adding the package to the 
`apt-get install` or `pip install` commands, 
then run the following command:

`docker compose up -d --build train`.

This will remove the current `train` session, rebuild the image, 
and start a new `train` session.
It will not, however, rebuild PyTorch (assuming no cache miss occurs).
Users thus need only wait a few minutes for the additional downloads, 
which are accelerated by caching and with fast mirror URLs.

To stop and restart a service after editing the 
`Dockerfile` or `docker-compose.yaml` file,
simply run `docker compose up -d train` again.

To remove all Compose containers, use the following:

`docker compose down`.

Users with remote servers may use Docker contexts
(see https://docs.docker.com/engine/context/working-with-contexts)
to access their containers from their local environments.
For more information on Docker Compose, see the documentation
https://github.com/compose-spec/compose-spec/blob/master/spec.md.


## Compose as Best Practice

I wish to emphasize that using Docker Compose in this manner 
is a general-purpose technique 
that does not depend on anything about this project.
As an example, an image from the NVIDIA NGC PyTorch repository 
has been used as the base image in `ngc.Dockerfile`.
The NVIDIA NGC PyTorch images contain many optimizations 
for the latest GPU architectures and provides
a multitude of pre-installed machine learning libraries. 
For anyone starting a new project, and therefore with no dependencies,
using the latest NGC image is recommended.

To use the NGC images, use the following commands:

1. `docker compose up -d ngc`
2. `docker compose exec ngc /bin/bash`

The only difference with the previous `train` session is the session name.


# Known Issues

1. Connecting to a running container by `ssh` will remove all variables set by `ENV`.
This is because `sshd` starts a new environment, wiping out all previous variables.
Using `docker`/`docker compose` to enter containers is strongly recommended.

2. Building on CUDA 11.4.x is not available as of October 2021 because `magma-cuda114`
has not been released on the `pytorch` channel of anaconda.
Users may attempt building with older versions of `magma-cuda` 
or try the version available on `conda-forge`.
A source build of `magma` would be welcome as a pull request.

3. Ubuntu 16.04 build fails. 
This is because the default `git` installed by `apt` on 
Ubuntu 16.04 does not support the `--jobs` flag. 
Add the `git-core` PPA to `apt` and install the latest version of git.
Also, PyTorch v1.9+ will not build on Ubuntu 16. 
Lower the version tag to v1.8.2 to build.
However, the project will not be modified to accommodate 
Ubuntu 16.04 builds as Xenial Xerus has already reached EOL.


# Desiderata

0. **MORE STARS**. If you are reading this, star this repository immediately. I'm serious.

1. CentOS and UBI images have not been implemented yet.
As they require only simple modifications, 
pull requests implementing them would be very much welcome.

2. Translations into other languages are welcome. 
Please make a separate `LANG.README.md` file and create a PR.

3. Please feel free to share this project! I wish you good luck and happy coding!