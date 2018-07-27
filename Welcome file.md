
### 1. Brief introduction to CRI

At the bottom of each Kubernetes node, a program is responsible for the creation and deletion of specific container, and Kubernetes calls its interface to complete the container scheduling. We call this layer of software the Container Runtime, which is represented by the famous Docker.

Of course, Docker is not the only container runtime, including the RKT of CoreOS, the runV of hyper.sh, the gvisor of Google, and the PouchContainer of this article. All of them contain complete container operations that can be used to create containers with different characteristics. Different kinds of container runtime have their own unique advantages and can meet the needs of different users. Therefore, it is imperative for Kubernetes to support multiple container runtimes.

Initially, Kubernetes had a built-in call interface to Docker, and then the community integrated the RKT interface in Kubernetes 1.3, making it an optional container runtime in addition to Docker. However, at this time, both calls to Docker and to RKT are strongly coupled to Kubernetes' core code, which undoubtedly brings the following two problems:

1. Emerging container operators, such as PouchContainer, are struggling to add Kubernetes to their ecosystem. Developers of the container runtime must have a very deep understanding of Kubernetes's code (at least Kubelet) in order to successfully connect the two.
2. Kubernetes' code will be more difficult to maintain, which is reflected in two aspects:（1）If hard-coding all the call interfaces of the various container runtime into Kubernetes, the core code of Kubernetes will be bloated,（2）Minor changes to the container runtime interface will trigger changes to the core code of Kubernetes and increase its instability.

In order to solve these problems, the community introduced the Container Runtime Interface in Kubernetes 1.5. By defining a set of common interfaces of the Container Runtime, the calling Interface of Kubernetes for various container runtimes was shielded from the core code. The core code of Kubernetes only called the abstract Interface layer. However, for various containers, Kubernetes can be accessed smoothly as long as the interfaces defined in CRI are satisfied, which makes it one of the container runtime options. The solution, while simple, is a liberation for the Kubernetes community maintainers and container runtime developers.

### 5. Pod Network Configuration

Since all containers inside Pod share a Network Namespace, we only need to configure Network Namespace when creating the infra container.

All network functions of containers in Kubernetes ecosystem are implemented via CNI. Similar to CRI, CNI is a set of standard interfaces and any network schema implementing these interfaces can be integrated into Kubernetes seamlessly. CNI Manager in CRI Manager is a simple encapsulation of CNI. It will load configuration files under `/etc/cni/net.d` during initialization, as follows:

```sh
$ cat >/etc/cni/net.d/10-mynet.conflist <<EOF
{
        "cniVersion": "0.3.0",
        "name": "mynet",
        "plugins": [
          {
                "type": "bridge",
                "bridge": "cni0",
                "isGateway": true,
                "ipMasq": true,
                "ipam": {
                        "type": "host-local",
                        "subnet": "10.22.0.0/16",
                        "routes": [
                                { "dst": "0.0.0.0/0" }
                        ]
                }
          }
        ]
}
EOF
```

In which the CNI plugins used to configure Pod network are specified, such as the `bridge` above, as well as some network configurations, like the subnet range and routing configuration of Pod on current node.

Now let's show how to register a Pod into CNI step by step:

1. When creating a infra container via container manager, set `NetworkMode` as "None", indicating creating a unique Network Namespace owned by this infra container without any configuration.
2. Based on the corresponding PID of infra container, the directory of corresponding Network Namespace (`/proc/{pid}/ns/net`) can be obtained.
3. Invoke `SetUpPodNetwork` method of CNI Manager, the core argument is the Network Namespace directory obtained in step 2. This method in turn invokes CNI plugins specified during the initialization of CNI Manager, such as 'bridge' in above snippet, configuring Network Namespace specified in argumants, including creating various network devices, configuring network, registering this Network Namespace into corresponding CNI network.

For most Pods, network configuration can be done through the above steps and most of the work are done by CNI and corresponding plugins. But for some special Pods, whose network mode are set to "Host", that is, sharing Network Namespace with host machine. In this case, we only need to set `NetworkMode` to "Host" when creating infra container via Container Manager, and skip the configuration of CNI Manager.

For other containers in a Pod, either the Pod is in "Host" mode, or it has independent Network Namespace, we only need to set `NetworkMode` to "Container" when creating containers via Container Manager, and join the Network Namespace that infra container belongs to.

### 6. IO stream processing

Kubernetes provides features such as `kubectl exec/attach/port-forward` to enable direct interaction between the user and a specific Pod or container. The command are as follows:

```sh
aster $ kubectl exec -it shell-demo -- /bin/bash
root@shell-demo:/# ls
bin   dev  home  lib64  mnt  proc  run   srv  tmp  var
boot  etc  lib   media  opt  root  sbin  sys  usr
root@shell-demo:/#
```
As we can see, `exec` a Pod is equivalent to `ssh` logging into the container. Below, we analyze the processing of IO requests in Kubernetes in the execution flow of `kubectl exec` and the role that CRI Manager plays.



![stream-3.png | left | 827x296](https://cdn.yuque.com/lark/0/2018/png/103564/1527478375654-1c891ac5-7dd0-4432-9f72-56c4feb35ac6.png "")


As shown above, the steps to execute a kubectl exec command are as follows:

1.The essence of the `kubectl exec` command is to execute an exec command on a container in the Kubernetes cluster and forward the resulting IO stream to the users. So the request will first be forwarded to the Kubelet of the node where the container is located, and the Kubelet will then call the `Exec` interface in the CRI according to the configuration. The requested configuration parameters are as follows:

```go
type ExecRequest struct {
    ContainerId string  // execute the aimed container of exec
    Cmd []string    // specific execution of the exec command
    Tty bool    // whether to execute the exec command in a TTY
    Stdin bool  // whether to include Stdin flow
    Stdout bool // whether to include Stdout flow
    Stderr bool // whether to include Stderr flow
}
```

2.Surprisingly, CRI Manager's `Exec` method does not directly call Container Manager, and executes the exec command on the target container, but instead calls the built-in Stream Server `GetExec` method.

3.The work done by the Stream Server `GetExec` method is to save the contents of the exec request to the Request Cache shown in the figure above, and return a token. With this token, we can retrieve the corresponding exec request from the Request Cache. Finally, the token is written to a URL and returned to the ApiServer as a result layer.

4.ApiServer uses the returned URL to directly initiate a http request to the Stream Server of the node where the target container is located. The request header contains the "Upgrade" field, which is required to upgrade the http protocol to a streaming protocol such as websocket or SPDY to support multiple IOs. Stream processing, this article we take SPDY as an example.

5.The Stream Server processes the request sent by the ApiServer. First, the previously saved exec request configuration is obtained from the Request Cache according to the token in the URL. After that, it replies to the http request, agree to upgrade the protocol to SPDY, and waits for the ApiServer to create specified numbers of streams according to the configuration of the exec request, corresponding to the standard input Stdin, the standard output Stdout, and the standard error output Stderr.

6.After the Stream Server obtains the specified number of Streams, the Container Manager's `CreateExec` and `startExec` methods are invoked in turn to perform an exec operation on the target container and forward the IO stream to the corresponding stream.

7.Finally, the ApiServer forwards the data of each stream to the user, and starts the IO interaction between the user and the target container.

In fact, before the introduction of CRI, Kubernetes handled the IO in the same way as we expected. Kubelet will execute the exec command directly on the target container and forward the IO stream back to the ApiServer. But this will put Kubelet under too much pressure, and all IO streams need to be forwarded through it, which is obviously unnecessary. Therefore, although the above processing is complicated at first glance, it effectively alleviates the pressure of Kubelet and makes the processing of IO more efficient.

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEzNDg1NDEyMTUsNDIxNzA2NjU5LDQyMT
cwNjY1OSwtMTAzMDU0ODQ2LC0xMDMwNTQ4NDYsLTgyNTI5OTUy
NF19
-->