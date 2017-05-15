---
layout: post
title: "Writing a Custom Kubectl Exec Command"
description: "How to integrate a Websocket connection into the Kubernetes Master security layer"
summary: "How to integrate a Websocket connection into the Kubernetes Master security layer"
category: "Kubernetes"
image: "img/posts/kubevirt.png"
tags: [ "Kubernets", "client-go", "Websocket", "kubectl", "exec"]
---
{% include JB/setup %}

In this post I want to create my own `kubectl exec` command in GO. Kubernetes
provides a nice SDK called
[client-go](https://github.com/kubernetes/client-go). It allows you to write
your own controllers, watch for changes, do REST calls against the API-Server
and much more. However, until
[client-go/#45](https://github.com/kubernetes/client-go/issues/45) is resolved,
it is not possible to use the Websocket endpoints of the API-Server out of the
box. This is especially annoying if you want to write a Websocket client, which
can connect to arbitrary secured API-Servers. The
[Authenticating](https://kubernetes.io/docs/admin/authentication/) page shows
what the API-Server currently supports, and writing a client on your own, which
supports all these authentication mechanisms, does definitely not scale.


Luckily client-go gives you all the low-level tools to reuse all negotiation
code for your Websocket connection. Client-go allows, to create your own
[http.RoundTripper](https://go.org/pkg/net/http/#RoundTripper), which you can
then wrap with `rest.HTTPWrappersForConfig(config, myRoundTripper)`. The
wrapping RoundTrippers will make sure to do preflight requests, and add all
necessary security headers to your request. Further, via
`rest.TLSConfigFor(config)`, a `tls.Config` can be requested and set on our
custom request. It is also worth noting, that with RoundTrippers it is not
possible to get the underlying connection after the invokation. To solve this,
we can do our work on the Websocket via a callback inside the RoundTripper.
Now let's start with wrapping a `gorilla/websocket` connection with our custom
RoundTripper:

{% highlight go %}
type RoundTripCallback func(conn *websocket.Conn, resp *http.Response, err error) error

type WebsocketRoundTripper struct {
	Dialer *websocket.Dialer
	Do     RoundTripCallback
}

func (d *WebsocketRoundTripper) RoundTrip(r *http.Request) (*http.Response, error) {
	conn, resp, err := d.Dialer.Dial(r.URL.String(), r.Header)
	if err == nil {
		defer conn.Close()
	}
	return resp, d.Do(conn, resp, err)
}
{% endhighlight %}

All this does, is taking the connection details and the header from the
provided request, tries to establish a Websocket connection, and forwards the
result, to a callback `RoundTripCallback`. At this stage, we can
expect that the API-Server security negotiations are already done, and that we
just care about the additional security headers. 

Next, a callback, which prints out the response of our command, executed in the
pod, is needed:

{% highlight go %}
func WebsocketCallback(ws *websocket.Conn, resp *http.Response, err error) error {

	if err != nil {
		if resp != nil && resp.StatusCode != http.StatusOK {
			buf := new(bytes.Buffer)
			buf.ReadFrom(resp.Body)
			return fmt.Errorf("Can't connect to console (%d): %s\n", resp.StatusCode, buf.String())
		}
		return fmt.Errorf("Can't connect to console: %s\n", err.Error())
	}

	txt := ""
	for {
		_, body, err := ws.ReadMessage()
		if err != nil {
			fmt.Println(txt)
			if err == io.EOF {
				return nil
			}
			return err
		}
		txt = txt + string(body)
	}
}
{% endhighlight %}

It reads the response from the API-Server and prints the result to stdout, once
the connection is closed. Now that the receiving part is done, we can think about
constructing the actual `exec` request:

{% highlight go %}
func requestFromConfig(config *rest.Config, pod string, container string, namespace string, cmd string) (*http.Request, error) {

	u, err := url.Parse(config.Host)
	if err != nil {
		return nil, err
	}

	// gorilla/websocket expecst wss:// or ws:// urls
	switch u.Scheme {
	case "https":
		u.Scheme = "wss"
	case "http":
		u.Scheme = "ws"
	default:
		return nil, fmt.Errorf("Malformed URL %s", u.String())
	}

	// Construct an exec call
	u.Path = fmt.Sprintf("/api/v1/namespaces/%s/pods/%s/exec", namespace, pod)
	if container != "" {
		u.RawQuery = "command=" + cmd +
			"&container=" + container +
			"&stderr=true&stdout=true"
	}
	req := &http.Request{
		Method: http.MethodGet,
		URL:    u,
	}

	return req, nil
}
{% endhighlight %}


This will create a `GET` request, with all the necessary query parameters
attached, to execute an arbitrary command inside a Pod.

Finally the wrapped RoundTripper can be constructed:


{% highlight go %}
func roundTripperFromConfig(config *rest.Config) (http.RoundTripper, error) {

	// Configure TLS
	tlsConfig, err := rest.TLSConfigFor(config)
	if err != nil {
		return nil, err
	}

	// Configure the websocket dialer
	dialer := &websocket.Dialer{
		Proxy:           http.ProxyFromEnvironment,
		TLSClientConfig: tlsConfig,
	}

	// Create a roundtripper which will pass in the final underlying websocket connection to a callback
	rt := &WebsocketRoundTripper{
		Do:     WebsocketCallback,
		Dialer: dialer,
	}

	// Make sure we inherit all relevant security headers
	return rest.HTTPWrappersForConfig(config, rt)
}
{% endhighlight %}

In this function, all the wiring based on a provided `rest.Config` happens.
First the TLS configuration is constructed and passed to the Websocket dialer.
Then the Websocket RoundTripper is constructed and wrapped by the Kubernetes
RoundTrippers. Now only the main method is missing:

{% highlight go %}
func main() {

	flag.StringVar(&kubeconfig, "kubeconfig", "", "absolute path to the kubeconfig file")
	flag.StringVar(&master, "master", "", "master url")
	flag.StringVar(&pod, "pod", "", "pod")
	flag.StringVar(&namespace, "namespace", "", "namespace")
	flag.StringVar(&container, "container", "", "container")
	flag.StringVar(&command, "command", "", "command")
	flag.Parse()

	// creates the connection
	config, err := clientcmd.BuildConfigFromFlags(master, kubeconfig)
	if err != nil {
		glog.Fatal(err)
	}

	// Create a round tripper with all necessary kubernetes security details
	wrappedRoundTripper, err := roundTripperFromConfig(config)
	if err != nil {
		log.Fatalln(err)
	}

	// Create a request out of config and the query parameters
	req, err := requestFromConfig(config, pod, container, namespace, command)
	if err != nil {
		log.Fatalln(err)
	}

	// Send the request and let the callback do its work
	_, err = wrappedRoundTripper.RoundTrip(req)

	if err != nil {
		log.Fatalln(err)
	}
}
{% endhighlight %}

If I now run `ls` in the `virt-api` Pod on my KubeVirt development environment,

{% highlight go %}
./kubernetes-custom-exec -kubeconfig ~/go/src/kubevirt.io/kubevirt/cluster/vagrant/.kubeconfig \
    -pod virt-api-3813486938-t0mfg -namespace default -command ls -container virt-api
{% endhighlight %}
I see the resulting directories enumerated.

The full example is available at
[rmohr/kubernetes-custom-exec](https://github.com/rmohr/kubernetes-custom-exec).
