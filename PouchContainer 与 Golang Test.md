# 引言

>PouchContainer 是阿里巴巴集团自主研发容器技术，同时作为一个开源项目，本着开源的精神，欢迎所有感兴趣的同学参与。为了保证代码质量，PouchContainer 中的测试过程贯穿整个开发周期，贯穿整个架构。PouchContainer 社区提倡完成或修复一个功能时，能够附带单元测试。当然，当局者迷，旁观者清，一个程序员发现自己设计的程序中的尽可能多的错误可能是有难度的，所以还要有最终的「 旁观者 」来进行最终的测试工作，PouchCointainer 有专业的测试团队维护测试代码，测试代码位于 pouch/test。
>
>Go 语言从开发初期就注意了测试用例的编写，Go 语言自带一个轻量级框架 testing , 无需做其他额外的配置，即可编写 Go 语言的测试。通过 go test 命令运行测试。（如果还不熟悉，可参考 [http://docscn.studygolang.com/pkg/testing](http://docscn.studygolang.com/pkg/testing)）。PouchContainer 作为一个 Golang 项目，本文将通过 Client 、Daemon、CRI 的测试以及集成测试来了解 Golang Test 在 PouchContainer  的实践。

# Golang Test 简介

Go 语言自带 testing 框架在某些情况下显得有些捉襟见肘， 缺乏 assert、expect、setup、teardown 等机制，于是社区涌现出了许多第三方测试库，如 [stretchr/testify](https://github.com/stretchr/testify)、[gotestyourself/gotest.tools](https://github.com/gotestyourself/gotest.tools)、[smartystreets/goconvey](https://github.com/smartystreets/goconvey)、[onsi/ginkgo](https://github.com/onsi/ginkgo)、[go-check/check](https://github.com/go-check/check) 等，作为标准库 testing 包的扩展，给我们带来了很多方便。

# 单元测试

Client 端和 Daemon 端引入 [stretchr/testify](https://github.com/stretchr/testify) 包。

## Client 测试

单元测试的原则，就是你所测试的函数方法，不要受到所依赖环境的影响，比如网络访问等。因此由于我们运行 Client 端单元测试的时候，底层依赖的基础服务会比较多， 所以使用 mock 来斩断被测试代码中的依赖不失为一种好方法。

客户端简单改写 http.Client 的实现或者说通过劫持 Client 端的发送请求，进行适当的处理，返回模拟的 resp

```
type transportFunc func(*http.Request) (*http.Response, error)

func (transFunc transportFunc) RoundTrip(req *http.Request) (*http.Response, error) {
	return transFunc(req)
}

// Transport specifies the mechanism by which individual
// HTTP requests are made.
// If nil, DefaultTransport is used.
// Transport RoundTripper
func newMockClient(handler func(*http.Request) (*http.Response, error)) *http.Client {
	return &http.Client{
		Transport: transportFunc(handler), 
	}
}

func errorMockResponse(statusCode int, message string) func(req *http.Request) (*http.Response, error) {
	return func(req *http.Request) (*http.Response, error) {
		......
		return &http.Response{
			StatusCode: statusCode,
			Body:       ioutil.NopCloser(bytes.NewReader(body)),
			Header:     header,
		}, nil
	}
}

func TestContainerCreateError(t *testing.T) {
    // 注入自定义 HTTPCli
	client := &APIClient{
		HTTPCli: newMockClient(errorMockResponse(http.StatusInternalServerError, "Server error")),
	}
	_, err := client.ContainerCreate(context.Background(), types.ContainerConfig{}, nil, nil, "nothing")
	if err == nil || !strings.Contains(err.Error(), "Server error") {
		t.Fatalf("expected a Server Error, got %v", err)
	}
}
```

可以看到，通过自定义 httpClient 的 Transport 字段，模拟 Response，并传递给 APIClient 对象，来消除对底层的依赖，并达到测试 Client 端代码的目的。所以编写 Client 端测试代码时，我们需要构造合适的 HTTPClient 以及 HTTPResponse。

## Daemon 测试

软件测试提倡每个函数多个测试用例，GoLang 对此建议使用 Table-Driven Test 编写测试用例，代码结构简单，易扩展。

这里以 TestAddDefaultRegistry 为例：

```
func TestAddDefaultRegistry(t *testing.T) {
	defaultRegistry, defaultNamespace := "pouch.io", "library"

	for _, tc := range []struct {
		repo   string
		expect string
	}{
		{
			repo:   "docker.io/library/busybox",
			expect: "docker.io/library/busybox",
		}, {
			repo:   "library/busybox",
			expect: defaultRegistry + "/library/busybox",
		}, {
			repo:   "127.0.0.1:5000/bar",
			expect: "127.0.0.1:5000/bar",
		},
		......
	} {
		assert.Equal(t, addDefaultRegistryIfMissing(tc.repo, defaultRegistry, defaultNamespace), tc.expect)
	}
}
```

# 集成测试

集成测试代码位于 pouch/test ，基于 go language 编写，使用 [go-check/check](https://github.com/go-check/check) 软件包。

Gocheck 在 testing 库之上，丰富了很多功能，增加变量检查机制、setup/teardown 机制。尤其好用的特性包括：
+ 有用的错误报告有助于解决问题（见下文）
+ 更丰富的测试助手：立即中断测试的断言，deep multi-type  对比，字符串匹配等
+ 基于 suite 的测试分组
+ 基于每个 suite 或每个测试的 set up 和 tear down
+ 管理临时文件 / 目录，自动清理
+ 恐慌追踪逻辑，具有适当的错误报告
+ 显式指定 skip 的测试
+ 与「go test」兼容
......

**TestMain**

在写测试时，有时需要在测试之前或之后进行额外的设置（setup）或拆卸（teardown）；有时，测试还需要控制在主线程上运行的代码。为了支持这些需求，testing 提供了 TestMain 函数:

```
func TestMain(m *testing.M)
```

如果测试文件中包含该函数，那么生成的测试将调用 TestMain(m)，而不是直接运行测试。TestMain 运行在主 goroutine 中 , 可以在调用 m.Run 前后做任何设置和拆卸。注意，在 TestMain 函数的最后，应该使用 m.Run 的返回值作为参数调用 os.Exit。

```
// TestMain will do initializes and run all the cases.
func TestMain(m *testing.M) {
  var err error
  commonAPIClient, err := client.NewAPIClient(environment.PouchdAddress, environment.TLSConfig)
  if err != nil {
    fmt.Printf("fail to initializes pouch API client: %s", err.Error())
    os.Exit(1)
  }
  apiClient = commonAPIClient.(*client.APIClient)
  os.Exit(m.Run())
}

// Test is the entrypoint of integration test.
func Test(t *testing.T) {
	check.TestingT(t)
}
```

更多有关集成测试，可以查看 [源码](https://github.com/alibaba/pouch/tree/master/test) 

# CRI 测试

最后，我们来聊一聊 CRI （Container Runtime Interface ) 的测试。 

PouchContainer 作为容器界新起之秀，在编排大战 Kubernetes 绝对优势胜出的情况下，毫不犹豫的选择支持 Kubernetes。而 Kubernetes 为了兼容更多的 container runtime，便制定了一套标准 CRI ，PouchContainer 目前已经基本实现了这套标准所定义的方法，并同时兼容 CRI v1alpha1 和 CRI vlalpha2 版本。 

在 CRI 的实现过程中，为了验证其正确性、可用性等，测试工作就不可避免了，目前社区维护了一个 [cri-tools](https://github.com/kubernetes-incubator/cri-tools) 工具，提供了两个组件：
+ crictl：类似于 Docker, PouchContainer 的命令行工具，不需要通过 Kubelet 就可以跟 Container Runtime 通信，可用来调试或排查问题
+ critest：CRI 的验证测试工具，用来验证新的 Container Runtime 是否实现了 CRI 需要的功能

其中 critest 使用基于 Go 语言的 BDD 测试框架 [Ginkgo](https://github.com/onsi/ginkgo) + [Gomega](https://github.com/onsi/gomega) 编写。Kubernetes 的 e2e 测试也是基于该框架。

```
var _ = framework.KubeDescribe("Container", func() {
	f := framework.NewDefaultCRIFramework()

	var rc internalapi.RuntimeService
	var ic internalapi.ImageManagerService

	BeforeEach(func() {
		rc = f.CRIClient.CRIRuntimeClient
		ic = f.CRIClient.CRIImageClient
	})

	Context("runtime should support basic operations on container", func() {
		var podID string
		var podConfig *runtimeapi.PodSandboxConfig

		BeforeEach(func() {
			podID, podConfig = framework.CreatePodSandboxForContainer(rc)
		})

		AfterEach(func() {
			By("stop PodSandbox")
			rc.StopPodSandbox(podID)
			By("delete PodSandbox")
			rc.RemovePodSandbox(podID)
		})

		It("runtime should support creating container [Conformance]", func() {
			......
		})

		......

		It("runtime should support execSync [Conformance]", func() {
			By("create container")
			containerID := framework.CreateDefaultContainer(rc, ic, podID, podConfig, "container-for-execSync-test-")

			By("start container")
			startContainer(rc, containerID)

			By("test execSync")
			cmd := []string{"echo", "hello"}
			expectedLogMessage := "hello\n"
			verifyExecSyncOutput(rc, containerID, cmd, expectedLogMessage)
		})
	})

    ......	

})

```

结合上述代码， 可以看到 Ginkgo 测试具有以下特点：
+ 使用 Describe 和 Context 组织代码
+ 使用 BeforeEach / AfterEach 执行公共操作 before / after 每一个测试用例
+ 使用 It 包含代表一个测试用例块
+ 使用 By  失败时，将打印失败之前的步骤信息，方便定位失败的步骤
+ 使用 Gomega 作为首选 matcher 库
+ 代码组织结构相当清晰
+ Fail 时问题易定位
+ 支持正则匹配 运行 / 跳过 测试用例
+ 可以指定是否打乱顺序
+ 一般用于集成测试
......

# 总结 

PouchContainer 为了保证代码的质量，对于测试工作还是给予高度重视的，无论使用哪种框架，良好的测试用例都是我们所需要的，写好测试用例对于保持高效工作至关重要，同时也是后来者了解程序的一种很好的途径。当然，过度测试也是不被提倡的，因为它可能导致需要维护的大量测试代码。

# 参考资料

http://labix.org/gocheck

https://books.studygolang.com

https://github.com/alibaba/pouch/blob/master/docs/test/test.md
