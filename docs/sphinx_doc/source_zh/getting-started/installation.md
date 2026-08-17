# 安装指南

本指南介绍安装 TuFT 的不同方式。

```{admonition} 🚀 没有 GPU？没问题！
:class: tip

下面的说明假设你的机器配有 GPU。**没有 GPU？** 你可以将 TuFT 部署到按量付费的云服务商——
**[Modal](../deployment/modal.md)**（无服务器，缩容至零）或
**[Lambda Cloud](../deployment/lambda.md)**——然后在本地（笔记本）驱动微调。
参阅 **[部署指南](../deployment/index.md)**。
```

```{admonition} 🌐 网络受限？
:class: tip

安装 TuFT 需要访问 GitHub、PyPI 和 Hugging Face。如果其中任何一个访问缓慢或不可达，
请参阅 **[网络受限环境](#restricted-network-environments)**，其中提供了诊断脚本和相应的镜像配置。
```

## 快速安装

> **注意**：此脚本支持 Unix 平台。其他平台请参阅下面的手动安装部分。

使用单个命令安装 TuFT：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/agentscope-ai/tuft/main/scripts/install.sh)"
```

这将安装带有完整后端支持（GPU 依赖、持久化、flash-attn）的 TuFT，以及捆绑的 Python 环境到 `~/.tuft`。安装后，重启终端并运行：

```bash
tuft
```

## 从源代码安装

我们推荐使用 [uv](https://github.com/astral-sh/uv) 进行依赖管理。

1. 克隆仓库：

    ```bash
    git clone https://github.com/agentscope-ai/TuFT
    ```

2. 创建虚拟环境：

    ```bash
    cd TuFT
    uv venv --python 3.12
    ```

3. 激活环境：

    ```bash
    source .venv/bin/activate
    ```

4. 安装依赖：

    ```bash
    # 安装最小依赖（非开发安装）
    uv sync

    # 如果需要开发或运行测试，安装开发依赖
    uv sync --extra dev

    # 如果要运行完整功能集（如模型服务、持久化），
    # 请安装所有依赖
    uv sync --all-extras
    python scripts/install_flash_attn.py
    # 如果 flash-attn 安装遇到问题，可以尝试手动安装：
    # uv pip install flash-attn --no-build-isolation
    ```

## 通过 PyPI 安装

```bash
uv pip install "tuft>=0.1.8"

# 根据需要安装可选依赖
uv pip install "tuft[dev,backend,persistence]>=0.1.8"
```

## 使用预构建的 Docker 镜像

如果本地安装遇到问题或想快速开始，可以使用预构建的 Docker 镜像。

1. 从 GitHub Container Registry 拉取最新镜像：

    ```bash
    docker pull ghcr.io/agentscope-ai/tuft:latest
    ```

2. 运行 Docker 容器并在端口 10610 启动 TuFT 服务器：

    ```bash
    docker run -it \
        --gpus all \
        --shm-size="128g" \
        --rm \
        -p 10610:10610 \
        -v <host_dir>:/data \
        ghcr.io/agentscope-ai/tuft:latest \
        tuft launch --port 10610 --config /data/tuft_config.yaml
    ```

    请将 `<host_dir>` 替换为您主机上用于存储模型检查点和其他数据的目录。
    
    假设您的主机上有以下结构：

    ```text
    <host_dir>/
        ├── checkpoints/
        ├── Qwen3-4B/
        ├── Qwen3-8B/
        └── tuft_config.yaml
    ```

(restricted-network-environments)=
## 网络受限环境

TuFT 在安装和运行时都依赖 GitHub、PyPI 和 Hugging Face。如果其中任何一个访问缓慢或不可达，
请先运行仓库自带的诊断脚本，而不是手动排查：

```bash
cd TuFT
bash scripts/env_check.sh
```

该脚本会测量 TuFT 所需各个服务的延迟和下载速度，并针对失败的服务打印出可直接复制粘贴的镜像配置。
脚本检查的内容以及每项失败会阻塞的步骤：

| 失败的步骤 | 受阻的服务 | 解决方式 |
| --- | --- | --- |
| 安装 `uv` | GitHub releases | 从 PyPI 安装 `uv`，而不是使用 `curl` 安装脚本 |
| `uv venv --python 3.12` | GitHub 上的 python-build-standalone | 设置 `UV_PYTHON_INSTALL_MIRROR` |
| `uv sync` | PyPI | 设置 `UV_INDEX` |
| 启动服务器、加载模型或数据集 | Hugging Face | 设置 `HF_ENDPOINT` |

下面的命令使用中国大陆可访问的镜像。如果你有内部镜像源，请替换为自己的地址。

```bash
# 1. 当 github.com 不可达时，从 PyPI 安装 uv
python -m pip install -U pip
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/
pip install -U uv

# 2. `uv venv --python` 下载独立 Python 构建时使用的镜像
export UV_PYTHON_INSTALL_MIRROR=https://python-standalone.org/mirror/astral-sh/python-build-standalone

# 3. `uv sync` 和 `uv pip install` 使用的包索引
export UV_INDEX=https://mirrors.aliyun.com/pypi/simple/

# 4. 下载模型和数据集使用的端点
export HF_ENDPOINT=https://hf-mirror.com
```

把这些 `export` 语句写入 `~/.bashrc` 或 `~/.zshrc` 即可在所有 shell 中生效。`HF_ENDPOINT`
必须在发起下载的进程启动之前设置：在 `tuft launch` 之前 export；在 notebook 中则在第一个单元格里设置
（`os.environ["HF_ENDPOINT"] = ...`）并重启内核。

其他安装方式同样适用这些配置。运行[快速安装](#快速安装)脚本前先 export 这些变量——该脚本底层调用 `uv`，
并且在 `curl` 安装器无法访问 astral.sh 时会自动回退到 `pip install uv`；但请注意，下载 `install.sh`
本身仍然需要访问 `raw.githubusercontent.com`。`docker/Dockerfile` 和
`src/tuft/console/docker/Dockerfile` 中也提供了同样变量的注释行 `ENV`——构建自己的镜像前取消注释即可。

## 运行服务器

CLI 启动一个 FastAPI 服务器：

```bash
tuft launch --port 10610 --config /path/to/tuft_config.yaml
```

配置文件 `tuft_config.yaml` 指定服务器设置，包括可用的基础模型、认证、持久化和遥测。以下是一个最小示例：

```yaml
supported_models:
  - model_name: Qwen/Qwen3-4B
    model_path: Qwen/Qwen3-4B
    max_model_len: 32768
    tensor_parallel_size: 1
  - model_name: Qwen/Qwen3-8B
    model_path: Qwen/Qwen3-8B
    max_model_len: 32768
    tensor_parallel_size: 1
```

请参阅仓库中的 `config/tuft_config.example.yaml` 获取包含所有可用选项的完整示例配置。
