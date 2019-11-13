helm template --name jhub ./jupyterhub   --namespace jhub    --set version=0.8.2   --values config.yaml > jupyter-helm.yaml



## 用户容器资源回收

JupyterHub将自动删除一段时间内没有任何活动的用户pod容器资源，这将有助于释放计算资源并降低成本。当这些用户需要重新使用Jupyter服务时，需要重新启动服务器才能继续使用，并且先前会话的状态（他们创建的变量，任何内存中的数据等）将丢失。

在JupyterHub中，“不活动”定义为用户浏览器无响应。JupyterHub会持续对用户的JupyterHub浏览器会话执行ping操作，以检查其是否为打开状态。这意味着在JupyterHub窗口处于打开状态下运行计算机**不会**被视为不活动。

默认情况下，JupyterHub将每十分钟运行一次筛选过程，并且将剔除已超过一个小时不活动的所有用户Pod。您可以`config.yaml`使用以下字段在文件中配置此行为：

```
cull:
  timeout: <max-idle-seconds-before-user-pod-is-deleted>
  every: <number-of-seconds-this-check-is-done>
```



## Admin用户

JupyterHub具有 拥有特殊权限的[管理员用户](https://jupyterhub.readthedocs.io/en/latest/getting-started/authenticators-users-basics.html#configure-admins-admin-users)的概念 。他们可以启动/停止其他用户的服务器，还可以选择访问用户的笔记本。他们将在“控制面板”中看到一个新的“ **管理”**按钮，它将带他们进入“ **管理面板”**，在其中可以执行所有这些操作。

您可以在中指定管理员用户列表`config.yaml`：

```
auth:
  admin:
    users:
      - adminuser1
      - adminuser2
```

默认情况下，管理员可以访问用户的笔记本。如果您想禁用此功能，请在您的中使用`config.yaml`：

```
auth:
  admin:
    access: false
```