# codec-dubbo


---


Golang dubbo library used by kitex (https://github.com/cloudwego/kitex) for **kitex-dubbo interoperability**.


## Feature List

### Type Mapping

|    thrift type     |   golang type    | hessian2 type |       java type        |
|:------------------:|:----------------:|:-------------:|:----------------------:|
|        bool        |       bool       |    boolean    |   java.lang.Boolean    |
|        byte        |       int8       |      int      |     java.lang.Byte     |
|        i16         |      int16       |      int      |    java.lang.Short     |
|        i32         |      int32       |      int      |   java.lang.Integer    |
|        i64         |      int64       |     long      |     java.lang.Long     |
|       double       |     float64      |    double     |    java.lang.Double    |
|       string       |      string      |    string     |    java.lang.String    |
|       binary       |      []byte      |    binary     |         byte[]         |
|    list\<bool>     |      []bool      |     list      |     List\<Boolean>     |
|     list\<i32>     |     []int32      |     list      |     List\<Integer>     |
|     list\<i64>     |     []int64      |     list      |      List\<Long>       |
|   list\<double>    |    []float64     |     list      |     List\<Double>      |
|   list\<string>    |     []string     |     list      |     List\<String>      |
|  map\<bool, bool>  |  map[bool]bool   |      map      | Map\<Boolean, Boolean> |
|  map\<bool, i32>   |  map[bool]int32  |      map      | Map\<Boolean, Integer> |
|  map\<bool, i64>   |  map[bool]int64  |      map      |  Map\<Boolean, Long>   |
| map\<bool, double> | map[bool]float64 |      map      | Map\<Boolean, Double>  |
| map\<bool, string> | map[bool]string  |      map      | Map\<Boolean, String>  |

**Important notes**:
1. The list of map types is not exhaustive and includes only tested cases.
Please do not use keys and values with **i8**, **i16** and **binary**.
2. Currently only the thrift type and java type mappings documented in the table are supported.
More mappings(e.g. **bool** <-> **boolean**) would be supported in the future.
Please see [issue](https://github.com/kitex-contrib/codec-dubbo/issues/46).

### Kitex-Dubbo Interoperability

1. **kitex -> dubbo**  
Writing **api.thrift** based on existing **dubbo Interface API** and [**Type Mapping Table**](#type-mapping). Then using
latest kitex command tool and thriftgo to generate stub code.
2. **dubbo -> kitex**  
Writing dubbo client code based on existing **api.thrift** and [**Type Mapping Table**](#type-mapping).

## Getting Started

[**Concrete sample**](https://github.com/kitex-contrib/codec-dubbo/tree/main/samples//helloworld/).

### Prerequisites

```shell
# compile kitex cmd tool
mkdir ~/dubbo && cd ~/dubbo
git clone https://github.com/Lvnszn/kitex.git
cd kitex
git checkout feature_idl_for_hessian
go get github.com/cloudwego/thriftgo@main
go mod tidy
cd tool/cmd/kitex
go build

# compile thriftgo
cd ~/dubbo
git clone https://github.com/cloudwego/thriftgo.git
cd thriftgo
go build
```

### Generating kitex stub codes

```shell
export PATH=~/dubbo/thriftgo:$PATH
mkdir ~/kitex-dubbo-demo && cd ~/kitex-dubbo-demo
go mod init kitex-dubbo-demo
cat > api.thrift << EOF
namespace go hello

struct GreetRequest {
    1: required string req,
}(JavaClassName="org.cloudwego.kitex.samples.api.GreetRequest")

struct GreetResponse {
    1: required string resp,
}(JavaClassName="org.cloudwego.kitex.samples.api.GreetResponse")

service GreetService {
    string Greet(1: string req)
    GreetResponse GreetWithStruct(1: GreetRequest req)
}

EOF
~/dubbo/kitex/tool/cmd/kitex/kitex -module kitex-dubbo-demo -thrift template=slim -service GreetService -protocol Hessian2 ./api.thrift
```

Important Notes:
1. JavaClassName of struct in **api.thrift** must be consistent with the target one.

### Finishing business logic and configuration

#### business logic

```go
import (
    "context"
    hello "github.com/kitex-contrib/codec-dubbo/samples/helloworld/kitex/kitex_gen/hello"
)

// implement interface in handler.go
func (s *GreetServiceImpl) Greet(ctx context.Context, req string) (resp string, err error) {
	return "Hello " + req, nil
}

func (s *GreetServiceImpl) GreetWithStruct(ctx context.Context, req *hello.GreetRequest) (resp *hello.GreetResponse, err error) {
	return &hello.GreetResponse{Resp: "Hello " + req.Req}, nil
}
```

Implement interface in **handler.go**.

#### initializing client

```go
import (
	"context"
	"github.com/cloudwego/kitex/client"
	"github.com/cloudwego/kitex/pkg/klog"
	dubbo "github.com/kitex-contrib/codec-dubbo/pkg"  
	"github.com/kitex-contrib/codec-dubbo/samples/helloworld/kitex/kitex_gen/hello"
	"github.com/kitex-contrib/codec-dubbo/samples/helloworld/kitex/kitex_gen/hello/greetservice"
)

func main() {
	cli, err := greetservice.NewClient("helloworld",
		// specify address of target server
		client.WithHostPorts("127.0.0.1:21001"),
		// configure dubbo codec
		client.WithCodec(
			dubbo.NewDubboCodec(
				// target dubbo Interface Name
				dubbo.WithJavaClassName("org.cloudwego.kitex.samples.api.GreetProvider"),
			),
		),
	)
	if err != nil {
		panic(err)
	}

	resp, err := cli.Greet(context.Background(), "world")
	if err != nil {
		klog.Error(err)
		return
	}
	klog.Infof("resp: %s", resp)
	
	respWithStruct, err := cli.GreetWithStruct(context.Background(), &hello.GreetRequest{Req: "world"})
	if err != nil {
		klog.Error(err)
		return
	}
	klog.Infof("respWithStruct: %s", respWithStruct.Resp)
}
```

Important notes:
1. Each dubbo Interface corresponds to a DubboCodec. Please do not configure multiple clients sharing a single DubboCodec.

#### initializing server

```go
import (
	"github.com/cloudwego/kitex/server"
	dubbo "github.com/kitex-contrib/codec-dubbo/pkg"
	hello "github.com/kitex-contrib/codec-dubbo/samples/helloworld/kitex/kitex_gen/hello/greetservice"
	"log"
	"net"
)

func main() {
	// server address to listen on
	addr, _ := net.ResolveTCPAddr("tcp", ":21000")
	svr := hello.NewServer(new(GreetServiceImpl),
		server.WithServiceAddr(addr),
		// configure DubboCodec
		server.WithCodec(dubbo.NewDubboCodec(
			// Interface Name of kitex service. Other dubbo clients and kitex clients can refer to this name for invocation.
			dubbo.WithJavaClassName("org.cloudwego.kitex.samples.api.GreetProvider"),
		)),
	)

	err := svr.Run()

	if err != nil {
		log.Println(err.Error())
	}
}
```

Important notes:
1. Each Interface Name corresponds to a DubboCodec. Please do not configure multiple servers sharing a single DubboCodec.

## Benchmark

### Benchmark Environment 
CPU: **Intel(R) Xeon(R) Gold 5118 CPU @ 2.30GHz**  
Memory: **192GB**

### Benchmark Code

Referring to [dubbo-go-benchmark](https://github.com/dubbogo/dubbo-go-benchmark). Converting dubbo client and dubbo server
to kitex client and kitex server. Please see [this](https://github.com/kitex-contrib/codec-dubbo/tree/main/tests/benchmark).

### Benchmark Result

#### dubbo -> dubbo

```shell
bash deploy.sh dubbo_server -p 21003
bash deploy.sh dubbo_client -p 21004 -addr "127.0.0.1:21003"
bash deploy.sh stress -addr '127.0.0.1:21004' -t 1000000 -p 100 -l 256
```

Result:  

| average rt |  tps  | success rate |
|:----------:|:-----:|:------------:|
|  2659350   | 36974 |   1.000000   |
|  2655246   | 38500 |   1.000000   |
|  2511988   | 38489 |   1.000000   |
|  2615031   | 37960 |   1.000000   |

Resource:

|   process_name    | %CPU  | %MEM |
|:-----------------:|:-----:|:----:|
| dubbo_client_main | 1807  | 0.1  |
| dubbo_server_main | 792.1 | 0.0  |
|    stress_main    | 904.6 | 0.1  |

#### kitex -> kitex

```shell
bash deploy.sh kitex_server -p 21001
bash deploy.sh kitex_client -p 21002 -addr "127.0.0.1:21001"
bash deploy.sh stress -addr '127.0.0.1:21002' -t 1000000 -p 100 -l 256
```

Result:

| average rt |  tps  | success rate |
|:----------:|:-----:|:------------:|
|  2310628   | 46015 |   1.000000   |
|  2363729   | 44202 |   1.000000   |
|  2256177   | 43280 |   1.000000   |
|  2194147   | 43171 |   1.000000   |

Resource:

|   process_name    | %CPU  | %MEM |
|:-----------------:|:-----:|:----:|
| kitex_client_main | 914.6 | 0.0  |
| kitex_server_main | 520.5 | 0.0  |
|    stress_main    | 1029  | 0.1  |

### Benchmark Summary

1. Higher TPS: **46k** vs **38.5k**, **+19.5%**
2. Low Latency: **2.36ms** vs **2.65ms**, **-11%**
3. Low CPU Usage: (server) **520%** vs **792%**, **-34.3%**

## Acknowledgements

We extend our sincere appreciation to the dubbo-go development team for their valuable contribution!
- [**dubbo-go**](https://github.com/apache/dubbo-go)
- [**dubbo-go-hessian2**](https://github.com/apache/dubbo-go-hessian2)

## reference

- [hessian serialization](http://hessian.caucho.com/doc/hessian-serialization.html)