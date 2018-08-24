+++
categories = ["category", "subcategory"]
date = "2018-08-24T16:17:23+01:00"
draft = true
keywords = ["tech"]
tags = ["kubernetes"]
title = "Kubernetes Mutating Webhook Configuration"

+++
Sometimes in Kubernetes you want to have some control over what is allowed into your cluster and even some control over the properties.

With a Mutating Webhook Configuration you can do this. Essentially you can subscribe to different sorts of admission events on the API and each time an even happens it will forward it thought a webhook you can configure. I this way you can change the config before it is submitted to the nodes.

Its pretty straightforward and easy, you write a server that accepts JSON over HTTP and you respond with a base64 encoded patch to the JSON that describes the changes you want to make before it is admitted.

In the example I have been working on i have been using it to mutate the registry url on all deployments that meet certain criteria to a pre-defined registry name. This has allowed me to submit the same YAML to multiple clusters that have local registries and have the url updated to the local registry.

The code for this is fairly trivial (but took a long time to figure out from the docs!) and was created by Lawrence Gripper. You can see the blog about this here [https://blog.gripdev.xyz/2018/08/16/magic-mutatingadmissionscontrollers-and-kubernetes-mutating-pods-created-in-your-cluster/](https://blog.gripdev.xyz/2018/08/16/magic-mutatingadmissionscontrollers-and-kubernetes-mutating-pods-created-in-your-cluster/ "https://blog.gripdev.xyz/2018/08/16/magic-mutatingadmissionscontrollers-and-kubernetes-mutating-pods-created-in-your-cluster/")

The part of this that really started to stump me was how to deploy this into a real live cluster. The difficult part was that the hook will only work over HTTPS and I wanted this to be re-usable and be able to have a cert that could be provisioned on the fly.

I started by adding nginx into the pod and using that as the SSL termination and forwarding on all requests to the node app running in another container in the same pod. You can see here that its fairly straightforward and the pod just mounts 2 volumes, 1 containing the SSL from a secret (more on that later) and the other the contents of a config map containing the nginx config. [https://github.com/lawrencegripper/MutatingAdmissionsController/blob/master/chart/registry-rewriter/templates/deployment.yaml](https://github.com/lawrencegripper/MutatingAdmissionsController/blob/master/chart/registry-rewriter/templates/deployment.yaml "https://github.com/lawrencegripper/MutatingAdmissionsController/blob/master/chart/registry-rewriter/templates/deployment.yaml")

You also have to supply the root CA bundle to the webhook configuration which meant that self signed certificates were out (or at least beyond me to figure out!). You can see this config in the kubernetes YAML below. This is configured to use a function hosted on ngrok for development.

    apiVersion: admissionregistration.k8s.io/v1beta1
    kind: MutatingWebhookConfiguration
    metadata:
      name: local-repository-controller-webhook
    webhooks:
    - clientConfig:
        # ngrok public cabundle
        caBundle: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tDQpNSUlGcXpDQ0JKT2dBd0lCQWdJUURITW1IVnRRYmdwZGRocWhDUllpbVRBTkJna3Foa2lHOXcwQkFRc0ZBREJlDQpNUXN3Q1FZRFZRUUdFd0pWVXpFVk1CTUdBMVVFQ2hNTVJHbG5hVU5sY25RZ1NXNWpNUmt3RndZRFZRUUxFeEIzDQpkM2N1WkdsbmFXTmxjblF1WTI5dE1SMHdHd1lEVlFRREV4UlNZWEJwWkZOVFRDQlNVMEVnUTBFZ01qQXhPREFlDQpGdzB4T0RBek1USXdNREF3TURCYUZ3MHhPVEF6TVRJeE1qQXdNREJhTUJVeEV6QVJCZ05WQkFNTUNpb3VibWR5DQpiMnN1YVc4d2dnRWlNQTBHQ1NxR1NJYjNEUUVCQVFVQUE0SUJEd0F3Z2dFS0FvSUJBUURwSDlJbDZrUU05Y3JPDQpaaVJxeTJQUlg4Y2VZY3IwODdLL2l4L3I1SFNKZ2p4TUN3OXM2RTBEQ3lsMGNla0h1TmwvN2FvT0YrQ2NKbktkDQo4aEtQVThwVXVsUjJNdTE0NGY2V1Vkb29wcFYrUTY0V0pyc0w2M0xnVy9kaVJNRmZLOWppU3BTa2VWdmIwRFNTDQpEQnE4UTlNZ2theklrZUhGM1JadGEzUjZCOG5SWFVZNW1uU2pjY1hja2pUQlJLRE9hblNhOSt3cXF1MG45QzF5DQpFVVdzbTNibktQMFI2RDdZNU0zRXNsdG5XN3ZRTEhPSndialQzVC9BM00vSnk0dGJKanVSNUhHTm9ydG1XVS84DQpTRXZ0KzRTbi84MHhhNUlkL3NrblZmd0ZlU3ZITzVscEQzVXZuY1UyajNoeUpqK05jYnUyQUxvRGFiMlFuVjZGDQpaUTErQ1YybkFnTUJBQUdqZ2dLc01JSUNxREFmQmdOVkhTTUVHREFXZ0JSVHloZFovR3ZBQXlFdkdxN2txcWdjDQpnbGJhZFRBZEJnTlZIUTRFRmdRVXJibTJzUXpoK1NmSGVCOEd0Y1RHOE9xS091c3dId1lEVlIwUkJCZ3dGb0lLDQpLaTV1WjNKdmF5NXBiNElJYm1keWIyc3VhVzh3RGdZRFZSMFBBUUgvQkFRREFnV2dNQjBHQTFVZEpRUVdNQlFHDQpDQ3NHQVFVRkJ3TUJCZ2dyQmdFRkJRY0RBakErQmdOVkhSOEVOekExTURPZ01hQXZoaTFvZEhSd09pOHZZMlJ3DQpMbkpoY0dsa2MzTnNMbU52YlM5U1lYQnBaRk5UVEZKVFFVTkJNakF4T0M1amNtd3dUQVlEVlIwZ0JFVXdRekEzDQpCZ2xnaGtnQmh2MXNBUUl3S2pBb0JnZ3JCZ0VGQlFjQ0FSWWNhSFIwY0hNNkx5OTNkM2N1WkdsbmFXTmxjblF1DQpZMjl0TDBOUVV6QUlCZ1puZ1F3QkFnRXdkUVlJS3dZQkJRVUhBUUVFYVRCbk1DWUdDQ3NHQVFVRkJ6QUJoaHBvDQpkSFJ3T2k4dmMzUmhkSFZ6TG5KaGNHbGtjM05zTG1OdmJUQTlCZ2dyQmdFRkJRY3dBb1l4YUhSMGNEb3ZMMk5oDQpZMlZ5ZEhNdWNtRndhV1J6YzJ3dVkyOXRMMUpoY0dsa1UxTk1VbE5CUTBFeU1ERTRMbU55ZERBSkJnTlZIUk1FDQpBakFBTUlJQkJBWUtLd1lCQkFIV2VRSUVBZ1NCOVFTQjhnRHdBSFlBdTluZnZCK0tjYldUbENPWHFwSjdSemhYDQpsUXFyVXVnYWtKWmtObzRlMFlVQUFBRmlIRWtybVFBQUJBTUFSekJGQWlCZFRxNXZrdHJqRzZDWjltK24yRFk3DQptVEdndWJNVWpESHBFY1hJMHgzSU53SWhBTVFnZ0VpT3JGUlh0WHc4VnlZZzRlNVpGUjJhbnRCakdnWE5tWXJCDQp2K24xQUhZQWIxTjJyREh3TVJuWW1RQ2tVUlgvZHhVY0Vka0N3UUFwQm8yeUNKbzMyUk1BQUFGaUhFa3IvZ0FBDQpCQU1BUnpCRkFpRUExUTRVNmVyQ3pPWVJ1OW55UFh6MnFWY2RnL3BwVVdPREVyNWNTbEdJeVJ3Q0lIM2o0ZWMzDQpDbXROaitaN0l1T3R3YjZwMmNPSGlrRHBvNDFWTmlvM3pBcm1NQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUJBUUE1DQpCM25FZE5iU3huYmVrTUtHeFo3QlFoTi9uVVRiN0QraVMxMnIxK3BUL0VqdzhUUmEzdWVEWjdWcnNxU3AzaE1tDQpXMmYwMjJkanpocnhFSGkyYWJGb0VUS0Q1dUFSR1F2dDVML2h2bEhmZGtyZVMxSWZxK2YyUVllcU9zcVJlUUxHDQpOUTR6ekRXS1gxeTRBNGpQSi9uQ0diNk16UnlIUytHemhKMU50YnozNDJhQ2IxWER6aXBNSVpZUXhZTDFISjVTDQo5VzRYOGg2c2w4NDJXTmVvajBtYU10ZmVpajhURGlnUnJBUFE5anRYK29lOG1mdG4zOVVyK1ZvRC9FaTl3cWhMDQo1Rlh2WGR3MWxmT0RLNEpzd2JtTzdXMExwNzJnMDBHY3k5a1h1MUpoVkQvVDFFamh0NW92YjdIQ1VWYXkzT0plDQpKUE5Pemh6eTUzUzUzS0hiN0gvSQ0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQ0K
        url:  https://5db0a78d.ngrok.io/
      failurePolicy: Fail
      name: 5db0a78d.ngrok.io
      namespaceSelector: {}
      rules:
      - apiGroups:
        - apps
        - ""
        apiVersions:
        - v1
        operations:
        - CREATE
        resources:
        - deployments

I then spent some time on the realisation that you could get the Kubernetes cluster itself to issue out certificates from a CSR. This again is fairly trivial and can be done with the use of another Kubernetes YAML like this

    apiVersion: certificates.k8s.io/v1beta1
    kind: CertificateSigningRequest
    metadata:
      name: my-svc.my-namespace
    spec:
      groups:
      - system:authenticated
      request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1dUQ0NBVUVDQVFBd0ZERVNNQkFHQTFVRUF3d0pZV0V1WW1JdWMzWmpNSUlCSWpBTkJna3Foa2lHOXcwQgpBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUFwOC8xaEN6cEZpenhjL08vbG1vWHhMc3NxMzl0VGNyc0xXV2I2Z0Q4CkpyUEQ5SVlma2VPOWpteDdkR2VRbDcveUNOcU1ydE43R2RZenNYbTJpTkRJZFZTNjM4YlU3aU54RksyS2FBRm0KcHE2RnpNQ1htUHNONXNVRGFkY2x2ZkdsNnBQWEE5L2N4eHpKRzlTdEIraDM0VHJISkl3SXVaQjBSWE9LTkZPKwpNcHBvVWtTOG9Nd0ZCWUN1d3liZHMzd0p4NXQvZk02V0FZZ0ZGSHBqRXVRL1NZSlUyNXFYUUloNDhSL2plZ2M5CkdGc1BoUnlWWmNzR2VxQmIzUVA3b04rRFFaaHpMcHJTUU5PWno5MXRrRlhUK0htK1NBUzNwTVJTUjY1MEFzUG8KVjdwVkpaV3VPVW9aL0tkVnI5TXZDbjdvS3dLcHkrNmVkcHg1OFVLbG1CTmR0UUlEQVFBQm9BQXdEUVlKS29aSQpodmNOQVFFTEJRQURnZ0VCQUZodmozSEVmcG5kOHFRRXF5ZWpxL2hlaGRySlZiRVFtbFRremNMZmhLMkpFUjVxCklDMFEreHIvNjVzVzBDMVJNN0ROaXErU0JBcGJMeDRremR2RXhmcFBvL3pXTVc4WkFZd0hGMC9pZHlBbWtpSjkKcklQV2xWTkdVT1pYclF3clBBMDBWNmVLbytIMVV5U050SWJibmxPR1dIOFM1M0NiS1o0TjJ4WldENDVpb0g4RQoyUEhkU3RybVFzR2VtQXo3bVArdlR2NHlwVHNyRHpLTzg0VHhQd3RieWNRWjhtakdmQU1PUVZjUVBINFZKSEdKCjJTelNNK25OZHpOWXdNeTlLeURpWkVQWGZVKzFBbnJtTG1WRHI5b2srUm9lUkdTRllvTTQzY0dQcTgvemxkR3MKL0l2UFY1bEIzYXVDMXRvMW5uSUlJQWc2Q1BMUWpVTWRmRnVXUG9rPQotLS0tLUVORCBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0K
      usages:
      - digital signature
      - key encipherment
      - server auth

You just need to provide a base64 encoded representation of your CSR. Once you have file you can submit it to the API with `kubectl apply` and you will get the following when you run `kubectl get csr`

    NAME                  AGE       REQUESTOR   CONDITION
    my-svc.my-namespace   7s        client      Pending

You can see this certificate is pending approval from a user within kubernetes before it is issued out. You will need to run `kubectl certificate approve` to approve the csr and issue out the cert. Once approved you can get the certificate from the API by running `kubectl get csr my-svc.my-namespace -o jsonpath='{.status.certificate}'` and this will return the certificate in base64.

You can output the certificate to a file by using

    kubectl get csr my-svc.my-namespace -o jsonpath='{.status.certificate}' | openssl base64 -d -A -out server-cert.pem

Presuming you still have the key that you used to generate the CSR you can create a secret with the following command

    kubectl create secret tls my-new-cert \
                        --key=server-key.pem \
                        --cert=server-cert.pem \
                        --dry-run -o yaml |
                    kubectl -n default apply -f -

Following this process will give you a nice cluster signed cert outputted into a secret that could be consumed from any pod in that namespace. Making valid HTTPS possible within the cluster.

To create the webhook you would need to get the caBundle from the cluster using the following

    kubectl get configmap -n kube-system extension-apiserver-authentication -o=jsonpath='{.data.client-ca-file}' | base64 | tr -d '\n'

# All of that but with a touch of automation...

So the above seems very manual and has lots of steps, including a manual approval step, that i really want to be automatic as part of a helm chart or similar.

I chose to use a Job to create and issue out the certificate and save it into a secret for the deployment hosting the webhook to pick up.

I ended up with a job like this, that re-uses a script very close to that of istio.

    apiVersion: batch/v1
    kind: Job
    metadata:
      name: {{ template "registry-rewriter.fullname" . }}
      namespace: kube-system
      labels:
        app: {{ template "registry-rewriter.name" . }}
        chart: {{ template "registry-rewriter.chart" . }}
        release: {{ .Release.Name }}
        heritage: {{ .Release.Service }}
    spec:
      template:
        spec:
          containers:
          - name: pi
            image: pivotalservices/pks-kubectl:1.2.0-Beta-1
            command: ["/bin/bash"]
            args:
              - "-c"
              - |
                #!/bin/bash
    
                set -e
    
                usage() {
                    cat <<EOF
                Generate certificate suitable for use with an sidecar-injector webhook service.
                This script uses k8s' CertificateSigningRequest API to a generate a
                certificate signed by k8s CA suitable for use with sidecar-injector webhook
                services. This requires permissions to create and approve CSR. See
                https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster for
                detailed explantion and additional instructions.
                The server key/cert k8s CA cert are stored in a k8s secret.
                usage: ${0} [OPTIONS]
                The following flags are required.
                      --service          Service name of webhook.
                      --namespace        Namespace where webhook service and secret reside.
                      --secret           Secret name for CA certificate and server certificate/key pair.
                EOF
                    exit 1
                }
    
                while [[ $# -gt 0 ]]; do
                    case ${1} in
                        --service)
                            service="{{ template "registry-rewriter.fullname" . }}-service"
                            shift
                            ;;
                        --secret)
                            secret="{{ template "registry-rewriter.fullname" . }}"
                            shift
                            ;;
                        --namespace)
                            namespace="kube-system"
                            shift
                            ;;
                        *)
                            usage
                            ;;
                    esac
                    shift
                done
    
                [ -z ${service} ] && service={{ template "registry-rewriter.fullname" . }}-service
                [ -z ${secret} ] && secret={{ template "registry-rewriter.fullname" . }}
                [ -z ${namespace} ] && namespace=kube-system
    
                if [ ! -x "$(command -v openssl)" ]; then
                    echo "openssl not found"
                    exit 1
                fi
    
                csrName=${service}.${namespace}
                tmpdir=$(mktemp -d)
                echo "creating certs in tmpdir ${tmpdir} "
    
                cat <<EOF >> ${tmpdir}/csr.conf
                [req]
                req_extensions = v3_req
                distinguished_name = req_distinguished_name
                [req_distinguished_name]
                [ v3_req ]
                basicConstraints = CA:FALSE
                keyUsage = nonRepudiation, digitalSignature, keyEncipherment
                extendedKeyUsage = serverAuth
                subjectAltName = @alt_names
                [alt_names]
                DNS.1 = ${service}
                DNS.2 = ${service}.${namespace}
                DNS.3 = ${service}.${namespace}.svc
                EOF
    
                openssl genrsa -out ${tmpdir}/server-key.pem 2048
                openssl req -new -key ${tmpdir}/server-key.pem -subj "/CN=${service}.${namespace}.svc" -out ${tmpdir}/server.csr -config ${tmpdir}/csr.conf
    
                # clean-up any previously created CSR for our service. Ignore errors if not present.
                kubectl delete csr ${csrName} 2>/dev/null || true
    
                # create  server cert/key CSR and  send to k8s API
                cat <<EOF | kubectl create -f -
                apiVersion: certificates.k8s.io/v1beta1
                kind: CertificateSigningRequest
                metadata:
                  name: ${csrName}
                spec:
                  groups:
                  - system:authenticated
                  request: $(cat ${tmpdir}/server.csr | base64 | tr -d '\n')
                  usages:
                  - digital signature
                  - key encipherment
                  - server auth
                EOF
    
                # verify CSR has been created
                while true; do
                    kubectl get csr ${csrName}
                    if [ "$?" -eq 0 ]; then
                        break
                    fi
                done
    
                # approve and fetch the signed certificate
                kubectl certificate approve ${csrName}
                # verify certificate has been signed
                for x in $(seq 10); do
                    serverCert=$(kubectl get csr ${csrName} -o jsonpath='{.status.certificate}')
                    if [[ ${serverCert} != '' ]]; then
                        break
                    fi
                    sleep 1
                done
                if [[ ${serverCert} == '' ]]; then
                    echo "ERROR: After approving csr ${csrName}, the signed certificate did not appear on the resource. Giving up after 10 attempts." >&2
                    exit 1
                fi
    
                echo ${serverCert} | openssl base64 -d -A -out ${tmpdir}/server-cert.pem
    
    
                # create the secret with CA cert and server cert/key
                kubectl create secret tls ${secret} \
                        --key=${tmpdir}/server-key.pem \
                        --cert=${tmpdir}/server-cert.pem \
                        --dry-run -o yaml |
                    kubectl -n ${namespace} apply -f -
          restartPolicy: Never
      backoffLimit: 4

For the purposes of testing I didn't bake this into the container and just inlined the script in to the job definition. Its a bit messy but allowed me to test a lot of things out without the loop of creating a container and deploying it. If I continue to find this useful i will pull it out into its own repo and make it more re-usable.

So now we have a re-usable webhook that generates its own certificate and takes in properties to re-write the image name. A job runs on deployment to provision an SSL certificate and the deployment uses it once it becomes available.

You can see the details here [https://github.com/lawrencegripper/MutatingAdmissionsController#helm-chart](https://github.com/lawrencegripper/MutatingAdmissionsController#helm-chart "https://github.com/lawrencegripper/MutatingAdmissionsController#helm-chart")

To deploy this into you own cluster you would use the command below

    helm install <url of release> --name registry-rewiter --set containerRegistryUrl=someurl.com,caBundle=<rootbundleasbase64>

Releases can be found in the github releases.