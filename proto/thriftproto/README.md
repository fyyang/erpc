## thriftproto

thriftproto is implemented thrift communication protocol.

### Message IDL

```thrift
struct payload {
    1: binary  meta,
    2: binary  xferPipe,
    3: i32     bodyCodec,
    4: binary  body
}
```

### Usage

`import "github.com/henrylee2cn/teleport/proto/thriftproto"`

#### Test

```go
package thriftproto_test

import (
	"testing"
	"time"

	tp "github.com/henrylee2cn/teleport"
	"github.com/henrylee2cn/teleport/proto/thriftproto"
	"github.com/henrylee2cn/teleport/xfer/gzip"
)

type Home struct {
	tp.CallCtx
}

func (h *Home) Test(arg *thriftproto.Test) (*thriftproto.Test, *tp.Rerror) {
	return &thriftproto.Test{
		Author: arg.Author + "->OK",
	}, nil
}

func TestTProto(t *testing.T) {
	gzip.Reg('g', "gizp-5", 5)

	// server
	srv := tp.NewPeer(tp.PeerConfig{ListenPort: 9090, DefaultBodyCodec: "thrift"})
	srv.RouteCall(new(Home))
	go srv.ListenAndServe(thriftproto.NewTProtoFunc(nil, nil))
	time.Sleep(1e9)

	// client
	cli := tp.NewPeer(tp.PeerConfig{DefaultBodyCodec: "thrift"})
	sess, err := cli.Dial(":9090", thriftproto.NewTProtoFunc(nil, nil))
	if err != nil {
		t.Error(err)
	}
	var result thriftproto.Test
	rerr := sess.Call("Home.Test",
		&thriftproto.Test{Author: "henrylee2cn"},
		&result,
		tp.WithAddMeta("peer_id", "110"),
		tp.WithXferPipe('g'),
	).Rerror()
	if rerr != nil {
		t.Error(rerr)
	}
	if result.Author != "henrylee2cn->OK" {
		t.FailNow()
	}
	t.Logf("result:%v", result)
	time.Sleep(2e9)
}
```

test command:

```sh
go test -v -run=TestTProto
```