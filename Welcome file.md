
# PouchContainer Issue Guide
必读文档
[PouchContainer架构](https://github.com/alibaba/pouch/blob/master/docs/architecture.md)
[贡献者指南](https://github.com/alibaba/pouch/blob/master/CONTRIBUTING.md)
[代码风格指南](https://github.com/alibaba/pouch/blob/master/docs/contributions/code_styles.md)

## 任务
考虑到工作坊时间较紧张，PouchContainer团队为大家准备了一些单元测试用例待大家完成。下面会介绍如何为PouchContainer编写单元测试。

### 测试工具

Golang提供了方便好用的测试工具: [testing package](https://golang.org/pkg/testing/)，同时还有[go test](https://golang.org/cmd/go/#hdr-Test_packages)命令工具。

在Golang的约定中，测试代码文件名以"_test.go"结尾，测试函数名以"Test"开头，接受`*testing.T`为参数。
如

```go
// from https://golang.org/doc/code.html#Testing
package stringutil

import "testing"

func TestReverse(t *testing.T) {
    cases := []struct {
        in, want string
    }{
        {"Hello, world", "dlrow ,olleH"},
        {"Hello, 世界", "界世 ,olleH"},
        {"", ""},
    }
    for _, c := range cases {
        got := Reverse(c.in)
        if got != c.want {
            t.Errorf("Reverse(%q) == %q, want %q", c.in, got, c.want)
        }
    }
}
```

执行`go test`命令，会运行该package下的所有测试函数。

### Table Driven Test

当被测试的函数有各式各样的输入场景时，我们可以采用 Table-Driven 的形式来组织我们的测试用例，（有兴趣可以了解一下 [go tests](https://github.com/cweill/gotests) 框架）如接下来的代码所示。Table-Driven 采用数组的方式来组织测试用例，并通过循环执行的方式来验证函数的正确性。[TableDrivenTests 参考文档](https://github.com/golang/go/wiki/TableDrivenTests)

```go
// from https://golang.org/doc/code.html#Testing
package stringutil

import "testing"

func TestReverse(t *testing.T) {
    cases := []struct {
        in, want string
    }{
        {"Hello, world", "dlrow ,olleH"},
        {"Hello, 世界", "界世 ,olleH"},
        {"", ""},
    }
    for _, c := range cases {
        got := Reverse(c.in)
        if got != c.want {
            t.Errorf("Reverse(%q) == %q, want %q", c.in, got, c.want)
        }
    }
}
```

### Mock 模拟外部依赖

许多函数的运行结果是依赖于当时的环境状态的，可能依赖于存储的数据、网络状态。在测试时通常会模拟这些状态，hook真实处理方，直接返回预设的数据。

以PouchContainer的client为例，client函数执行的结果与server的状态有关，这时我们直接hook[http.Client](https://golang.org/pkg/net/http/#Client)的Transport方法，模拟了server对于client请求的响应。

```go
// https://github.com/alibaba/pouch/blob/master/client/client_mock_test.go#L12-L22
type transportFunc func(*http.Request) (*http.Response, error)

func (transFunc transportFunc) RoundTrip(req *http.Request) (*http.Response, error) {
        return transFunc(req)
}

func newMockClient(handler func(*http.Request) (*http.Response, error)) *http.Client {
        return &http.Client{
                Transport: transportFunc(handler),
        }
}

// https://github.com/alibaba/pouch/blob/master/client/image_remove_test.go
func TestImageRemove(t *testing.T) {
        expectedURL := "/images/image_id"

        httpClient := newMockClient(func(req *http.Request) (*http.Response, error) {
                if !strings.HasPrefix(req.URL.Path, expectedURL) {
                        return nil, fmt.Errorf("expected URL '%s', got '%s'", expectedURL, req.URL)
                }
                if req.Method != "DELETE" {
                        return nil, fmt.Errorf("expected DELETE method, got %s", req.Method)
                }

                return &http.Response{
                        StatusCode: http.StatusNoContent,
                        Body:       ioutil.NopCloser(bytes.NewReader([]byte(""))),
                }, nil
        })

        client := &APIClient{
                HTTPCli: httpClient,
        }

        err := client.ImageRemove(context.Background(), "image_id", false)
        if err != nil {
                t.Fatal(err)
        }
}
```

### 持续集成

为了保障PouchContainer项目的代码质量，PouchContainer的持续集成(Continuous integration)包含了代码风格检查、shellcheck、拼写检查、单元测试、集成测试等检查。在开发者提交PR后，会自动运行CI，没有通过检查的代码将不具有合入的资格。

建议开发者在本地运行相关脚本自检，以减少线上测试的负担。具体检查方法见 [.circleci/config.yml](https://github.com/alibaba/pouch/blob/master/.circleci/config.yml)

### 264期百技工作坊测试 issues

+ [add unit-test for apis/opts/portbindings.go areas/test](https://github.com/alibaba/pouch/issues/1905)
+ [add unit-test for apis/opts/ports.go areas/test](https://github.com/alibaba/pouch/issues/1906)
+ [add unit-test for updateCreateConfig areas/test](https://github.com/alibaba/pouch/issues/1916)
+ [add unit-test for toCriSandbox areas/test](https://github.com/alibaba/pouch/issues/1915)
+ [add unit-test for imageToCriImage areas/test](https://github.com/alibaba/pouch/issues/1914)
+ [add unit-test for getAppArmorSecurityOpts&getSeccompSecurityOpts areas/test](https://github.com/alibaba/pouch/issues/1913)
+ [add unit-test for getSELinuxSecurityOpts areas/test](https://github.com/alibaba/pouch/issues/1912)
+ [add unit-test for modifyContainerNamespaceOptions](https://github.com/alibaba/pouch/issues/1911)


注意：关于Volume相关的Unit Test需要Mock volume，请参考[该PR](https://github.com/alibaba/pouch/pull/1626)。
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTgzMTE3MDU3Nyw0MjE3MDY2NTksNDIxNz
A2NjU5LC0xMDMwNTQ4NDYsLTEwMzA1NDg0NiwtODI1Mjk5NTI0
XX0=
-->