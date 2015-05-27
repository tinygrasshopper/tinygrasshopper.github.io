---
layout:     post
title:      Testing HTTP client wrappers in Golang
date:       2015-05-05 12:31:19
summary:
categories: golang testing
---

There are a lot of times I have to create a service wrapper for better separation,
here I would like to go over the structure of code.

For this example I will around the very simplest non authenticated public API I could find [ipify](http://www.ipify.org/).

The way I normally go about modeling this is to expose the endpoint the code will hit, as a package variable, alternativly I the other way which comes to mind is to inject this as a parameter to the construction of the client

_ipify.go_
{% highlight go %}
package ipify

import (
	"io/ioutil"
	"net/http"
)

var EndPoint string = "https://api.ipify.org/"

func FindIP() (string, error) {
	resp, err := http.Get(EndPoint)
	if err != nil {
		return "", err
	}
	ip, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		return "", err
	}

	return string(ip), nil
}
{% endhighlight %}

The [httptest](http://golang.org/pkg/net/http/httptest/) package provides a standard way to start a test HTTP server, which you can pass handlers to. To assert on the requests received and to inject responses, we capture the request objects in the handlers, and write known responses back. The solution is the most trivial and duct tapy thing that could be thought of to capture requests and mock responses, at least it has no dependency requirement outside of standard lib.

_ipify\_test.go_
{% highlight go %}

package ipify_test

import (
	"io/ioutil"
	"net/http"
	"net/http/httptest"

	"github.com/tinygrasshopper/ipify"

	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
)

var _ = Describe("Ipify", func() {
	It("fetches the ip", func() {
		mockedServer := NewMockedServer("192.168.0.1")
		ipify.EndPoint = mockedServer.URL

		Expect(ipify.FindIP()).To(Equal("192.168.0.1"))
	})
})

type MockedServer struct {
	*httptest.Server
	Requests [][]byte
	Response string
}

func NewMockedServer(response string) *MockedServer {
	ser := &MockedServer{}
	ser.Server = httptest.NewServer(ser)
	ser.Requests = [][]byte{}
	ser.Response = response
	return ser
}
func (ser *MockedServer) ServeHTTP(resp http.ResponseWriter, req *http.Request) {
	var err error
	lastRequest, err := ioutil.ReadAll(req.Body)
	if err != nil {
		panic(err)
	}
	ser.Requests = append(ser.Requests, lastRequest)
	resp.Write([]byte(ser.Response))
}

{% endhighlight %}

The solution wont really work for a lot of scenarios, and is pretty limited. Thus began the search for a better way to mock server behavior, which ended pretty quickly as [gomega](http://onsi.github.io/gomega/) the test library I use has a really good http client testing library [ghttp](http://onsi.github.io/gomega/#ghttp-testing-http-clients), which cleans up stuff quite a bit. 

_ipify\_test.go_
{% highlight go %}
package ipify_test

import (
	"net/http"

	"github.com/tinygrasshopper/ipify"

	. "github.com/onsi/ginkgo"
	. "github.com/onsi/gomega"
	"github.com/onsi/gomega/ghttp"
)

var _ = Describe("Ipify", func() {
	It("fetches the ip", func() {
		mockedServer := ghttp.NewServer()
		mockedServer.AppendHandlers(ghttp.RespondWith(http.StatusOK, "192.168.0.1"))

		ipify.EndPoint = mockedServer.URL()

		Expect(ipify.FindIP()).To(Equal("192.168.0.1"))
	})
})
{% endhighlight %}
