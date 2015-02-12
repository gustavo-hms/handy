Handy
==========================================

Handy is a fast and simple HTTP multiplexer for Golang. It fills some gaps
related to the default Golang's HTTP multiplexer:

	* URI variable support (eg: "/test/{foo}")
	* Codecs
	* Interceptors

Handy uses the Handler As The State Of the Request. This approach allows simple and advanced usages.

## Creating a Handler
You just need to embed handy.DefaultHandler in your structure and override the HTTP method:

~~~ go
package main

import (
	"github.com/trajber/handy"
	"log"
	"net/http"
)

func main() {
	srv := handy.NewHandy()
	srv.Handle("/hello/", func() handy.Handler { return new(MyHandler) })
	log.Fatal(http.ListenAndServe(":8080", srv))
}

type MyHandler struct {
	handy.DefaultHandler
}

func (h *MyHandler) Get(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello World"))
}
~~~

# Path with variables
Path variables must be enclosed by braces.

~~~ go
srv.Handle("/hello/{name}", func() handy.Handler { 
	return new(MyHandler) 
})
~~~

And you can read them using the Handler's fields. You just need to tag the field.

~~~ go
type MyHandler struct {
	handy.DefaultHandler
	Name string `param:"name"`
}

func (h *MyHandler) Get(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello " + h.Name))
}
~~~

### URI variables - a complete example:
~~~ go
package main

import (
	"github.com/trajber/handy"
	"log"
	"net/http"
)

func main() {
	srv := handy.NewHandy()
	srv.Handle("/hello/{name}", func() handy.Handler {
		return new(MyHandler)
	})
	log.Fatal(http.ListenAndServe(":8080", srv))
}

type MyHandler struct {
	handy.DefaultHandler
	Name string `param:"name"`
}

func (h *MyHandler) Get(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello " + h.Name))
}
~~~

# Interceptors
To execute functions before and/or after the verb method being called you can use interceptors. To do so you need to create a InterceptorChain in you Handler to be executed Before or After the HTTP verb method.

## Interceptors - a complete example
~~~ go
package main

import (
	"github.com/trajber/handy"
	"log"
	"net/http"
)

func main() {
	srv := handy.NewHandy()
	srv.Handle("/hello/", func() handy.Handler {
		return new(MyHandler)
	})
	log.Fatal(http.ListenAndServe(":8080", srv))
}

type MyHandler struct {
	handy.DefaultHandler
}

func (h *MyHandler) Get(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Success"))
}

func (h *MyHandler) Interceptors() handy.InterceptorChain {
	return handy.NewInterceptorChain().Chain(new(TimerInterceptor))
}

type TimerInterceptor struct {
	Timer time.Time
	handy.NoErrorInterceptor
}

func (i *TimerInterceptor) Before(w http.ResponseWriter, r *http.Request) {
	i.Timer = time.Now()
	return nil
}

func (i *TimerInterceptor) After(w http.ResponseWriter, r *http.Request) {
	log.Println("Took", time.Since(i.Timer))
}
~~~

## JSON Codec interceptor
Handy comes with a JSONCodec interceptor out of the box. It can be used to automatically unmarshal requests and marshal responses using JSON. It does so by reading special tags in your handler:

~~~ go
type MyResponse struct {
	Message string `json:"message"`
}

type MyHandler struct {
	handy.DefaultHandler
	// this structure will be used only for get and put methods
	Response MyResponse `response:"get,put"` 
}
~~~

Now, you just need to include JSONCode in the handler's interceptor chain:
~~~ go
func (h *MyHandler) Interceptors() handy.InterceptorChain {
	codec := inteceptor.NewJSONCodec(h)
	return handy.NewInterceptorChain().Chain(codec)
}
~~~

### JSON Codec inteceptor - a complete example:
~~~ go
package main

import (
		"github.com/trajber/handy"
		"github.com/trajber/handy/interceptor"
    "log"
    "net/http"
)

func main() {
    srv := handy.NewHandy()
    srv.Handle("/hello/", func() handy.Handler {
        return new(MyHandler)
    })
    log.Fatal(http.ListenAndServe(":8080", srv))
}

type MyHandler struct {
    handy.DefaultHandler
    Response MyResponse `response:"all"`
}

func (h *MyHandler) Get(w http.ResponseWriter, r *http.Request) {
    h.Response.Message = "hello world"
}

func (h *MyHandler) Interceptors() handy.InterceptorChain {
	codec := interceptor.NewJSONCodec(h)
	return handy.NewInterceptorChain().Chain(codec)
}

type MyResponse struct {
    Message string `json:"message"`
}
~~~

### JSON Codec inteceptor - An example with JSON in request and response:
~~~go
package main

import (
	"log"
	"net/http"

	"github.com/trajber/handy"
	"github.com/trajber/handy/interceptor"
)

func main() {
	srv := handy.NewHandy()
	srv.Handle("/hello/", func() handy.Handler {
		return new(MyHandler)
	})
	log.Fatal(http.ListenAndServe(":8181", srv))
}

type MyHandler struct {
	handy.DefaultHandler
	Response MyResponse `response:"post"`
	Request  MyRequest  `request:"post"`
}

func (h *MyHandler) Post(w http.ResponseWriter, r *http.Request) {
	h.Response.Answer = "The answer for " + h.Request.Question
}

func (h *MyHandler) Interceptors() handy.InterceptorChain {
	codec := interceptor.NewJSONCodec(h)
	return handy.NewInterceptorChain().Chain(codec)
}

type MyResponse struct {
	Answer string `json:"answer"`
}

type MyRequest struct {
	Question string `json:"question"`
}
~~~

#Logging
Bad things happens even inside Handy; You can set your own function to handle Handy errors.

~~~go
package main

import (
		"github.com/trajber/handy"
    "log"
    "net/http"
)

func main() {
    srv := handy.NewHandy()
    // This function will be called when 
    // some error occurs inside Handy code.
    srv.ErrorFunc = func(e error) {
    	// here you can handle the error
    	log.Println(e)
  	}
    srv.Handle("/hello/", func() handy.Handler {
        return new(MyHandler)
    })

    log.Fatal(http.ListenAndServe(":8080", srv))
}
~~~

# Tests
You can use [Go's httptest package] (http://golang.org/pkg/net/http/httptest/)

~~~ go
package handler

import (
	"github.com/trajber/handy"
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestHandler(t *testing.T) {
	mux := handy.NewHandy()
	h := new(HelloHandler)
	mux.Handle("/{name}/{id}", func() handy.Handler {
		return h
	})

	req, err := http.NewRequest("GET", "/foo/10", nil)
	if err != nil {
		t.Fatal(err)
	}

	w := httptest.NewRecorder()
	mux.ServeHTTP(w, req)

	if h.Id != 10 {
		t.Errorf("Unexpected Id value %d", h.Id)
	}

	if h.Name != "foo" {
		t.Errorf("Unexpected Name value %s", h.Name)
	}

	t.Logf("%d - %s - %d", w.Code, w.Body.String(), h.Id)
}
~~~

