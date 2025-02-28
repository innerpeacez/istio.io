---
title: Plugging in External CA Key and Certificate
description: Shows how operators can configure Citadel with existing root certificate, signing certificate and key.
weight: 10
keywords: [security,certificates]
aliases:
    - /docs/tasks/security/plugin-ca-cert/
---

This task shows how operators can configure Citadel with existing root certificate, signing certificate and key.

By default, Citadel generates self-signed root certificate and key, and uses them to sign the workload certificates.
Citadel can also use the operator-specified certificate and key to sign workload certificates, with
operator-specified root certificate. This task demonstrates an example to plug certificates and key into Citadel.

## Before you begin

* Set up Istio by following the instructions in the [quick start](/docs/setup/install/kubernetes/):

{{< tip >}}
You can use [authentication policy](/docs/concepts/security/#authentication-policies) to configure mutual TLS for all/selected services in a namespace (repeated for all namespaces to get global setting). See [authentication policy task](/docs/tasks/security/authentication/authn-policy/)
{{< /tip >}}

## Plugging in the existing certificate and key

Suppose we want to have Citadel use the existing signing (CA) certificate `ca-cert.pem` and key `ca-key.pem`.
Furthermore, the certificate `ca-cert.pem` is signed by the root certificate `root-cert.pem`.
We would like to use `root-cert.pem` as the root certificate for Istio workloads.

In the following example,
Citadel's signing (CA) certificate (`ca-cert.pem`) is different from root certificate (`root-cert.pem`),
so the workload cannot validate the workload certificates directly from the root certificate.
The workload needs a `cert-chain.pem` file to specify the chain of trust,
which should include the certificates of all the intermediate CAs between the workloads and the root CA.
In our example, it contains Citadel's signing certificate, so `cert-chain.pem` is the same as `ca-cert.pem`.
Note that if your `ca-cert.pem` is the same as `root-cert.pem`, the `cert-chain.pem` file should be empty.

These files are ready to use in the `samples/certs/` directory.

  {{< tip >}}
  The default Citadel installation sets [command line options](/docs/reference/commands/istio_ca/index.html) to configure the location of certificates and keys based on the predefined secret and file names used in the command below (i.e., secret named `cacert`, root certificate in a file named `root-cert.pem`, Citadel key in `ca-key.pem`, etc.)
  You must use these specific secret and file names, or reconfigure Citadel when you deploy it.
  {{< /tip >}}

The following steps enable plugging in the certificates and key into Citadel:

1.  Create a secret `cacert` including all the input files `ca-cert.pem`, `ca-key.pem`, `root-cert.pem` and `cert-chain.pem`:

    {{< text bash >}}
    $ kubectl create secret generic cacerts -n istio-system --from-file=samples/certs/ca-cert.pem \
        --from-file=samples/certs/ca-key.pem --from-file=samples/certs/root-cert.pem \
        --from-file=samples/certs/cert-chain.pem
    {{< /text >}}

1.  Redeploy Citadel with `global.mtls.enabled` set to `true` and `security.selfSigned` to `false`.
    Citadel will read certificates and key from the secret-mount files.

    {{< text bash >}}
    $ istioctl manifest apply --set values.global.mtls.enabled=true,values.security.selfSigned=false
    {{< /text >}}

1.  To make sure the workloads obtain the new certificates promptly,
    delete the secrets generated by Citadel (named as `istio.\*`).
    In this example, `istio.default`. Citadel will issue new certificates for the workloads.

    {{< text bash >}}
    $ kubectl delete secret istio.default
    {{< /text >}}

## Verifying the new certificates

In this section, we verify that the new workload certificates and root certificates are propagated.
This requires you have `openssl` installed on your machine.

1. Deploy the Bookinfo application following the [instructions](/docs/examples/bookinfo/).

1.  Retrieve the mounted certificates.
    In the following, we take the ratings pod as an example, and verify the certificates mounted on the pod.

    Set the pod name to `RATINGSPOD`:

    {{< text bash >}}
    $ RATINGSPOD=`kubectl get pods -l app=ratings -o jsonpath='{.items[0].metadata.name}'`
    {{< /text >}}

    Run the following commands to retrieve the certificates mounted on the proxy:

    {{< text bash >}}
    $ kubectl exec -it $RATINGSPOD -c istio-proxy -- /bin/cat /etc/certs/root-cert.pem > /tmp/pod-root-cert.pem
    {{< /text >}}

    The file `/tmp/pod-root-cert.pem` contains the root certificate propagated to the pod.

    {{< text bash >}}
    $ kubectl exec -it $RATINGSPOD -c istio-proxy -- /bin/cat /etc/certs/cert-chain.pem > /tmp/pod-cert-chain.pem
    {{< /text >}}

    The file `/tmp/pod-cert-chain.pem` contains the workload certificate and the CA certificate propagated to the pod.

1.  Verify the root certificate is the same as the one specified by operator:

    {{< text bash >}}
    $ openssl x509 -in @samples/certs/root-cert.pem@ -text -noout > /tmp/root-cert.crt.txt
    $ openssl x509 -in /tmp/pod-root-cert.pem -text -noout > /tmp/pod-root-cert.crt.txt
    $ diff /tmp/root-cert.crt.txt /tmp/pod-root-cert.crt.txt
    {{< /text >}}

    Expect the output to be empty.

1.  Verify the CA certificate is the same as the one specified by operator:

    {{< text bash >}}
    $ tail -n 23 /tmp/pod-cert-chain.pem > /tmp/pod-cert-chain-ca.pem
    $ openssl x509 -in @samples/certs/ca-cert.pem@ -text -noout > /tmp/ca-cert.crt.txt
    $ openssl x509 -in /tmp/pod-cert-chain-ca.pem -text -noout > /tmp/pod-cert-chain-ca.crt.txt
    $ diff /tmp/ca-cert.crt.txt /tmp/pod-cert-chain-ca.crt.txt
    {{< /text >}}

    Expect the output to be empty.

1.  Verify the certificate chain from the root certificate to the workload certificate:

    {{< text bash >}}
    $ head -n 21 /tmp/pod-cert-chain.pem > /tmp/pod-cert-chain-workload.pem
    $ openssl verify -CAfile <(cat @samples/certs/ca-cert.pem@ @samples/certs/root-cert.pem@) /tmp/pod-cert-chain-workload.pem
    /tmp/pod-cert-chain-workload.pem: OK
    {{< /text >}}

## Cleanup

*   To remove the secret `cacerts`:

    {{< text bash >}}
    $ kubectl delete secret cacerts -n istio-system
    {{< /text >}}

*   To remove the Istio components: follow the [uninstall instructions](/docs/setup/install/kubernetes/#uninstall) to remove.

