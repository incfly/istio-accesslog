# Istio Access Log

This repo contains a sample to illustrate how the sidecar access log can be used to termine the time spent in Envoy.

TL;DR: `%DURATION% - %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%` is the time spent in the Envoy.


# Goal Analyze Istio Envoy time spent there.

## Setup

Install Istio and enable sidecar auto-injection to the default namespace.

```sh
istioctl install --set profile=demo -y
kubectl label namespace default istio-injection=enabled --overwrite
```

Deploy httpbin app and expose it on the ingress gateway.

```sh
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/httpbin/httpbin.yaml
# sample/ folder contains the necessary gateway and virtula service to expose httpbin on ingress gateway.
kubectl apply -f ./sample/
```

## Start by Basic Requests

Send a few requests to the httpbin.

```sh
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
curl  -I -HHost:httpbin.example.com  "http://$INGRESS_HOST:$INGRESS_PORT/status/200"
```

Inspect the ingress gateway and httpbin sidecar access log.

```
ingress gateway access log

[2022-05-03T20:24:37.057Z] "HEAD /status/200 HTTP/1.1" 200 - via_upstream - "-" 0 0 3 2 "10.138.0.42" "curl/7.68.0" "e3c5d335-85e1-9174-9e13-3a5aec0d69c5" "httpbin.example.com" "10.100.0.7:80" outbound|8000||httpbin.default.svc.cluster.local 10.100.0.8:50810 10.100.0.8:8080 10.138.0.42:19665 - -

httpbin sidecar access log

1 1 -- duration, upstream service time.
[2022-05-03T20:24:37.058Z] "HEAD /status/200 HTTP/1.1" 200 - via_upstream - "-" 0 0 1 1 "10.138.0.42" "curl/7.68.0" "e3c5d335-85e1-9174-9e13-3a5aec0d69c5" "httpbin.example.com" "10.100.0.7:80" inbound|80|| 127.0.0.6:60953 10.100.0.7:80 10.138.0.42:0 outbound_.8000_._.httpbin.default.svc.cluster.local default
```

**Undestand what happened**

- The request flow: `curl -> ingress gateway -> httpbin sidear -> httpbin`. You can see the request id matches.
- According to [Istio access log](https://istio.io/latest/docs/tasks/observability/logs/access-log/#default-access-log-format),
`%DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%` is printed out in the access log. Specifically
    - In ingress gateway part, `3 2`, the total request duration seen by ingress gateway is 3ms, the time from ingress gateway to route to httpbin service, till the httpbin finishes response is 2ms.
    - In another words, time spent in Ingress gateawy is approximately 3 - 2 = 1ms. Time in httpbin sidecar is under 1 ms (1 - 1 = 0ms).
    - For more detailed reference, see [`DURATION`](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#command-operators)
    and [`X-ENVOY-UPSTREAM-SERVICE-TIME`](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter#x-envoy-upstream-service-time) in the Envoy AccessLog documentation.

## Inject Delays in Ingress Gateway

Istio allows you to inject artificial delays in the envoy to simulate failures. We will rely on this feature to understand.


```sh
kubectl apply -f ./sample/vs-inject-delay.yaml
# Wait a few seconds and then send requests again.
curl  -I -HHost:httpbin.example.com   "http://$INGRESS_HOST:$INGRESS_PORT/status/200"
```

This time the `curl` has noticeable delays for the response. We will confirm this from the access log.


```
Ingress gateway access log.

[2022-05-03T20:32:51.440Z] "HEAD /status/200 HTTP/1.1" 200 DI via_upstream - "-" 0 0 502 2 "10.138.0.43" "curl/7.68.0" "24012ce2-bb7a-9cb5-a683-1af1e91a9c37" "httpbin.example.com" "10.100.0.7:80" outbound|8000||httpbin.default.svc.cluster.local 10.100.0.8:53388 10.100.0.8:8080 10.138.0.43:37455 - -

Httpbin sidecar log.

[2022-05-03T20:32:51.941Z] "HEAD /status/200 HTTP/1.1" 200 - via_upstream - "-" 0 0 1 1 "10.138.0.43" "curl/7.68.0" "24012ce2-bb7a-9cb5-a683-1af1e91a9c37" "httpbin.example.com" "10.100.0.7:80" inbound|80|| 127.0.0.6:59811 10.100.0.7:80 10.138.0.43:0 outbound_.8000_._.httpbin.default.svc.cluster.local default
```

**Undestand what happened**

We inject the delays in the ingress gateway by 500ms. This matches the ingress gateway, 502 - 2 = 500ms.
The httpbin sidecar should not be affected. The acess log `DURATION` and `UPSTREAM-SERVICE-TIME` stays the same.

Now let's remove the injected delays in the ingress gateway.

```sh
# This overrides the previous virtual service to remove the injected delay.
kubectl apply -f ./samples/vs-httpbin.yaml
```

## Inject delay in application

Httpbin app itself has a predefined `/delay/<seconds-to-delay>` path. This is perfect for our purpose.

```sh
curl  -I -HHost:httpbin.example.com   "http://$INGRESS_HOST:$INGRESS_PORT/delay/1"
```

```
ingress gateway
[2022-05-03T20:41:48.643Z] "HEAD /delay/1 HTTP/1.1" 200 - via_upstream - "-" 0 0 1004 1003 "10.138.0.42" "curl/7.68.0" "19ff72c6-39d4-960e-a44c-60d801517d8d" "httpbin.example.com" "10.100.0.7:80" outbound|8000||httpbin.default.svc.cluster.local 10.100.0.8:56198 10.100.0.8:8080 10.138.0.42:30953 - -

sidecar
[2022-05-03T20:41:48.643Z] "HEAD /delay/1 HTTP/1.1" 200 - via_upstream - "-" 0 0 1004 1003 "10.138.0.42" "curl/7.68.0" "19ff72c6-39d4-960e-a44c-60d801517d8d" "httpbin.example.com" "10.100.0.7:80" outbound|8000||httpbin.default.svc.cluster.local 10.100.0.8:56198 10.100.0.8:8080 10.138.0.42:30953 - -
```

**Undestand what happened**

This time you can see the request duration increases in both ingress gateway and sidecar. This is expected because the actual incresed latency are in the application hop.

The actual latency is introduced in the application, so the sidecar's `UPSTREAM-SERVICE-TIME` contributes the actual part.

## Misc

TODO: figure out how to exclude the latency for the downstream connection part. more drill down.
TODO: now matter of making it more self sustained in this repo.
