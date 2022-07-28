# 简介
`Compose` 项目是 Docker 官方的开源项目，负责实现对 Docker 容器集群的快速编排。

其代码目前在 [https://github.com/docker/compose](https://github.com/docker/compose)  上开源。

允许用户通过一个单独的 `docker-compose.yml` 模板文件（YAML 格式）来定义一组相关联的应用容器为一个项目（project）。

`Compose` 中有两个重要的概念：

-   服务 (`service`)：一个应用的容器，实际上可以包括若干运行相同镜像的容器实例。
-   项目 (`project`)：由一组关联的应用容器组成的一个完整业务单元，在 `docker-compose.yml` 文件中定义。
一个项目可以由多个服务（容器）关联而成，`Compose` 面向 `project` 进行管理

# 使用
