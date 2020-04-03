# Cloud IAM 政策继承机制

本文对谷歌云官方文档进行了重新整理和排版，以方便复习。

Google Cloud 提供了 Cloud IAM (身份和访问管理)，可让您分配对特定 Google Cloud 资源的精细访问权限，并防止对其他资源进行不必要的访问。通过 Cloud IAM，您可以为资源设置 Cloud IAM 政策，借此控制谁（用户）对哪些资源具有什么访问权限（角色）。

您可以在组织级层、文件夹级层、项目级层或（在某些情况下）资源级层设置 Cloud IAM 政策。 资源会继承父节点的政策。如果您在组织级层设置政策，则组织下的所有子文件夹和项目都会继承该政策；如果您在项目级层设置政策，则项目下的所有子资源都会继承该政策。

资源的有效政策是指为该资源设置的政策与从其祖先继承而来的政策的集合。这种继承具有传递性。换句话说，资源继承项目的政策，项目会继承组织的政策。因此，组织级层的政策也适用于资源级层。

![](https://cloud.google.com/resource-manager/img/cloud-folders-hierarchy.png)

举例来说，在上面的资源层次结构图中，如果您对文件夹“Dept Y”设置了向 bob@example.com 授予 Project Editor 角色的政策，那么 Bob 将对“Dev GCP Project”、“Test GCP Project”和“Production GCP Project”项目具有 Editor 角色。反之，如果您在“Test GCP Project”项目上向 alice@example.com 分配 Instance Admin 角色，那么她将只能管理该项目中的 Compute Engine 实例。

Cloud IAM 政策层次结构遵循与 Google Cloud 资源层次结构相同的路径。如果您更改资源层次结构，则政策层次结构也会发生更改。例如，将项目移到组织中会将会更新该项目的 Cloud IAM 政策以沿用组织的 Cloud IAM 政策。同样，将项目从一个文件夹移动到另一个文件夹将更改继承的权限。将某个项目移动到新文件夹后，该项目从原始父项继承的权限将丢失。项目在迁移时会继承在目标文件夹设置的权限。

## 参考链接
[资源层次结构](https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy), by 谷歌云官方文档
