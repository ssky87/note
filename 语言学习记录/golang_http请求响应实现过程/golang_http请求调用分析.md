<!-- TOC -->

- [golang学习纪要](#golang学习纪要)
    - [一个http连接处理流程：](#一个http连接处理流程)
        - [基本原理和开发教程：](#基本原理和开发教程)
        - [代码调用流程分析：](#代码调用流程分析)
            - [用户代码：](#用户代码)
            - [调用分析：](#调用分析)
                - [监听端口：](#监听端口)
                - [接受用户请求：](#接受用户请求)
                - [处理用户请求：](#处理用户请求)
    - [golang的http核心：Conn和ServeMux](#golang的http核心conn和servemux)
        - [Conn：](#conn)
        - [ServeMux：](#servemux)
        - [整体执行流程总结：](#整体执行流程总结)
            - [首先调用Http.HandleFunc，按顺序做了几件事：](#首先调用httphandlefunc按顺序做了几件事)
            - [其次调用http.ListenAndServe(":9090", nil)，按顺序做了几件事情：](#其次调用httplistenandserve9090-nil按顺序做了几件事情)
            - [一些使用细节：](#一些使用细节)
            - [go处理表单提交文件：](#go处理表单提交文件)
            - [模拟POST表单提交文件：](#模拟post表单提交文件)
    - [session和数据存储：](#session和数据存储)
        - [cookie分析：](#cookie分析)
            - [特点：](#特点)
            - [golang中的处理：](#golang中的处理)
                - [设置cookie：](#设置cookie)
                - [获取cookie：](#获取cookie)
                - [删除cookie：](#删除cookie)
            - [cookie原理解析：](#cookie原理解析)
        - [session分析：](#session分析)
            - [特点：](#特点-1)
            - [golang中的处理：](#golang中的处理-1)
                - [session创建流程：](#session创建流程)
                - [session的管理流程设计：](#session的管理流程设计)
                    - [session管理器和session的接口设计：](#session管理器和session的接口设计)
                    - [session的注册管理：](#session的注册管理)
                    - [session的ID生成：](#session的id生成)
                    - [session的用户绑定生成：](#session的用户绑定生成)
                    - [session的销毁：](#session的销毁)
                    - [session的GC：](#session的gc)
                - [session的内存存储实现：](#session的内存存储实现)
            - [session原理解析：](#session原理解析)
        - [参考文档：](#参考文档)

<!-- /TOC -->

# golang学习纪要
golang的学习还是需要继续进行的，作为一个技术人员，技术的提升才是最为关键的。
比较好的方法就是使用实践+代码分析的方式，这样可以快速熟悉golang提供的基本语法和功能。

## 一个http连接处理流程：
自己按照当前的工作情况，需要再次熟悉golang相关的内容，并且结合web应用开发来完成需要补全的实践过程。所以目前就当前笔记本上，需要搭建GoLang的开发环境（golang安装文件和LiteIDE，或者vscode），然后按照教程来完成开发。

### 基本原理和开发教程：
HTTP请求的原理：
> - [Web工作方式](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/03.1.md)

![http包执行流程](http包执行流程.png)

通过这个教程，完整的描述了整个web服务器的运行原理和使用golang如何按照这个原理来完成功能的过程，对于自己加深互联网的认识有非常好的作用。

> - 创建Listen Socket, 监听指定的端口, 等待客户端请求到来。
> - Listen Socket接受客户端的请求, 得到Client Socket, 接下来通过Client Socket与客户端通信。
> - 处理客户端的请求, 首先从Client Socket读取HTTP请求的协议头, 如果是POST方法, 还可能要读取客户端提交的数据, 然后交给相应的handler处理请求, handler处理完毕准备好客户端需要的数据, 通过Client Socket写给客户端。

这整个的过程里面我们只要了解清楚下面三个问题，也就知道Go是如何让Web运行起来了

> - 如何监听端口？
> - 如何接受客户端的请求？
> - 如何分配handler？

将这三个问题按照步骤解决，就能完成自己搭建一个web服务器的过程（非常喜欢这个过程，因为按照原理进行分析，然后拆分实现的步骤，最后完成整个流程，非常漂亮）。

### 代码调用流程分析：
现在结合示例代码和golang提供的基础库：http(c:\go\src\net\http\server.go)，对完整的一个HTTP请求和响应流程进行分析。


#### 用户代码：
编辑httpserver.go文件，内容为：
```golang
package main

import (
	"io"
	"net/http"
)

type helloHandler struct{}

func (h *helloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello, world!"))
}

func main() {
	http.Handle("/", &helloHandler{})
	http.ListenAndServe(":9090", nil)
}
```
用户端代码分析：
golang 的标准库 net/http 提供了 http 编程有关的接口，封装了内部TCP连接和报文解析的复杂琐碎的细节，使用者只需要和 http.request 和 http.ResponseWriter 两个对象交互就行。也就是说，我们只要写一个 handler，请求会通过参数传递进来，而它要做的就是根据请求的数据做处理，把结果写到 Response 中。
运行 go run httpserver.go，我们的服务器就会监听在本地的 9090 端口，对所有的请求都会返回 hello, world!

关键就在于：我们只要实现的一个 Handler，它的接口原型是（也就是说只要实现了 ServeHTTP 方法的对象都可以作为 Handler）：
```golang
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
然后，注册到对应的路由路径上就 OK 了。

整个流程涉及到两个函数：
> - http.HandleFunc接受两个参数：第一个参数是字符串表示的 url 路径，第二个参数是该 url 实际的处理对象。
> - http.ListenAndServe 监听在某个端口，启动服务，准备接受客户端的请求（第二个参数这里设置为 nil，这里也不要纠结什么意思，后面会有讲解）。每次客户端有请求的时候，把请求封装成 http.Request，调用对应的 handler 的 ServeHTTP 方法，然后把操作后的 http.ResponseWriter 解析，返回到客户端。

上面的代码没有什么问题，但是有一个不便：每次写 Handler 的时候，都要定义一个类型，然后编写对应的 ServeHTTP 方法，这个步骤对于所有 Handler 都是一样的。重复的工作总是可以抽象出来，net/http 也正这么做了，它提供了 http.HandleFunc 方法，允许直接把特定类型的函数作为 handler。
在http包里面还定义了一个类型HandlerFunc，这个类型默认就实现了ServeHTTP这个接口，即我们调用了HandlerFunc(f),强制类型转换f成为HandlerFunc类型，这样f就拥有了ServeHTTP方法。
上面的代码可以改成：
```golang
package main

import (
	"io"
	"net/http"
)

func helloHandler(w http.ResponseWriter, req *http.Request) {
	io.WriteString(w, "hello, world!\n")
}

func main() {
	http.HandleFunc("/", helloHandler)
	http.ListenAndServe(":9090", nil)
}
```
其实，HandleFunc 只是一个适配器：
```golang
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers. If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```
自动给 f 函数添加了 HandlerFunc 这个壳，最终调用的还是 ServerHTTP，只不过会直接使用 f(w, r)。这样封装的好处是：使用者可以专注于业务逻辑的编写，省去了很多重复的代码处理逻辑。如果只是简单的 Handler，会直接使用函数；如果是需要传递更多信息或者有复杂的操作，会使用上部分的方法。
为了更为清楚的看出到底如何调用的，还可以这样调用：
```golang
func main() {
    // 通过 HandlerFunc 把函数转换成 Handler 接口的实现对象
    hh := http.HandlerFunc(helloHandler)
	http.Handle("/", hh)
	http.ListenAndServe(":9090", nil)
}
```

参考文档：
> - [go http 服务器编程](http://cizixs.com/2016/08/17/golang-http-server-side)

#### 调用分析：
用户端代码：
```golang
package main

import (
	"fmt"
	"log"
	"net/http"
	"strings"
)

func sayhelloName(w http.ResponseWriter, r *http.Request) {
	r.ParseForm()       //解析参数，默认是不会解析的
	fmt.Println(r.Form) //这些信息是输出到服务器端的打印信息
	fmt.Println("path", r.URL.Path)
	fmt.Println("scheme", r.URL.Scheme)
	fmt.Println(r.Form["url_long"])
	for k, v := range r.Form {
		fmt.Println("key:", k)
		fmt.Println("val:", strings.Join(v, ""))
	}
	fmt.Fprintf(w, "Hello world!") //这个写入到w的是输出到客户端的
}

func main() {
	http.HandleFunc("/", sayhelloName)       //设置访问的路由
	err := http.ListenAndServe(":9090", nil) //设置监听的端口
	if err != nil {
		log.Fatal("ListenAndServe: ", err)
	}
}
```
1 用户创建一个ServerHTTP的实例，完成这个实例功能的是函数sayhelloName，也就是说通过这个函数将会对HTTP请求的响应进行处理。
2 然后用户通过ListenAndServe开启HTTP服务。

##### 监听端口：
ListenAndServer这个函数底层是通过初始化一个server对象，然后调用了net.Listen("tcp", addr)，也就是底层用TCP协议搭建了一个服务，然后监控我们设置的端口。
查看ListenAndServe这个函数的底层实现：
```golang
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```
可以看出，底层调用了server.ListenAndServe()来完成端口的监控，再进入：server.ListenAndServe()
```golang
func (srv *Server) ListenAndServe() error {
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
}
```
其中使用了：net.Listen("tcp", addr) 来对addr所指定的端口进行了监听，也就是说通过TCP协议直接进行了端口的监听（没有通过ngix等代理完成）。
上述就完成了第一步：监听端口的要求。

##### 接受用户请求：
监控之后如何接收客户端的请求呢？上面代码执行监控端口之后，调用了srv.Serve(net.Listener)函数，这个函数就是处理接收客户端的请求信息。
进入：srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})看一下：
```golang
func (srv *Server) Serve(l net.Listener) error {
	defer l.Close()
	if fn := testHookServerServe; fn != nil {
		fn(srv, l)
	}
	var tempDelay time.Duration // how long to sleep on accept failure

	if err := srv.setupHTTP2_Serve(); err != nil {
		return err
	}

	srv.trackListener(l, true)
	defer srv.trackListener(l, false)

	baseCtx := context.Background() // base is always background, per Issue 16220
	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	ctx = context.WithValue(ctx, LocalAddrContextKey, l.Addr())
	for {
		rw, e := l.Accept()
		if e != nil {
			select {
			case <-srv.getDoneChan():
				return ErrServerClosed
			default:
			}
			if ne, ok := e.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", e, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return e
		}
		tempDelay = 0
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew) // before Serve can return
		go c.serve(ctx)
	}
}
```
这个函数里面起了一个for{}，对于每一个用户发起的HTTP请求：
> - 首先通过Listener接收请求：rw, e := l.Accept()，然后进行参数的检查和设置；
> - 其次创建一个Conn：srv.newConn(rw)；
> - 最后单独开了一个goroutine，把这个请求的数据当做参数扔给这个conn去服务：go c.serve()。

这个就是高并发体现了，用户的每一次请求都是在一个新的goroutine去服务，相互不影响。
golang中的关键字go是语言提供的多线程关键字，表示将当前的执行代码开启一个新的线程，原生的支持多线程并发的开发。

##### 处理用户请求：
如何使用指定的方法来处理用户请求？我们需要进入启动的server中看conn如何对于用户请求的处理过程。
```golang
func (c *conn) serve(ctx context.Context) {
	c.remoteAddr = c.rwc.RemoteAddr().String()
	defer func() {
		if err := recover(); err != nil && err != ErrAbortHandler {
			const size = 64 << 10
			buf := make([]byte, size)
			buf = buf[:runtime.Stack(buf, false)]
			c.server.logf("http: panic serving %v: %v\n%s", c.remoteAddr, err, buf)
		}
		if !c.hijacked() {
			c.close()
			c.setState(c.rwc, StateClosed)
		}
	}()

	if tlsConn, ok := c.rwc.(*tls.Conn); ok {
		if d := c.server.ReadTimeout; d != 0 {
			c.rwc.SetReadDeadline(time.Now().Add(d))
		}
		if d := c.server.WriteTimeout; d != 0 {
			c.rwc.SetWriteDeadline(time.Now().Add(d))
		}
		if err := tlsConn.Handshake(); err != nil {
			c.server.logf("http: TLS handshake error from %s: %v", c.rwc.RemoteAddr(), err)
			return
		}
		c.tlsState = new(tls.ConnectionState)
		*c.tlsState = tlsConn.ConnectionState()
		if proto := c.tlsState.NegotiatedProtocol; validNPN(proto) {
			if fn := c.server.TLSNextProto[proto]; fn != nil {
				h := initNPNRequest{tlsConn, serverHandler{c.server}}
				fn(c.server, tlsConn, h)
			}
			return
		}
	}

	// HTTP/1.x from here on.

	ctx, cancelCtx := context.WithCancel(ctx)
	c.cancelCtx = cancelCtx
	defer cancelCtx()

	c.r = &connReader{conn: c}
	c.bufr = newBufioReader(c.r)
	c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)

	for {
		w, err := c.readRequest(ctx)
		if c.r.remain != c.server.initialReadLimitSize() {
			// If we read any bytes off the wire, we're active.
			c.setState(c.rwc, StateActive)
		}
		if err != nil {
			const errorHeaders = "\r\nContent-Type: text/plain; charset=utf-8\r\nConnection: close\r\n\r\n"

			if err == errTooLarge {
				// Their HTTP client may or may not be
				// able to read this if we're
				// responding to them and hanging up
				// while they're still writing their
				// request. Undefined behavior.
				const publicErr = "431 Request Header Fields Too Large"
				fmt.Fprintf(c.rwc, "HTTP/1.1 "+publicErr+errorHeaders+publicErr)
				c.closeWriteAndWait()
				return
			}
			if isCommonNetReadError(err) {
				return // don't reply
			}

			publicErr := "400 Bad Request"
			if v, ok := err.(badRequestError); ok {
				publicErr = publicErr + ": " + string(v)
			}

			fmt.Fprintf(c.rwc, "HTTP/1.1 "+publicErr+errorHeaders+publicErr)
			return
		}

		// Expect 100 Continue support
		req := w.req
		if req.expectsContinue() {
			if req.ProtoAtLeast(1, 1) && req.ContentLength != 0 {
				// Wrap the Body reader with one that replies on the connection
				req.Body = &expectContinueReader{readCloser: req.Body, resp: w}
			}
		} else if req.Header.get("Expect") != "" {
			w.sendExpectationFailed()
			return
		}

		c.curReq.Store(w)

		if requestBodyRemains(req.Body) {
			registerOnHitEOF(req.Body, w.conn.r.startBackgroundRead)
		} else {
			if w.conn.bufr.Buffered() > 0 {
				w.conn.r.closeNotifyFromPipelinedRequest()
			}
			w.conn.r.startBackgroundRead()
		}

		// HTTP cannot have multiple simultaneous active requests.[*]
		// Until the server replies to this request, it can't read another,
		// so we might as well run the handler in this goroutine.
		// [*] Not strictly true: HTTP pipelining. We could let them all process
		// in parallel even if their responses need to be serialized.
		// But we're not going to implement HTTP pipelining because it
		// was never deployed in the wild and the answer is HTTP/2.
		serverHandler{c.server}.ServeHTTP(w, w.req)
		w.cancelCtx()
		if c.hijacked() {
			return
		}
		w.finishRequest()
		if !w.shouldReuseConnection() {
			if w.requestBodyLimitHit || w.closedRequestBodyEarly() {
				c.closeWriteAndWait()
			}
			return
		}
		c.setState(c.rwc, StateIdle)
		c.curReq.Store((*response)(nil))

		if !w.conn.server.doKeepAlives() {
			// We're in shutdown mode. We might've replied
			// to the user without "Connection: close" and
			// they might think they can send another
			// request, but such is life with HTTP/1.1.
			return
		}

		if d := c.server.idleTimeout(); d != 0 {
			c.rwc.SetReadDeadline(time.Now().Add(d))
			if _, err := c.bufr.Peek(4); err != nil {
				return
			}
		}
		c.rwc.SetReadDeadline(time.Time{})
	}
}
```
conn首先会解析request：w, err := c.readRequest(ctx)
然后获取相应的handler：serverHandler{c.server}.ServeHTTP(w, w.req)，也就是我们刚才在调用函数ListenAndServe时候的第二个参数，我们前面例子传递的是nil，也就是为空。
进入ServeHTTP(w, w.req)：
```golang
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	if req.RequestURI == "*" && req.Method == "OPTIONS" {
		handler = globalOptionsHandler{}
	}
	handler.ServeHTTP(rw, req)
}
```
因为传入参数为空，那么默认获取handler = DefaultServeMux，这个变量就是一个路由器，它用来匹配url跳转到其相应的handle函数。我们调用的代码里面第一句不是调用了http.HandleFunc("/", sayhelloName)嘛。这个作用就是注册了请求/的路由规则，当请求uri为"/"，路由就会转到函数sayhelloName，DefaultServeMux会调用ServeHTTP方法，这个方法内部其实就是调用sayhelloName本身，最后通过写入response的信息反馈到客户端。

完整的调用流程图为：

![一个http连接golang的处理流程](一个http连接golang的处理流程.png)


## golang的http核心：Conn和ServeMux
也就是接受用户HTTP请求的链接和处理用户请求的路由器。

### Conn：
Go为了实现高并发和高性能, 使用了goroutines来处理Conn的读写事件, 这样每个请求都能保持独立，相互不会阻塞，可以高效的响应网络事件。这是Go高效的保证。
在等待用户请求的代码中，使用for循环获取用户请求，然后对每一个请求新建Conn。这个Conn里面保存了该次请求的信息，然后再传递到对应的handler，该handler中便可以读取到相应的header信息，这样保证了每个请求的独立性。

### ServeMux：
Conn通过调用server方法，将用户的HTTP请求通过路由器把本次请求的信息传递到了后端的处理函数。如果没有设置路由器，就会使用golang内置的默认路由器进行处理。
那么该如何自定义路由器？首先看看ServerMux这个结构体：
```golang
type ServeMux struct {
	mu    sync.RWMutex          //锁，由于请求涉及到并发处理，因此这里需要一个锁机制
	m     map[string]muxEntry   // 路由规则，一个string对应一个mux实体，这里的string就是注册的路由表达式
	hosts bool                  // 是否在任意的规则中带有host信息
}
```
> 注意：golang中 map 对象的声明形式为 map[TypeOfKey]TypeOfValue，所以上述结构体类型中的m为map类型，key的类型为string，value的类型为muxEntry。

然后muxEntry结构定义为：
```golang
type muxEntry struct {
	explicit bool       // 是否精确匹配
	h        Handler    // 这个路由表达式对应哪个handler
	pattern  string     // 匹配字符串
}
```
然后查看Handler的定义为：
```golang
type Handler interface {
	ServeHTTP(ResponseWriter, *Request) // 路由实现器
}
```
Handler是一个接口，路由器的功能需要实现这个接口。
在http包里面还定义了一个类型HandlerFunc，也就是上述分析中看到的强制类型转换函数：
```golang
// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```
这个类型默认就实现了ServeHTTP这个接口，即我们调用了HandlerFunc(f)，强制类型转换f成为HandlerFunc类型，这样f就拥有了ServeHTTP方法。

路由器里面存储好了相应的路由规则之后，那么具体的请求又是怎么分发的呢？请看下面的代码，默认的路由器实现了ServeHTTP：
```golang
// ServeHTTP dispatches the request to the handler whose
// pattern most closely matches the request URL.
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
	if r.RequestURI == "*" {
		if r.ProtoAtLeast(1, 1) {
			w.Header().Set("Connection", "close")
		}
		w.WriteHeader(StatusBadRequest)
		return
	}
	h, _ := mux.Handler(r)
	h.ServeHTTP(w, r)
}
```
如上所示路由器接收到请求之后，如果是*那么关闭链接，不然调用mux.Handler(r)返回对应设置路由的处理Handler，然后执行h.ServeHTTP(w, r)。
也就是调用对应路由的handler的ServerHTTP接口，那么mux.Handler(r)怎么处理的呢？
```golang
// Handler returns the handler to use for the given request,
// consulting r.Method, r.Host, and r.URL.Path. It always returns
// a non-nil handler. If the path is not in its canonical form, the
// handler will be an internally-generated handler that redirects
// to the canonical path.
//
// Handler also returns the registered pattern that matches the
// request or, in the case of internally-generated redirects,
// the pattern that will match after following the redirect.
//
// If there is no registered handler that applies to the request,
// Handler returns a ``page not found'' handler and an empty pattern.
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
	if r.Method != "CONNECT" {
		if p := cleanPath(r.URL.Path); p != r.URL.Path {
			_, pattern = mux.handler(r.Host, p)
			url := *r.URL
			url.Path = p
			return RedirectHandler(url.String(), StatusMovedPermanently), pattern
		}
	}

	return mux.handler(r.Host, r.URL.Path)
}

// handler is the main implementation of Handler.
// The path is known to be in canonical form, except for CONNECT methods.
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
	mux.mu.RLock()
	defer mux.mu.RUnlock()

	// Host-specific pattern takes precedence over generic ones
	if mux.hosts {
		h, pattern = mux.match(host + path)
	}
	if h == nil {
		h, pattern = mux.match(path)
	}
	if h == nil {
		h, pattern = NotFoundHandler(), ""
	}
	return
}
```
原来他是根据用户请求的URL和路由器里面存储的map去匹配的，当匹配到之后返回存储的handler，调用这个handler的ServeHTTP接口就可以执行到相应的函数了。

然后继续看看Handler 函数接受的两个参数：http.Request 和 http.ResponseWriter。
Request 就是封装好的客户端请求，包括 URL，method，header 等等所有信息，以及一些方便使用的方法：
```golang
// A Request represents an HTTP request received by a server
// or to be sent by a client.
//
// The field semantics differ slightly between client and server
// usage. In addition to the notes on the fields below, see the
// documentation for Request.Write and RoundTripper.
type Request struct {
	// Method specifies the HTTP method (GET, POST, PUT, etc.).
	// For client requests an empty string means GET.
	Method string

	// URL specifies either the URI being requested (for server
	// requests) or the URL to access (for client requests).
	//
	// For server requests the URL is parsed from the URI
	// supplied on the Request-Line as stored in RequestURI.  For
	// most requests, fields other than Path and RawQuery will be
	// empty. (See RFC 2616, Section 5.1.2)
	//
	// For client requests, the URL's Host specifies the server to
	// connect to, while the Request's Host field optionally
	// specifies the Host header value to send in the HTTP
	// request.
	URL *url.URL

	// The protocol version for incoming server requests.
	//
	// For client requests these fields are ignored. The HTTP
	// client code always uses either HTTP/1.1 or HTTP/2.
	// See the docs on Transport for details.
	Proto      string // "HTTP/1.0"
	ProtoMajor int    // 1
	ProtoMinor int    // 0

	// Header contains the request header fields either received
	// by the server or to be sent by the client.
	//
	// If a server received a request with header lines,
	//
	//	Host: example.com
	//	accept-encoding: gzip, deflate
	//	Accept-Language: en-us
	//	fOO: Bar
	//	foo: two
	//
	// then
	//
	//	Header = map[string][]string{
	//		"Accept-Encoding": {"gzip, deflate"},
	//		"Accept-Language": {"en-us"},
	//		"Foo": {"Bar", "two"},
	//	}
	//
	// For incoming requests, the Host header is promoted to the
	// Request.Host field and removed from the Header map.
	//
	// HTTP defines that header names are case-insensitive. The
	// request parser implements this by using CanonicalHeaderKey,
	// making the first character and any characters following a
	// hyphen uppercase and the rest lowercase.
	//
	// For client requests, certain headers such as Content-Length
	// and Connection are automatically written when needed and
	// values in Header may be ignored. See the documentation
	// for the Request.Write method.
	Header Header

	// Body is the request's body.
	//
	// For client requests a nil body means the request has no
	// body, such as a GET request. The HTTP Client's Transport
	// is responsible for calling the Close method.
	//
	// For server requests the Request Body is always non-nil
	// but will return EOF immediately when no body is present.
	// The Server will close the request body. The ServeHTTP
	// Handler does not need to.
	Body io.ReadCloser

	// GetBody defines an optional func to return a new copy of
	// Body. It is used for client requests when a redirect requires
	// reading the body more than once. Use of GetBody still
	// requires setting Body.
	//
	// For server requests it is unused.
	GetBody func() (io.ReadCloser, error)

	// ContentLength records the length of the associated content.
	// The value -1 indicates that the length is unknown.
	// Values >= 0 indicate that the given number of bytes may
	// be read from Body.
	// For client requests, a value of 0 with a non-nil Body is
	// also treated as unknown.
	ContentLength int64

	// TransferEncoding lists the transfer encodings from outermost to
	// innermost. An empty list denotes the "identity" encoding.
	// TransferEncoding can usually be ignored; chunked encoding is
	// automatically added and removed as necessary when sending and
	// receiving requests.
	TransferEncoding []string

	// Close indicates whether to close the connection after
	// replying to this request (for servers) or after sending this
	// request and reading its response (for clients).
	//
	// For server requests, the HTTP server handles this automatically
	// and this field is not needed by Handlers.
	//
	// For client requests, setting this field prevents re-use of
	// TCP connections between requests to the same hosts, as if
	// Transport.DisableKeepAlives were set.
	Close bool

	// For server requests Host specifies the host on which the
	// URL is sought. Per RFC 2616, this is either the value of
	// the "Host" header or the host name given in the URL itself.
	// It may be of the form "host:port". For international domain
	// names, Host may be in Punycode or Unicode form. Use
	// golang.org/x/net/idna to convert it to either format if
	// needed.
	//
	// For client requests Host optionally overrides the Host
	// header to send. If empty, the Request.Write method uses
	// the value of URL.Host. Host may contain an international
	// domain name.
	Host string

	// Form contains the parsed form data, including both the URL
	// field's query parameters and the POST or PUT form data.
	// This field is only available after ParseForm is called.
	// The HTTP client ignores Form and uses Body instead.
	Form url.Values

	// PostForm contains the parsed form data from POST, PATCH,
	// or PUT body parameters.
	//
	// This field is only available after ParseForm is called.
	// The HTTP client ignores PostForm and uses Body instead.
	PostForm url.Values

	// MultipartForm is the parsed multipart form, including file uploads.
	// This field is only available after ParseMultipartForm is called.
	// The HTTP client ignores MultipartForm and uses Body instead.
	MultipartForm *multipart.Form

	// Trailer specifies additional headers that are sent after the request
	// body.
	//
	// For server requests the Trailer map initially contains only the
	// trailer keys, with nil values. (The client declares which trailers it
	// will later send.)  While the handler is reading from Body, it must
	// not reference Trailer. After reading from Body returns EOF, Trailer
	// can be read again and will contain non-nil values, if they were sent
	// by the client.
	//
	// For client requests Trailer must be initialized to a map containing
	// the trailer keys to later send. The values may be nil or their final
	// values. The ContentLength must be 0 or -1, to send a chunked request.
	// After the HTTP request is sent the map values can be updated while
	// the request body is read. Once the body returns EOF, the caller must
	// not mutate Trailer.
	//
	// Few HTTP clients, servers, or proxies support HTTP trailers.
	Trailer Header

	// RemoteAddr allows HTTP servers and other software to record
	// the network address that sent the request, usually for
	// logging. This field is not filled in by ReadRequest and
	// has no defined format. The HTTP server in this package
	// sets RemoteAddr to an "IP:port" address before invoking a
	// handler.
	// This field is ignored by the HTTP client.
	RemoteAddr string

	// RequestURI is the unmodified Request-URI of the
	// Request-Line (RFC 2616, Section 5.1) as sent by the client
	// to a server. Usually the URL field should be used instead.
	// It is an error to set this field in an HTTP client request.
	RequestURI string

	// TLS allows HTTP servers and other software to record
	// information about the TLS connection on which the request
	// was received. This field is not filled in by ReadRequest.
	// The HTTP server in this package sets the field for
	// TLS-enabled connections before invoking a handler;
	// otherwise it leaves the field nil.
	// This field is ignored by the HTTP client.
	TLS *tls.ConnectionState

	// Cancel is an optional channel whose closure indicates that the client
	// request should be regarded as canceled. Not all implementations of
	// RoundTripper may support Cancel.
	//
	// For server requests, this field is not applicable.
	//
	// Deprecated: Use the Context and WithContext methods
	// instead. If a Request's Cancel field and context are both
	// set, it is undefined whether Cancel is respected.
	Cancel <-chan struct{}

	// Response is the redirect response which caused this request
	// to be created. This field is only populated during client
	// redirects.
	Response *Response

	// ctx is either the client or server context. It should only
	// be modified via copying the whole Request using WithContext.
	// It is unexported to prevent people from using Context wrong
	// and mutating the contexts held by callers of the same request.
	ctx context.Context
}
```
Handler 需要知道关于请求的任何信息，都要从这个对象中获取，除非是做一些用户请求的修改等内容，一般不会直接修改这个对象。

ResponseWriter 是一个接口，定义了三个方法：
```golang
// A ResponseWriter interface is used by an HTTP handler to
// construct an HTTP response.
//
// A ResponseWriter may not be used after the Handler.ServeHTTP method
// has returned.
type ResponseWriter interface {
	// Header returns the header map that will be sent by
	// WriteHeader. The Header map also is the mechanism with which
	// Handlers can set HTTP trailers.
	//
	// Changing the header map after a call to WriteHeader (or
	// Write) has no effect unless the modified headers are
	// trailers.
	//
	// There are two ways to set Trailers. The preferred way is to
	// predeclare in the headers which trailers you will later
	// send by setting the "Trailer" header to the names of the
	// trailer keys which will come later. In this case, those
	// keys of the Header map are treated as if they were
	// trailers. See the example. The second way, for trailer
	// keys not known to the Handler until after the first Write,
	// is to prefix the Header map keys with the TrailerPrefix
	// constant value. See TrailerPrefix.
	//
	// To suppress implicit response headers (such as "Date"), set
	// their value to nil.
	Header() Header

	// Write writes the data to the connection as part of an HTTP reply.
	//
	// If WriteHeader has not yet been called, Write calls
	// WriteHeader(http.StatusOK) before writing the data. If the Header
	// does not contain a Content-Type line, Write adds a Content-Type set
	// to the result of passing the initial 512 bytes of written data to
	// DetectContentType.
	//
	// Depending on the HTTP protocol version and the client, calling
	// Write or WriteHeader may prevent future reads on the
	// Request.Body. For HTTP/1.x requests, handlers should read any
	// needed request body data before writing the response. Once the
	// headers have been flushed (due to either an explicit Flusher.Flush
	// call or writing enough data to trigger a flush), the request body
	// may be unavailable. For HTTP/2 requests, the Go HTTP server permits
	// handlers to continue to read the request body while concurrently
	// writing the response. However, such behavior may not be supported
	// by all HTTP/2 clients. Handlers should read before writing if
	// possible to maximize compatibility.
	Write([]byte) (int, error)

	// WriteHeader sends an HTTP response header with status code.
	// If WriteHeader is not called explicitly, the first call to Write
	// will trigger an implicit WriteHeader(http.StatusOK).
	// Thus explicit calls to WriteHeader are mainly used to
	// send error codes.
	WriteHeader(int)
}
```
Golang 中是没有类class 的概念，golang是通过 interface 类型接口实现的继承多态的效果。
interface就是一组抽象方法的集合，它必须由其他非interface类型实现，而不能自我实现， Go通过interface实现了duck-typing:即"当看到一只鸟走起来像鸭子、游泳起来像鸭子、叫起来也像鸭子，那么这只鸟就可以被称为鸭子"。

> 参考文档：
> - [golang_Interface](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/02.6.md)

我们看看这三个方法：
> - Header()：返回一个 Header 对象，可以通过它的 Set() 方法设置头部，注意最终返回的头部信息可能和你写进去的不完全相同，因为后续处理还可能修改头部的值（比如设置 Content-Length、Content-type 等操作）
> - Write()： 写 response 的主体部分，比如 html 或者 json 的内容就是放到这里的
> - WriteHeader()：设置 status code，如果没有调用这个函数，默认设置为 http.StatusOK，就是200状态码

实际上反馈的内容为：
```golang
// A response represents the server side of an HTTP response.
type response struct {
	conn             *conn
	req              *Request // request for this response
	reqBody          io.ReadCloser
	cancelCtx        context.CancelFunc // when ServeHTTP exits
	wroteHeader      bool               // reply header has been (logically) written
	wroteContinue    bool               // 100 Continue response was written
	wants10KeepAlive bool               // HTTP/1.0 w/ Connection "keep-alive"
	wantsClose       bool               // HTTP request has Connection "close"

	w  *bufio.Writer // buffers output in chunks to chunkWriter
	cw chunkWriter

	// handlerHeader is the Header that Handlers get access to,
	// which may be retained and mutated even after WriteHeader.
	// handlerHeader is copied into cw.header at WriteHeader
	// time, and privately mutated thereafter.
	handlerHeader Header
	calledHeader  bool // handler accessed handlerHeader via Header

	written       int64 // number of bytes written in body
	contentLength int64 // explicitly-declared Content-Length; or -1
	status        int   // status code passed to WriteHeader

	// close connection after this reply.  set on request and
	// updated after response from handler if there's a
	// "Connection: keep-alive" response header and a
	// Content-Length.
	closeAfterReply bool

	// requestBodyLimitHit is set by requestTooLarge when
	// maxBytesReader hits its max size. It is checked in
	// WriteHeader, to make sure we don't consume the
	// remaining request body to try to advance to the next HTTP
	// request. Instead, when this is set, we stop reading
	// subsequent requests on this connection and stop reading
	// input from it.
	requestBodyLimitHit bool

	// trailers are the headers to be sent after the handler
	// finishes writing the body. This field is initialized from
	// the Trailer response header when the response header is
	// written.
	trailers []string

	handlerDone atomicBool // set true when the handler exits

	// Buffers for Date and Content-Length
	dateBuf [len(TimeFormat)]byte
	clenBuf [10]byte

	// closeNotifyCh is the channel returned by CloseNotify.
	// TODO(bradfitz): this is currently (for Go 1.8) always
	// non-nil. Make this lazily-created again as it used to be?
	closeNotifyCh  chan bool
	didCloseNotify int32 // atomic (only 0->1 winner should send)
}
```
实现了上面提到的三个方法，所以可以作为参数交给Handler作为参数使用。



### 整体执行流程总结：
对http包的分析之后，现在让我们来梳理一下整个的代码执行过程：

#### 首先调用Http.HandleFunc，按顺序做了几件事：
> - 1 调用了DefaultServeMux的HandleFunc
> - 2 调用了DefaultServeMux的Handle
> - 3 往DefaultServeMux的map[string]muxEntry中增加对应的handler和路由规则

#### 其次调用http.ListenAndServe(":9090", nil)，按顺序做了几件事情：

> - 1 实例化Server
> - 2 调用Server的ListenAndServe()
> - 3 调用net.Listen("tcp", addr)监听端口
> - 4 启动一个for循环，在循环体中Accept请求
> - 5 对每个请求实例化一个Conn，并且开启一个goroutine为这个请求进行服务go c.serve()
> - 6 读取每个请求的内容w, err := c.readRequest()
> - 7 判断handler是否为空，如果没有设置handler（这个例子就没有设置handler），handler就设置为DefaultServeMux
> - 8 调用handler的ServeHttp
> - 9 在这个例子中，下面就进入到DefaultServeMux.ServeHttp
> - 10 根据request选择handler，并且进入到这个handler的ServeHTTP：mux.handler(r).ServeHTTP(w, r)
> - 11 选择handler：<br> A 判断是否有路由能满足这个request（循环遍历ServerMux的muxEntry）<br> B 如果有路由满足，调用这个路由handler的ServeHttp <br> C 如果没有路由满足，调用NotFoundHandler的ServeHttp

#### 一些使用细节：
设置response的返回类型：
[HTTP Response Snippets for Go](http://www.alexedwards.net/blog/golang-response-snippets)

json转换为字符串的正则替换：
" 替换为 \"
[ \n]+ 替换为空

if else结构中，if和else之间的花括号中不能插入注释，否则无法识别。
```golang
if err != nil {
    fmt.Println("wrong password")
} else {
    //接下来就可以判断这个数字的大小范围了
    fmt.Println(getint)
}
```
不能加入括号的原因在于：Golang 编译器会在源代码中自动添加分号 ;，如果 { 单独放在一行的开头，其前面可能会被加入 ;，造成编译错误。
官方文档：[Semicolons](https://golang.org/ref/spec#Semicolons)
并且不是因为：} 和 else { 之间不能插入注释，换行也是会造成编译错误：syntax error: unexpected else, expecting }
因为如果这样做，将会在大括号的前方插入一个分号，这可能导致出现不想要的结果。
具体可以参考：[GO 语言简介（上）— 语法](http://coolshell.cn/articles/8460.html)



golang中使用了re2作为正则表达式的引擎：[RE2](https://github.com/google/re2)，并且默认支持UTF-8编码。
golang提供了对中文的良好支持，对于中文的检测可以使用如下正则：
```golang
m, _ := regexp.MatchString("^\\p{Han}+$", r.Form.Get("realname"))
```

range关键字使用起来非常的方便，它允许你遍历某个slice或者map，并通过两个参数(index和value)，分别获取到slice或者map中某个元素所在的index以及其值。
和for循环结合可以使用：
```golang
for index, value := range mySlice {
    fmt.Println("index: " + index)
    fmt.Println("value: " + value)
}
```
参考文档：[聊聊Go中的Range关键字](https://xiaozhou.net/something-about-range-of-go-2016-04-10.html)

要区分数组和切片，尤其是定义的时候：
```golang
arr4 := []int{1, 2, 3}
```
这个定义了一个slice，而不是一个数组，因为数组是固定长度的，必须在方括号中写明长度或者三个点号，表示用值的个数来确定数组长度。
与数组相比切片的长度是不固定的，可以追加元素，在追加时可能使切片的容量增大。与数组相比切片的长度是不固定的，可以追加元素，在追加时可能使切片的容量增大。


关于使用md5加密字符串:
1 md5加密的方式有两种：固定长度返回和不固定长度返回（切片），参考文档：[golang中的md5的用法](http://blog.haohtml.com/archives/16839)
第一种加密方法所调用的函数：
```golang
// The size of an MD5 checksum in bytes.
const Size = 16

................

// Sum returns the MD5 checksum of the data.
func Sum(data []byte) [Size]byte {
	var d digest
	d.Reset()
	d.Write(data)
	return d.checkSum()
}
```
其 [Size]byte 是固定死的.所以说第一种方法返回的是 16长度的数组(无法转string类型，需使用第二种加密方法)

第二种加密方法:
使用 func New() hash.Hash {} 函数进行生成对象.
使用 func (d *digest) Write(p []byte) (nn int, err error) {} 方法进行写入要加密的数据.
使用 func (d0 *digest) Sum(in []byte) []byte {} 方法进行数据的加密 看其返回值.
[]byte 可见使用第二种方式加密返回的是 []byte 类型的切片.
```golang
// New returns a new hash.Hash computing the MD5 checksum.
func New() hash.Hash {
	d := new(digest)
	d.Reset()
	return d
}
// 这里只放了函数签名部分, 关于函数具体内容这里就不详细复制了.
func New() hash.Hash {}
func (d *digest) Write(p []byte) (nn int, err error) {}
func (d0 *digest) Sum(in []byte) []byte {}
```

2 涉及到的IO问题：参考文档：[Golang学习 - io 包](http://www.cnblogs.com/golove/p/3276678.html)
func WriteString(w Writer, s string) (n int, err error)
功能描述为：
WriteString 将字符串 s 写入到 w 中，返回写入的字节数和遇到的错误。
如果 w 实现了 WriteString 方法，则优先使用该方法将 s 写入 w 中。
否则，将 s 转换为 []byte，然后调用 w.Write 方法将数据写入 w 中

#### go处理表单提交文件：
概念学习：[细说 Form (表单)](http://www.cnblogs.com/fish-li/archive/2011/07/17/2108884.html)

要使表单能够上传文件，首先第一步就是要添加form的enctype属性，enctype属性有如下三种情况:
> - application/x-www-form-urlencoded   表示在发送前编码所有字符（默认）
> - multipart/form-data	  不对字符编码。在使用包含文件上传控件的表单时，必须使用该值。
> - text/plain	  空格转换为 "+" 加号，但不对特殊字符编码。

所以，新建upload.gtpl文件中的内容为：
```html
<html>
	<head>
	    <title></title>
	</head>
	<body>
	    <form enctype="multipart/form-data" action="/upload" method="post">
		    <input type="file" name="uploadfile" />
	  		<input type="hidden" name="token" value="{{.}}"/>
	  		<input type="submit" value="upload" />
	    </form>
	</body>
</html>
```
然后编写一个处理文件上传操作的函数：
```golang
func uploadServer(w http.ResponseWriter, r *http.Request) {
	fmt.Println("method:", r.Method) //获取请求的方法
	if r.Method == "GET" {
		crutime := time.Now().Unix()
		h := md5.New()
		io.WriteString(h, strconv.FormatInt(crutime, 10))
		token := fmt.Sprintf("%x", h.Sum(nil))

		t, _ := template.ParseFiles("upload.gtpl")
		t.Execute(w, token)
	} else {
		r.ParseMultipartForm(32 << 20)
		file, handler, err := r.FormFile("uploadfile")
		if err != nil {
			fmt.Println(err)
			return
		}
		defer file.Close()
		fmt.Fprintf(w, "%v", handler.Header)
		f, err := os.OpenFile("./test/"+handler.Filename, os.O_WRONLY|os.O_CREATE, 0666) // 此处假设当前目录下已存在test目录
		if err != nil {
			fmt.Println(err)
			return
		}
		defer f.Close()
		io.Copy(f, file)
	}
}
```
处理表单上传的文件，我们需要调用r.ParseMultipartForm，里面的参数表示maxMemory，调用ParseMultipartForm之后，上传的文件存储在maxMemory大小的内存里面，如果文件大小超过了maxMemory，那么剩下的部分将存储在系统的临时文件中。
上面的参数设置为32左移20位，也就是2^12byte

上传文件主要三步处理：
> - 表单中增加enctype="multipart/form-data"
> - 服务端调用r.ParseMultipartForm,把上传的文件存储在内存和临时文件中
> - 使用r.FormFile获取文件句柄，然后对文件进行存储等处理。

这儿从表单中通过函数FormFile获取文件，返回Handler是一个multipart.FileHeader指针，这个结构体定义为：

定义为：
```golang
// A FileHeader describes a file part of a multipart request.
type FileHeader struct {
	Filename string
	Header   textproto.MIMEHeader

	content []byte
	tmpfile string
}
```
执行上述upload请求，返回结果为：
```shell
map[Content-Disposition:[form-data; name="uploadfile"; filename="产品简介.zip"] Content-Type:[application/x-zip-compressed]]
```
也就是返回的内容，可以和HTTP请求的返回做一个对比：
```shell
map[Content-Disposition:[form-data; name="uploadfile"; filename="产品简介.zip"] Content-Type:[application/x-zip-compressed]]
```
可以看出是完全一样的。

#### 模拟POST表单提交文件：
我们还可以通过程序来模拟网页的POST表单提交，这个在测试HTTP接口的时候非常有帮助。
具体的函数为：
```golang
func postFile(filename string, targetUrl string) error {
	bodyBuf := &bytes.Buffer{}
	bodyWriter := multipart.NewWriter(bodyBuf)

	//关键的一步操作
	fileWriter, err := bodyWriter.CreateFormFile("uploadfile", filename)
	if err != nil {
		fmt.Println("error writing to buffer")
		return err
	}

	//打开文件句柄操作
	fh, err := os.Open(filename)
	if err != nil {
		fmt.Println("error opening file")
		return err
	}
	defer fh.Close()

	//iocopy
	_, err = io.Copy(fileWriter, fh)
	if err != nil {
		return err
	}

	contentType := bodyWriter.FormDataContentType()
	bodyWriter.Close()

	resp, err := http.Post(targetUrl, contentType, bodyBuf)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	resp_body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		return err
	}
	fmt.Println(resp.Status)
	fmt.Println(string(resp_body))
	return nil
}
```
首先，申请了一个缓存：
bodyBuf := &bytes.Buffer{}
这儿使用了Slice确保是一个可变长度的字节容器，参考：[Builtin append vs. bytes.Buffer write](http://stackoverflow.com/questions/39319024/builtin-append-vs-bytes-buffer-write)

然后，使用NewWriter函数返回一个设定了一个随机边界的Writer，将获取的上传文件的数据写入到申请的缓存中：
bodyWriter := multipart.NewWriter(bodyBuf)

接着，CreateFormFile是CreatePart方法的包装， 使用给出的属性名和文件名创建一个新的form-data头：
fileWriter, err := bodyWriter.CreateFormFile("uploadfile", filename)

接着，使用系统函数打开要上传的文件缓存：
fh, err := os.Open(filename)
if err != nil {
	fmt.Println("error opening file")
	return err
}
defer fh.Close()
这儿对于资源的操作都有打开和关闭处理，使用了defer关键字。
defer后面的表达式会被放入一个列表中（这个list可以看作是一个栈(stack)的结构，是一个后进先出的栈。），在当前方法返回的时候，列表中的表达式就会被执行。一个方法中可以在一个或者多个地方使用defer表达式
也就是如果文件打开错误，执行return err的时候，才会调用defer保存的内容，确保资源文件能够正确的被关闭，然后最终返回。
参考：[Golang中defer的那些事](https://xiaozhou.net/something-about-defer-2014-05-25.html)
可以编写defer的三个特性的测试用例：
```golang
// defer表达式中变量的值在defer表达式被定义时就已经明确
func defer_test_1() {
    i := 0
    defer fmt.Println(i)
    i++
    return
}

// defer表达式的调用顺序是按照先进后出的方式
func defer_test_2() {
    defer fmt.Print(1)
    defer fmt.Print(2)
    defer fmt.Print(3)
    defer fmt.Print(4)
}

// defer表达式中可以修改函数中的命名返回值
func defer_test_3() (i int) {
    defer func() { i++ }()
    return 1
}
```

接着，对正确打开的文件内容进行拷贝：
_, err = io.Copy(fileWriter, fh)
函数原型为：
```golang
// Copy copies from src to dst until either EOF is reached
// on src or an error occurs. It returns the number of bytes
// copied and the first error encountered while copying, if any.
//
// A successful Copy returns err == nil, not err == EOF.
// Because Copy is defined to read from src until EOF, it does
// not treat an EOF from Read as an error to be reported.
//
// If src implements the WriterTo interface,
// the copy is implemented by calling src.WriteTo(dst).
// Otherwise, if dst implements the ReaderFrom interface,
// the copy is implemented by calling dst.ReadFrom(src).
func Copy(dst Writer, src Reader) (written int64, err error) {
	return copyBuffer(dst, src, nil)
}
```
可以看出，将缓存文件的内容拷贝到了fileWriter中。

接着，格式化form表单提交数据的结构：
contentType := bodyWriter.FormDataContentType()
其中函数FormDataContentType定义为：
```golang
// FormDataContentType returns the Content-Type for an HTTP
// multipart/form-data with this Writer's Boundary.
func (w *Writer) FormDataContentType() string {
	return "multipart/form-data; boundary=" + w.boundary
}
```
也就是说：函数FormDataContentType所属类型的对象为Writer指针对象；这个函数不接受参数，返回一个字符串。
在Go中函数也是一种变量，我们通过type定义这种变量的类型。拥有相同参数和相同返回值的函数属于同一种类型。
在Go中通过给函数标明所属类型，来给该类型定义方法，上面的 w *Writer 即表示给*Writer声明了一个方法，这个方法就是FormDataContentType。如果没有 w *Writer ，则纯粹是一个函数，通过包名称访问。
再看看golang中函数的定义的示例：
```golang
func (variable Type) funcName(var1 Type1, var2 Type2, variabicPara ...TypeX) (ret1 ReturnType1, ret2 ReturnType2) {   }
 ^        ^      ^      ^       ^     ^     ^     ^                 ^          ^        ^        ^        ^         ^
 1        2      3      4       5     6     7     8                 9          10       11       12       13        14
```
每一个位置的含义为：
```golang
1 使用 func 关键字来定义函数。
2 如果定义时有 (variable Type) 的内容，variable 表示函数所属类型的对象，如果函数体中不需要用到则可以省略。
3 如果定义时有 (variable Type) 的内容，Type 表示函数所属的类型，Type 本身也可以是一个类型的指针，比如 struct User 的指针 *User。
4 函数的名称。
5 函数可以有 0 个或者多个形参，函数第一个形参的名字。
6 函数第一个形参的类型。
7 函数第二个形参的名字。
8 函数第二个形参的类型。
9 variabicPara ... TypeX 表示可以有可变参数，也就是是零个或者多个同一类型 TypeX 的形参，实参传递的时候放入一个名为 variabicPara 的 slice 对象中。
10 函数可以有多个返回值，第一个返回值的名称。（函数可以指定 命名返回值，也就是说返回值名称在定义函数时确定，并初始化为各自类型的零值。它们在函数内部的使用方式和函数传递的实参是一样的。）
11 第一个返回值的类型。
12 第二个返回值的名称。
13 第二个返回值的类型。
14 函数体，在里面可以定义函数的行为。
```
也就是说FormDataContentType定义属于Writer指针的一个方法，可以对照理解为一个类的成员函数。
将函数作为一种类型的变量，我们可以将这种类型的函数作为值传递。

总体上：客户端通过multipart.Write把文件的文本流写入一个缓存中，然后调用http的Post方法把缓存传到服务器。



## session和数据存储：
Web开发中一个很重要的议题就是如何做好用户的整个浏览过程的控制，因为HTTP协议是无状态的，所以用户的每一次请求都是无状态的，我们不知道在整个Web操作过程中哪些连接与该用户有关，我们应该如何来解决这个问题呢？Web里面经典的解决方案是cookie和session，cookie机制是一种客户端机制，把用户数据保存在客户端，而session机制是一种服务器端的机制，服务器使用一种类似于散列表的结构来保存信息，每一个网站访客都会被分配给一个唯一的标志符,即sessionID,它的存放形式无非两种:要么经过url传递,要么保存在客户端的cookies里.当然,你也可以将Session保存到数据库里,这样会更安全,但效率方面会有所下降。

场景描述：
> 当用户来到微博登陆页面，输入用户名和密码之后点击“登录”后浏览器将认证信息POST给远端的服务器，服务器执行验证逻辑，如果验证通过，则浏览器会跳转到登录用户的微博首页，在登录成功后，服务器如何验证我们对其他受限制页面的访问呢？因为HTTP协议是无状态的，所以很显然服务器不可能知道我们已经在上一次的HTTP请求中通过了验证。

当然，最简单的解决方案就是所有的请求里面都带上用户名和密码，这样虽然可行，但大大加重了服务器的负担（对于每个request都需要到数据库验证），也大大降低了用户体验(每个页面都需要重新输入用户名密码，每个页面都带有登录表单)。既然直接在请求中带上用户名与密码不可行，那么就只有在服务器或客户端保存一些类似的可以代表身份的信息了，所以就有了cookie与session。

cookie，简而言之就是在本地计算机保存一些用户操作的历史信息（当然包括登录信息），并在用户再次访问该站点时浏览器通过HTTP协议将本地cookie内容发送给服务器，从而完成验证，或继续上一步操作。
比较直观的图示：
![cookie原理图](cookie原理图.png)

session，简而言之就是在服务器上保存用户操作的历史信息。服务器使用session id来标识session，session id由服务器负责产生，保证随机性与唯一性，相当于一个随机密钥，避免在握手或传输中暴露用户真实密码。但该方式下，仍然需要将发送请求的客户端与session进行对应，所以可以借助cookie机制来获取客户端的标识（即session id），也可以通过GET方式将id提交给服务器。
比较直观的图示：
![session原理图](session原理图.png)

### cookie分析：

#### 特点：
Cookie是由浏览器维持的，存储在客户端的一小段文本信息，伴随着用户请求和页面在Web服务器和浏览器之间传递。用户每次访问站点时，Web应用程序都可以读取cookie包含的信息。浏览器设置里面有cookie隐私数据选项，打开它，可以看到很多已访问网站的cookies。
cookie是有时间限制的，根据生命期不同分成两种：会话cookie和持久cookie。
> - 1. 如果不设置过期时间，则表示这个cookie生命周期为从创建到浏览器关闭止，只要关闭浏览器窗口，cookie就消失了。这种生命期为浏览会话期的cookie被称为会话cookie。会话cookie一般不保存在硬盘上而是保存在内存里。
> - 2. 如果设置了过期时间(setMaxAge(606024))，浏览器就会把cookie保存到硬盘上，关闭后再次打开浏览器，这些cookie依然有效直到超过设定的过期时间。存储在硬盘上的cookie可以在不同的浏览器进程间共享，比如两个IE窗口。而对于保存在内存的cookie，不同的浏览器有不同的处理方式。

#### golang中的处理：
##### 设置cookie：
Go语言中通过net/http包中的SetCookie来设置：
```golang
// SetCookie adds a Set-Cookie header to the provided ResponseWriter's headers.
// The provided cookie must have a valid Name. Invalid cookies may be
// silently dropped.
func SetCookie(w ResponseWriter, cookie *Cookie) {
	if v := cookie.String(); v != "" {
		w.Header().Add("Set-Cookie", v)
	}
}
```
w表示需要写入的response，cookie是一个struct，定义为：
```golang
// A Cookie represents an HTTP cookie as sent in the Set-Cookie header of an
// HTTP response or the Cookie header of an HTTP request.
//
// See http://tools.ietf.org/html/rfc6265 for details.
type Cookie struct {
	Name  string
	Value string

	Path       string    // optional
	Domain     string    // optional
	Expires    time.Time // optional
	RawExpires string    // for reading cookies only

	// MaxAge=0 means no 'Max-Age' attribute specified.
	// MaxAge<0 means delete cookie now, equivalently 'Max-Age: 0'
	// MaxAge>0 means Max-Age attribute present and given in seconds
	MaxAge   int
	Secure   bool
	HttpOnly bool
	Raw      string
	Unparsed []string // Raw text of unparsed attribute-value pairs
}
```
对应的代码为：
```golang
// cookie处理：设置服务端给定的cookie名称
var cookieName string = "testWriteCookie"

// cookie处理：服务端设置请求的cookie，这儿为了满足路由器的函数要求，添加了request端的指针，但是实际没有使用
func WriteCookie(w http.ResponseWriter, req *http.Request) {
	expiration := time.Now()
	expiration = expiration.AddDate(1, 0, 0)
	cookie := http.Cookie{Name: cookieName, Value: "hellmonky", Expires: expiration}
	http.SetCookie(w, &cookie)
	fmt.Println(cookie.Value)
}
```
之所以函数带有两个参数，但是只用了一个的原因就在于：
这个函数作为HTTP服务的路由器来添加到指定的域名进行测试，所以必须满足路由器函数的定义。

##### 获取cookie：
**golang只能获取满足RFC规范的cookie**，得到这个教训的代码就是上述写入的cookie值为一个单纯的字符串，在response解析cookie的时候被丢弃，所以无法返回。
造成的现象就是：chrome调试中可以看到这个cookie，但是使用golang获取不到，但是可以删除成功。
```golang
// cookie处理：服务端读取用户保存的cookie信息，然后返回给用户页面显示
func ReadCookie(w http.ResponseWriter, req *http.Request) {

	for _, cookie := range req.Cookies() {
		fmt.Fprint(w, cookie.Name)
	}

	cookie, err := req.Cookie(cookieName)
	if err == nil {
		cookievalue := cookie.Value
		w.Write([]byte("<b>cookie的值是：" + cookievalue + "</b>\n"))
	} else {
		w.Write([]byte("<b>读取出现错误：" + err.Error() + "</b>\n"))
	}
}
```
> 参考文档：
> [让人哭笑不得的原因（golang的cookie）](https://www.oschina.net/question/593413_139087)
> [GO语言(golang)无法获取Cookie值](http://blog.sina.com.cn/s/blog_7047e44d01014waf.html)
> [全面解读HTTP Cookie](http://www.webryan.net/2011/08/wiki-of-http-cookie/)
> [cookie规范（RFC 6265）翻译](https://github.com/renaesop/blog/issues/4)

chrome在windows7环境下的cookie默认存储位置为：
```shell
C:\Users\yuanlai.xwt\AppData\Local\Google\Chrome\User Data\Default\Cookies
```
这个文件是SQLite3的数据库文件，可以使用[DBeaver(http://dbeaver.jkiss.org/download/)这个软件打开查看。
然后打开服务器，在chrome中访问生成cookie的服务，然后关闭浏览器，拷贝这个Cookies文件，打开查询：
```sql
select * from cookies where cookies.host_key = 'localhost'
```
可以看到已经有写入的内容了：
```shell
creation_utc	host_key	name	value	path	expires_utc	secure	httponly	last_access_utc	has_expires	persistent	priority	encrypted_value	firstpartyonly
13138707405877370	localhost	Idea-6f89e3fb		/	13454067405877370	0	1	13138861844489448	1	1	1	    ¯óâ   ® óz ¿O½è     çCjÔô@Ãfý  ÛÎI           f  ¿       [r×Ü;  hç[Ñv :2L     ÿ  ß       Æ óvvÍ?e+Q  <(   ëTâÓ3¹¹}CM##Ù3$,Ád:%QSû èÑ!¦´ÓWõ,ÎÙV    :-ìç"Õ]nµeÔ¯µWÎP T	2
13138854069735452	localhost	io		/	0	0	1	13138861844489448	0	0	1	    ¯óâ   ® óz ¿O½è     çCjÔô@Ãfý  ÛÎI           f  ¿       SàÞ]ÅÈ  Y&ý%; ;     ÿ  ß       þ­)¶Wl,   m MJT    ZÖÈ=ÙÂì^ÑÕ¥»Íx û^C ¹S©    ïÐ s,×Ô³5@ùì"Z² ÆÍ	0
13138861844802311	localhost	testWriteCookie		/	13170397844802311	0	0	13138861844802311	1	1	1	    ¯óâ   ® óz ¿O½è     çCjÔô@Ãfý  ÛÎI           f  ¿       \ ÿ 0 <èÖ¡Ö     ÿ  ß       ® ¡ ýôûïj-yXôö    ®ËÁÈ~ ÊÆè jH³HôÏ) CBÇ;    a¾   ¿ b¿  =ZOX Æ	0
```
可以看到有对应的name出现了，但是value内容都为空。可是通过浏览器或者服务端还是可以获取具体的值的，那么这些值存放在哪里？可能需要根据chromium的源代码来确定了。
> 参考文档：
> [Chrome doesn't create cookie for domain=localhost in broken https](http://stackoverflow.com/questions/8134384/chrome-doesnt-create-cookie-for-domain-localhost-in-broken-https)
> [Issue 56211 in chromium: chrome.cookies fails for localhost domains](https://groups.google.com/a/chromium.org/forum/#!topic/chromium-bugs/4HFLDhvvXsc)
> [Cookies on localhost with explicit domain](http://stackoverflow.com/questions/1134290/cookies-on-localhost-with-explicit-domain)
> [CookieMonster](https://www.chromium.org/developers/design-documents/network-stack/cookiemonster)

##### 删除cookie：
golang中删除cookie很简单，就是将指定cookie的值覆盖为空，然后由浏览器自动清楚这个条目。具体代码为：
```golang
// cookie处理：删除用户端cookie信息
func DeleteCookie(w http.ResponseWriter, req *http.Request) {
	cookie := http.Cookie{Name: cookieName, Path: "/", MaxAge: -1}
	fmt.Println("after delete, cookie value is : ", cookie.Value)
	http.SetCookie(w, &cookie)
	w.Write([]byte("<b>删除cookie成功。</b>\n"))
}
```

#### cookie原理解析：
[详解 Cookie 纪要](http://jeffjade.com/2016/10/31/115-summary-of-cookie/)

### session分析：
session，中文经常翻译为会话，其本来的含义是指有始有终的一系列动作/消息，比如打电话是从拿起电话拨号到挂断电话这中间的一系列过程可以称之为一个session。然而当session一词与网络协议相关联时，它又往往隐含了“面向连接”和/或“保持状态”这样两个含义。

session在Web开发环境下的语义又有了新的扩展，它的含义是指一类用来在客户端与服务器端之间保持状态的解决方案。有时候Session也用来指这种解决方案的存储结构。

session机制是一种服务器端的机制，服务器使用一种类似于散列表的结构(也可能就是使用散列表)来保存信息。

但程序需要为某个客户端的请求创建一个session的时候，服务器首先检查这个客户端的请求里是否包含了一个session标识－称为session id，如果已经包含一个session id则说明以前已经为此客户创建过session，服务器就按照session id把这个session检索出来使用(如果检索不到，可能会新建一个，这种情况可能出现在服务端已经删除了该用户对应的session对象，但用户人为地在请求的URL后面附加上一个JSESSION的参数)。如果客户请求不包含session id，则为此客户创建一个session并且同时生成一个与此session相关联的session id，这个session id将在本次响应中返回给客户端保存。

session机制本身并不复杂，然而其实现和配置上的灵活性却使得具体情况复杂多变。这也要求我们不能把仅仅某一次的经验或者某一个浏览器，服务器的经验当作普遍适用的。

#### 特点：
session和cookie的目的相同，都是为了克服http协议无状态的缺陷，但完成的方法不同。session通过cookie，在客户端保存session id，而将用户的其他会话消息保存在服务端的session对象中，与此相对的，cookie需要将所有信息都保存在客户端。因此cookie存在着一定的安全隐患，例如本地cookie中保存的用户名密码被破译，或cookie被其他网站收集（例如：1. appA主动设置域B cookie，让域B cookie获取；2. XSS，在appA上通过javascript获取document.cookie，并传递给自己的appB）。

#### golang中的处理：
目前Go标准包没有为session提供任何支持，只能自己手动根据需要编写相关的代码来实现。

##### session创建流程：
session的基本原理是由服务器为每个会话维护一份信息数据，客户端和服务端依靠一个全局唯一的标识来访问这份数据，以达到交互的目的。当用户访问Web应用时，服务端程序会随需要创建session，这个过程可以概括为三个步骤：
> - 1. 生成全局唯一标识符（sessionid）；
> - 2. 开辟数据存储空间。一般会在内存中创建相应的数据结构，但这种情况下，系统一旦掉电，所有的会话数据就会丢失，如果是电子商务类网站，这将造成严重的后果。所以为了解决这类问题，你可以将会话数据写到文件里或存储在数据库中，当然这样会增加I/O开销，但是它可以实现某种程度的session持久化，也更有利于session的共享；
> - 3. 将session的全局唯一标示符发送给客户端。

以上三个步骤中，最关键的是如何发送这个session的唯一标识这一步上。考虑到HTTP协议的定义，数据无非可以放到请求行、头域或Body里，所以一般来说会有两种常用的方式：cookie和URL重写。
> - 1. Cookie 服务端通过设置Set-cookie头就可以将session的标识符传送到客户端，而客户端此后的每一次请求都会带上这个标识符，另外一般包含session信息的cookie会将失效时间设置为0(会话cookie)，即浏览器进程有效时间。至于浏览器怎么处理这个0，每个浏览器都有自己的方案，但差别都不会太大(一般体现在新建浏览器窗口的时候)；
> - 2. URL重写 所谓URL重写，就是在返回给用户的页面里的所有的URL后面追加session标识符，这样用户在收到响应之后，无论点击响应页面里的哪个链接或提交表单，都会自动带上session标识符，从而就实现了会话的保持。虽然这种做法比较麻烦，但是，如果客户端禁用了cookie的话，此种方案将会是首选。


##### session的管理流程设计：
我们知道session管理涉及到如下几个因素
> - 1. 全局session管理器
> - 2. 保证sessionid 的全局唯一性
> - 3. 为每个客户关联一个session
> - 4. session 的存储(可以存储到内存、文件、数据库等)
> - 5. session 过期处理

###### session管理器和session的接口设计：
首先我们定义管理器自身需要的参数：
```golang
type Manager struct {
	cookieName  string     //private cookiename
	lock        sync.Mutex // protects session
	provider    Provider
	maxlifetime int64
}
```
一个管理器需要的参数为：cookie的名称，当前cookie的提供者，当前session的最长有效时间。

需要注意的是，使用了互斥量作为线程同步的方式。关于多线程加锁保证并发的正确性是一个非常基本的问题，不管是在那种语言中，都会有原理上一致的解决方案，那就是对并发竞争的资源进行加锁处理，C++如此，java如此，在golang中也是如此。
> 参考文档：[Benign data races: what could possibly go wrong?](https://software.intel.com/en-us/blogs/2013/01/06/benign-data-races-what-could-possibly-go-wrong)

那么生成一个管理器的函数就应该是：
```golang
func NewManager(provideName, cookieName string, maxlifetime int64) (*Manager, error) {
	provider, ok := provides[provideName]
	if !ok {
		return nil, fmt.Errorf("session: unknown provide %q (forgotten import?)", provideName)
	}
	return &Manager{provider: provider, cookieName: cookieName, maxlifetime: maxlifetime}, nil
}
```

我们知道session是保存在服务器端的数据，它可以以任何的方式存储，比如存储在内存、数据库或者文件中。因此我们抽象出一个Provider接口，用以表征session管理器底层存储结构。
```golang
type Provider interface {
	SessionInit(sid string) (Session, error)
	SessionRead(sid string) (Session, error)
	SessionDestroy(sid string) error
	SessionGC(maxLifeTime int64)
}
```
> - 1. SessionInit函数实现Session的初始化，操作成功则返回此新的Session变量
> - 2. SessionRead函数返回sid所代表的Session变量，如果不存在，那么将以sid为参数调用SessionInit函数创建并返回一个新的Session变量
> - 3. SessionDestroy函数用来销毁sid对应的Session变量
> - 4. SessionGC根据maxLifeTime来删除过期的数据

我们的session管理器只要实现这些接口函数就可以完成对session的管理了，那么Session需要实现什么样的功能呢？
有过Web开发经验的读者知道，对Session的处理基本就 设置值、读取值、删除值以及获取当前sessionID这四个操作，所以我们的Session需要提供的接口也就实现这四个操作：
```golang
type Session interface {
	Set(key, value interface{}) error //set session value
	Get(key interface{}) interface{}  //get session value
	Delete(key interface{}) error     //delete session value
	SessionID() string                //back current sessionID
}
```

###### session的注册管理：
定义好了session管理器和session，我们就需要将管理器和session本身通过一种机制完成管理关联，这种在驱动实现中非常常见，就是注册管理的方式：
```golang
var provides = make(map[string]Provider)

// Register makes a session provide available by the provided name.
// If Register is called twice with the same name or if driver is nil,
// it panics.
func Register(name string, provider Provider) {
	if provider == nil {
		panic("session: Register provide is nil")
	}
	if _, dup := provides[name]; dup {
		panic("session: Register called twice for provide " + name)
	}
	provides[name] = provider
}
```
将直接实现session的内容，和对应的名称以map的行书填写到全局的provides中，然后每次使用的时候就可以先查询map中是否已经存在，如果存在就直接取出来使用了。

> 关于实现session管理器设计中的一些思考：
> 因为实现一个功能，首先要明确的就是功能的要点，然后按照功能点的划分对整体设计进行一个合理的规划，规划完毕就需要指定明确的实现过程和方式，然后拆分并且按照优先级分派任务，接着并行执行，最后合并联调，完成功能的统一测试，最后交付。这个过程中，在整体设计规划的时候数据结构和算法的作用就会凸显，因为会影响整体的执行效率和开发方式，属于顶层设计。

参考文档：
> - [go-algorithms](https://github.com/0xAX/go-algorithms)
> - [Go 语言 Session 实现](http://blog.guoqiangti.com/?p=318)


###### session的ID生成：
唯一ID的生成非常常见，在Java中官方提供了唯一生成识别码的UUUID包来简化这一过程，在golang中应该怎么完成相同的功能？
我们以session管理器生成唯一ID作为实例：
```golang
func (manager *Manager) sessionId() string {
	b := make([]byte, 32)
	if _, err := io.ReadFull(rand.Reader, b); err != nil {
		return ""
	}
	return base64.URLEncoding.EncodeToString(b)
}
```
方式就是随机生成一个字符串，然后放在32位长的字节数组中，最后使用base64编码作为唯一识别码。
需要注意的是，rand调用了IO包：[rand.Read() 和 io.ReadFull(rand.Reader) 的区别?](http://www.cnblogs.com/ghj1976/p/3435940.html)

###### session的用户绑定生成：
到目前为止，设计了session的管理器和session的接口定义，可以从session管理器生成一个session实例的使用场景来进行描述。
我们需要为每个来访用户分配或获取与他相关连的Session，以便后面根据Session信息来验证操作。SessionStart这个函数就是用来检测是否已经有某个Session与当前来访用户发生了关联，如果没有则创建之。
```golang
func (manager *Manager) SessionStart(w http.ResponseWriter, r *http.Request) (session Session) {
	manager.lock.Lock()
	defer manager.lock.Unlock()
	cookie, err := r.Cookie(manager.cookieName)
	if err != nil || cookie.Value == "" {
		sid := manager.sessionId()
		session, _ = manager.provider.SessionInit(sid)
		cookie := http.Cookie{Name: manager.cookieName, Value: url.QueryEscape(sid), Path: "/", HttpOnly: true, MaxAge: int(manager.maxlifetime)}
		http.SetCookie(w, &cookie)
	} else {
		sid, _ := url.QueryUnescape(cookie.Value)
		session, _ = manager.provider.SessionRead(sid)
	}
	return
}
```
这个Manager类型的方法供用户调用生成对应的session。
> - 首先，使用了管理器的锁保证一致性，并且使用defer在退出时自动解锁。
> - 接着，检查当前用户请求带上来的cookie的名称是否已经在服务器端存在；
> - 然后，如果不存在或者对应名称的cookie的值为空，就进入生成session流程：生成唯一ID，使用sessionInit生成session，将当前的cookiename和生成的session的ID等信息交给response，返回给用户；如果已经存在cookie名称对应的ID，就获取当前用户端cookie名称对应的值，然后将QueryEscape转码的字符串还原（它会把%AB改为字节0xAB，将'+'改为' '等）；
> - 最后从session管理器中，通过获取到的session ID来得到对应的session本身。

配合前面的login函数看看如何响应用户请求的同时检查生成session：
```golang
func login(w http.ResponseWriter, r *http.Request) {
	sess := globalSessions.SessionStart(w, r)
	r.ParseForm()
	if r.Method == "GET" {
		t, _ := template.ParseFiles("login.gtpl")
		w.Header().Set("Content-Type", "text/html")
		t.Execute(w, sess.Get("username"))
	} else {
		sess.Set("username", r.Form["username"])
		http.Redirect(w, r, "/", 302)
	}
}
```
将这个函数和用户访问login页面绑定，通过检查当前用户cookie中对应的ID来查询服务端的session：
首先，从全局函数获取
如果是GET请求，就对模板生成的内容，从session中获取用户名进行模板的执行（Template.Execute）；
如果是其他请求，就将当前用户登录的名称写入session，然后重定位页面到/下。

Redirect告诉request重定向到一个url,这个URL可以是请求路径的的相对路径。定义为：
```golang
func Redirect(w ResponseWriter, r *Request, urlStr string, code int)
```
> - w：服务器响应
> - r：客户端请求
> - urlStr：要重定向的地址
> - code：定义在http里面，一般我们可以使用http.StatusFound

> 需要注意的是：调用http.Redirect()函数后，并不会立刻进行跳转，而是继续顺序执行函数中的所有的代码后，再进行跳转。但是Redirect后面的写界面的代码不会发送到游览器前端的。

###### session的销毁：
Web应用中有用户退出这个操作，那么当用户退出应用的时候，我们需要对该用户的session数据进行销毁操作。也就是通过seesion管理器来销毁sid对应的Session变量。
```golang
//Destroy sessionid
func (manager *Manager) SessionDestroy(w http.ResponseWriter, r *http.Request){
	cookie, err := r.Cookie(manager.cookieName)
	if err != nil || cookie.Value == "" {
		return
	} else {
		manager.lock.Lock()
		defer manager.lock.Unlock()
		manager.provider.SessionDestroy(cookie.Value)
		expiration := time.Now()
		cookie := http.Cookie{Name: manager.cookieName, Path: "/", HttpOnly: true, Expires: expiration, MaxAge: -1}
		http.SetCookie(w, &cookie)
	}
}
```
首先检查用户cookie中是否有对应的键值对，如果没有就不用销毁session，如果存在，使用session管理将内容清空；
然后将cookie失效时间写为现在；
接着，将用户的cookie对应key的value写为空，设置用户cookie，造成失效写入。

###### session的GC：
上面我们看到了怎么实际上在用户登出之后进行session销毁，还有一种情况需要对用户session进行回收，那就是超时。
当用户长时间登入不执行session更新，那么就需要通过session管理器对超时的session进行回收，也就是GC，一般根据maxLifeTime来删除过期的数据。
```golang
func init() {
	go globalSessions.GC()
}

func (manager *Manager) GC() {
	manager.lock.Lock()
	defer manager.lock.Unlock()
	manager.provider.SessionGC(manager.maxlifetime)
	time.AfterFunc(time.Duration(manager.maxlifetime), func() { manager.GC() })
}
```
也就是通过在初始化的时候使用go关键字开启一个线程，然后在这个线程中添加GC操作。并且在GC操作中，使用time包的AfterFunc函数来完成调用。

其中涉及到的知识点有两个：关键字go的使用和time包中AfterFunc函数的使用。
go关键字开头的表达式会启动一个新的goroutine来执行关键字后面的函数。go关键字的使用方式有一下三种：
```golang
//go 关键字放在方法调用前新建一个 goroutine 并让他执行方法体
go GetThingDone(param1, param2);

//上例的变种，新建一个匿名方法并执行
go func(param1, param2) {
}(val1, val2)

//直接新建一个 goroutine 并在 goroutine 中执行代码块
go {
    //do someting...
}
```
需要注意的是，go关键字后面的函数（块）中，所有变量都是原goroutine中的值，只有函数执行是在新的goroutine中。新的goroutine不能和当前go线程用同一个栈，否则会相互覆盖。所以对go关键字的调用协议与普通函数调用是不同的。
既然在新的goruntine中执行了，那么函数的执行结果该如何拿回到原goroutine中？这个时候就需要channel来帮助了。
Channal是什么？Channal就是用来通信的，就像Unix下的管道一样，在Go中是这样使用Channel的。
```golang
package main
 
import "fmt"
 
func main() {
    //创建一个string类型的channel
    channel := make(chan string)
 
    //创建一个goroutine向channel里发一个字符串
    go func() { channel <- "hello" }()
 
    msg := <- channel
    fmt.Println(msg)
}
```
首先，创建一个string类型的channel变量，用于在不同的goroutine中进行值传递。
然后，使用go关键字开启一个新的goroutine，这个goroutine中将一个字符串变量传递给上面新建的channel变量；
最后，在原goroutine中将channel变量中的内容拿出来交给msg，打印显示。

但是上述对channel的使用有一个问题：开启了一个channel，但是没有关闭，会造成内存的泄露。
关于goroutine开启堆栈的方式和演变，可以参考官方博客：
> - [How Stacks are Handled in Go](http://blog.cloudflare.com/how-stacks-are-handled-in-go/)
参考文章：
> - [如何优雅地关闭Go channel](http://www.jianshu.com/p/d24dfbb33781)
> - [Golang 语言基础之十： goroutine, channel](http://xhrwang.me/2014/12/31/golang-fundamentals-10-goroutine-channel.html)

time包中的AfterFunc函数定义为：
```golang
// AfterFunc waits for the duration to elapse and then calls f
// in its own goroutine. It returns a Timer that can
// be used to cancel the call using its Stop method.
func AfterFunc(d Duration, f func()) *Timer {
	t := &Timer{
		r: runtimeTimer{
			when: when(d),
			f:    goFunc,
			arg:  f,
		},
	}
	startTimer(&t.r)
	return t
}
```
可以看到，通过调用这个函数，会生成一个timer channel，然后在迟设置的时间之后，在func f在新的goroutine中执行。

参考文档：
> - [go语言的time包](https://my.oschina.net/u/943306/blog/149395)
> - [How can I use 'time.After' and 'default' in Golang?](https://stackoverflow.com/questions/39212333/how-can-i-use-time-after-and-default-in-golang)

##### session的内存存储实现：
在理解了上述基本流程之后，需要进行实现来完后验证。毕竟实践是检验真理的唯一标准。这一节的主要内容就是从底层实现一个基于内存的session管理代码。
通过init函数注册到session管理器中。这样就可以方便的调用了。具体的方式为：
当import的时候已经执行了memory函数里面的init函数，来完成session注册到session管理器中，用户就可以使用了。




注意：
1 在引入一个功能比较独立的部分，需要将这部分作为一个独立的package进行组织。其他的package引用这个包的时候，需要根据当前GOPATH作为根，然后查询到这个引用包的位置进行描述。
2 并且golang的可视范围就限制于package这个级别，分开文件只是为了方便编码，而不是逻辑概念上的区分，所以在一个package下，同名的interface和struct是会报redeclared错误的。
参考文档：
[Go: “instance” redeclared in this block](https://stackoverflow.com/questions/34344172/go-instance-redeclared-in-this-block)

关于golang中的interface关键字和面向对象编程：
golang使用了非侵入式接口实现（和java完全不一样），一个很重要的好处就是去掉了繁杂的继承体系。golang中接口的定义和类对接口的实现分离，减少了耦合，而且可以在类的实现过程中动态的决定需要实现那些接口规范，而不是全部都需要实现，增强了灵活性。但是在带来便利性的同时也会造成接口实现关系不明确的问题，太过于灵活不好把控。
golang不支持完整的面向对象思想，它没有继承，多态则完全依赖接口实现。
其中关键的一点就是：golang是通过 interface 类型接口实现的继承多态的效果。
golang只能模拟继承，其本质是组合，只不过golang语言为我们提供了一些语法糖使其看起来达到了继承的效果。面向对象中一个很重要的基本原则--里氏代换原则(Liskov Substitution Principle LSP)在这里就行不通了，习惯面向对象语言的同学可能会有些不适应，当你将一个父类的指针指向子类的对象时，golang会毫不吝啬的抛出一个编译错误。






#### session原理解析：

### 参考文档：
[]()
[]()
[]()
[]()
[]()












































RESTFul API测试框架：
http://www.hascode.com/2011/10/testing-restful-web-services-made-easy-using-the-rest-assured-framework/
https://github.com/rest-assured/rest-assured/wiki/Usage
https://testerhome.com/topics/7060
http://www.cnblogs.com/wade-xu/p/4298819.html


http://www.cnblogs.com/jinsdu/p/4606113.html
http://wetest.qq.com/lab/view/145.html


https://www.aliway.com/read.php?spm=a1z2e.8101737.list.6.nTQqbR&tid=350194

//
http://cizixs.com/2016/08/17/golang-http-server-side
https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/04.4.md
http://blog.haohtml.com/archives/16839
http://www.cnblogs.com/golove/p/3276678.html

gradle:
https://spring.io/guides/gs/gradle/#scratch
http://www.importnew.com/15881.html

JSON format:
https://jsonformatter.curiousconcept.com/

参考教程：
系列文章：
[WEB请求处理一：浏览器请求发起处理](http://www.jianshu.com/p/558455228c43)
[WEB请求处理二：Nginx请求反向代理](http://www.jianshu.com/p/bed000e1830b)
[WEB请求处理三：Servlet容器请求处理](http://www.jianshu.com/p/571c474279af)
[WEB请求处理四：Tomcat配置实践](http://www.jianshu.com/p/b5a3c269cae3)
[WEB请求处理五：MVC框架请求处理](http://www.jianshu.com/p/6462e69ce241)
[WEB请求处理六：浏览器HTTP协议漫谈](http://www.jianshu.com/p/b1ea16450fff)
这个系列文章完整的描述了整个HTTP请求的处理流程，对于理解原理有非常重要的帮助。只有明白了网络通信和服务的原理才能正确的处理流程。