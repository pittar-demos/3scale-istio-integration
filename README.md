# 3Scale - Advanced Integrations Demo

This demo repo will cover the following topics:

1. 3Scale Integration with Red Hat Single Sign-On
2. 3Scale with OIDC auth for an API
3. 3Scale integration with OpenShift Service Mesh (Istio)

## Getting Started

To get started, you will need a few things in place.

1. An OpenShift 4.6+ cluster.
2. 3Scale 2.9+ installed in the cluster.
3. OpenShift ServiceMesh 2.0+ installed in the cluster.
4. Red Hat Single Sign-On installed in the cluter.

## Installing Infrastructure

### Install 3Scale 2.9+

Use whatever method you like to install 3Scale 2.9+ in your cluster.  If you don't have access to a RWX storage class or an S3 bucket, you can use Noobaa by following these instructions.

1. Install the OpenShift Container Storage operator, either by using the UI or by running:

```
oc apply -k 
```



## 3Scale Adapter

1. When deploying the SMCP, make sure to enable the 3Scale adapter, and change policy and telementry to "Mixer".
    * Addons -> 3Scale -> Install 3Scale Adapter = true
    * Policy -> Type of Policy -> Mixer
    * Telemetry -> Type of Telemetry -> Mixer

```
patch="$(oc get deployment -n "${BOOKINFO_NS}" bookstore --template='{"spec":{"template":{"metadata":{"labels":{ {{ range $k,$v := .spec.template.metadata.labels }}"{{ $k }}":"{{ $v }}",{{ end }}"service-mesh.3scale.net/service-id":"'"${SERVICE_ID}"'","service-mesh.3scale.net/credentials":"'"${HANDLER_NAME}"'"}}}}}' )"

{"spec":{
    "template":{
        "metadata":{
            "labels":{
                "app":"bookstore",
                "deploymentconfig":"bookstore",
                "service-mesh.3scale.net/credentials":"threescale",
                "service-mesh.3scale.net/service-id":"3",
                "version":"v1"
                }}}}}

oc patch -n "${BOOKINFO_NS}"  deployment bookstore --patch ''"${patch}"''


```

809ad3bcb6908c86685705a2ac79fe34

e43dfa62c026ab98090665ebe59028c7

46d1cb6778dbc0ce8737d1df112c9835