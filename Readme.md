<p align="center">
    <img src="logo/kubewebhook_logo@0,5x.png" width="30%" align="center" alt="kubewebhook">
</p>

# kubewebhook [![Build Status][travis-image]][travis-url] [![Go Report Card][goreport-image]][goreport-url] [![GoDoc][godoc-image]][godoc-url]

Kubewebhook is a small Go framework to create [external admission webhooks][aw-url] for Kubernetes.

With Kubewebhook you can make validating and mutating webhooks very fast and focusing mainly on the domain logic of the webhook itself.

## Features

- Ready for mutating and validating webhook kinds (compatible with CRDs).
- Easy and testable API.
- Simple, extensible and flexible.
- Multiple webhooks on the same server.
- Webhook metrics ([RED][red-metrics-url]) for [Prometheus][prometheus-url] with [Grafana dashboard][grafana-dashboard] included.
- Webhook tracing with [Opentracing][opentracing-url].
- Type specific (static) webhooks and multitype (dynamic) webhooks.

## Status

Kubewebhook has been used in production for several months, and the results have been good.

## Example

Here is a simple example of mutating webhook that will add `mutated=true` and `mutator=pod-annotate` annotations.

```go
func main() {
    logger := &log.Std{Debug: true}
    cfg := initFlags()

    // Mutator.
    mt := mutatingwh.MutatorFunc(func(_ context.Context, obj metav1.Object) (bool, error) {
        pod, ok := obj.(*corev1.Pod)
        if !ok {
            return false, nil
        }

        if pod.Annotations == nil {
            pod.Annotations = make(map[string]string)
        }
        pod.Annotations["mutated"] = "true"
        pod.Annotations["mutator"] = "pod-annotate"

        return false, nil
    })

    mcfg := mutatingwh.WebhookConfig{
        Name: "podAnnotate",
        Obj:  &corev1.Pod{},
    }
    wh, err := mutatingwh.NewWebhook(mcfg, mt, nil, nil, logger)
    if err != nil {
        fmt.Fprintf(os.Stderr, "error creating webhook: %s", err)
        os.Exit(1)
    }

    // Get the handler for our webhook.
    whHandler := whhttp.HandlerFor(wh)
    logger.Infof("Listening on :8080")
    err = http.ListenAndServeTLS(":8080", cfg.certFile, cfg.keyFile, whHandler)
    if err != nil {
        fmt.Fprintf(os.Stderr, "error serving webhook: %s", err)
        os.Exit(1)
    }
}
```

You can get more examples in [here](examples)

## Production ready example

This repository is a production ready webhook app: https://github.com/slok/k8s-webhook-example

It shows, different webhook use cases, app structure, testing domain logic, kubewebhook use case, how to deploy...

## Static and dynamic webhooks

We have 2 kinds of webhooks:

- Static: Common one, is a single resource type webhook.
  - Use [`mutating.WebhookConfig.Obj`][mutating-cfg] to configure.
  - Use [`validating.WebhookConfig.Obj`][validating-cfg] to configure.
- Dynamic: Used when the same webhook act on multiple types, unknown types and/or is used for generic stuff (e.g labels).
  - To use this kind of webhook, don't set the type on the configuration or set to `nil`.
  - If a request for an unknown type is not known by the webhook libraries, it will fallback to [`runtime.Unstructured`][runtime-unstructured] object type.
  - Very useful to manipulate multiple resources on the same webhook (e.g `Deployments`, `Statfulsets`).
  - CRDs are unknown types so they will fallback to [`runtime.Unstructured`][runtime-unstructured]`.
  - If using CRDs, better use `Static` webhooks.
  - Very useful to maniputale any `metadata` based validation or mutations (e.g `Labels, annotations...`)

## Compatibility matrix

Depending on your Kubernetes cluster version, you should select the Kubewebhook version.

This Matrix ensures that Kubernetes libs are used with the described versions and the
integration tests have been tested against this Kubernetes cluster versions, this doesn't mean
that other Kubewebhook versions different to the matched ones to Kubernetes versions don't
work (e.g k8s v1.15 with Kubewebhook v0.3). You can try it and check if they work for you.

| k8s version | Kubewebhook version | Supported admission reviews | Support dynamic webhooks |
| ------------| ------------------- | --------------------------- | ------------------------ |
| 1.18        | v0.10               | v1beta1                     | ✔                        |
| 1.18        | v0.9                | v1beta1                     | ✖                        |
| 1.17        | v0.8                | v1beta1                     | ✖                        |
| 1.16        | v0.7                | v1beta1                     | ✖                        |
| 1.15        | v0.6                | v1beta1                     | ✖                        |
| 1.14        | v0.5                | v1beta1                     | ✖                        |
| 1.13        | v0.4                | v1beta1                     | ✖                        |
| 1.12        | v0.3                | v1beta1                     | ✖                        |
| 1.11        | v0.2                | v1beta1                     | ✖                        |
| 1.10        | v0.2                | v1beta1                     | ✖                        |

## Documentation

- [Documentation][docs]
- [API][godoc-url]

## Integration tests

Tools required

- [mkcert] (optional if you want to create new certificates).
- [kind] (option1, to run the cluster).
- [k3s] (option2, to run the cluster)
- ssh (to expose our webhook to the internet).

### (Optional) Certificates

Certificates are ready to be used on [/test/integration/certs]. This certificates are valid for [ngrok] tcp tunnels so, they should be valid for our exposed webhooks using [ngrok].

If you want to create new certificates execute this:

```bash
make create-integration-test-certs
```

### Running the tests

The integration tests are on [/tests/integration], there are the certificates valid for [ngrok] where the tunnel will be exposing the webhooks.

Go integration tests require this env vars:

- `TEST_WEBHOOK_URL`: The url where the apiserver should make the webhook requests.
- `TEST_LISTEN_PORT`: The port where our webhook will be listening the requests.

There are 2 ways of bootstrapping the integration tests, one using kind and another using [k3s].

To run the integration tests do:

```bash
make integration-test
```

This it will bootstrap a cluster with [kind] by default and a [k3s] cluster if `K3S=true` env var is set. A ssh tunnel in a random address, and finally use the precreated certificates (see previous step), after this will execute the tests, and do it's best effort to tear down the clusters (on k3s could be problems, so have a check on k3s processes).

### Developing integration tests

To develop integration test is handy to run a k3s cluster and a serveo tunnel, then check out [/tests/integration/helper/config] and use this development settings on the integration tests.

[travis-image]: https://travis-ci.org/slok/kubewebhook.svg?branch=master
[travis-url]: https://travis-ci.org/slok/kubewebhook
[goreport-image]: https://goreportcard.com/badge/github.com/slok/kubewebhook
[goreport-url]: https://goreportcard.com/report/github.com/slok/kubewebhook
[godoc-image]: https://godoc.org/github.com/slok/kubewebhook?status.svg
[godoc-url]: https://godoc.org/github.com/slok/kubewebhook
[aw-url]: https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers
[docs]: https://slok.github.io/kubewebhook/
[red-metrics-url]: https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/
[prometheus-url]: https://prometheus.io/
[grafana-dashboard]: https://grafana.com/dashboards/7088
[opentracing-url]: http://opentracing.io
[mkcert]: https://github.com/FiloSottile/mkcert
[kind]: https://github.com/kubernetes-sigs/kind
[k3s]: https://k3s.io
[ngrok]: https://ngrok.com/
[mutating-cfg]: https://pkg.go.dev/github.com/slok/kubewebhook/pkg/webhook/mutating?tab=doc#WebhookConfig
[validating-cfg]: https://pkg.go.dev/github.com/slok/kubewebhook/pkg/webhook/validating?tab=doc#WebhookConfig
[runtime-unstructured]: https://pkg.go.dev/k8s.io/apimachinery/pkg/runtime?tab=doc#Unstructured