# Docker Swarm



## networks

在 Docker 中，如果你希望手动创建一个名为 `idc` 的网络，可以使用以下命令：

### 1. **创建 Overlay 网络**（适用于 Docker Swarm 模式）

如果你正在使用 Docker Swarm 模式，并且希望创建一个 **Overlay 网络**，可以使用以下命令：

```
docker network create --driver overlay idc
```

- **`--driver overlay`** 表示你要创建一个跨多个 Docker 主机的 Overlay 网络。
- **`idc`** 是你想要创建的网络名称。

### 2. **创建 Bridge 网络**（适用于单节点 Docker）

如果你只是在 Docker 的单个节点上工作，并且希望创建一个 **Bridge 网络**，可以使用以下命令：

```
docker network create --driver bridge idc
```

- **`--driver bridge`** 表示你要创建一个本地的桥接网络，它仅适用于在同一主机上的容器。

### 3. **查看网络**

创建网络后，你可以通过以下命令查看当前 Docker 网络的列表：

```
docker network ls
```

你将看到包括 `idc` 在内的网络列表。

### 4. **连接容器到网络**

当你启动容器时，可以使用 `--network` 标志将容器连接到你创建的 `idc` 网络：

```
docker run -d --network idc <image_name>
```

这样，容器将加入 `idc` 网络，能够与同一网络下的其他容器通信。

### 总结：

- 在 **Docker Swarm** 模式下，使用 **Overlay 网络**：`docker network create --driver overlay idc`
- 在 **单节点 Docker** 下，使用 **Bridge 网络**：`docker network create --driver bridge idc`