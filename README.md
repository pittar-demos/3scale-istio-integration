# 3Scale:  OpenShift Service Mesh (Istio) Integration

## Getting Started

To get started, you will need a few things in place.

1. An OpenShift 4.6+ cluster.
2. 3Scale 2.9+ installed in the cluster.
3. OpenShift ServiceMesh 2.0+ installed in the cluster.

## Installing Infrastructure

First, clone this repository and open the root directory:

```
git clone https://github.com/pittar-demos/3scale-istio-integration.git
cd 3scale-istio-integration
```

Make sure you have logged into your OpenShift cluster with an up-to-date verion of the `oc` cli as a *cluster-admin* user.

### Install 3Scale 2.9+

Use whatever method you like to install 3Scale 2.9+ in your cluster.  If you don't have access to a RWX storage class or an S3 bucket, you can use Noobaa by following these instructions.

1. Install the OpenShift Container Storage operator, either by using the UI or by running:

```
oc apply -k resources/ocs/operator
```

2. Install the Noobaa component of OCS:

```
oc apply -k resources/ocs/noobaa
```

Noobaa will take a few minutes to fully deploy.  When it's done, you will have a new `openshift-storage.noobaa.io` StorageClass available in your cluster.

3. Create a 3Scale project:

```
oc new-project 3scale
```

4. In the new 3Scale project, including the 3Scale Operator and an `ObjectBucketClaim`:

```
oc apply -k resources/3scale/operator
```

5. Wait until the ObjectBucketClaim is "Bound".  You can check this with the command:

```
oc get obc -n 3scale
NAME        STORAGE-CLASS                 PHASE   AGE
3scale-s3   openshift-storage.noobaa.io   Bound   5m10s
```

If your ObjectBucketClaim isn't *Bound* yet, then Noobaa probably hasn't finished deploying.  It can take some time, so go get a beverage and check again in a few minutes.

6. Now you can update the `s3-auth-secret.yaml` in `/resources/3scale/apimanager` with the following values:

```
echo "AWS_HOSTNAME: `oc get route -n openshift-storage s3 -o go-template='{{ .spec.host }}'`"

echo "AWS_ACCESS_KEY_ID: `oc get secret -n 3scale 3scale-s3 -o go-template='{{ .data.AWS_ACCESS_KEY_ID }}' | base64 -d`"

echo "AWS_SECRET_ACCESS_KEY: `oc get secret -n 3scale 3scale-s3 -o go-template='{{ .data.AWS_SECRET_ACCESS_KEY }}' | base64 -d`"
```

7. Now, update `apimanager.yaml` in `/resources/3scale/apimanager` by replacing `<cluster domain>` in the `wildcardDomain` field with your actual cluster domain.

8. Create your 3Scale API Manager instance:

```
oc apply -k resources/3scale/apimanager
```

3Scale should now be deploying in your 3scale namespace.  This will take a few moments to complete.

Once the deployment is complete, you can get the default password for the default **3scale** tenant admin portal by running:

```
echo "3Scale tenant admin password: `oc get secret -n 3scale system-seed -o go-template='{{ .data.ADMIN_PASSWORD }}' | base64 -d`"
```

The admin portal for the 3Scale tenant will be available at:
https://3scale-admin.apps.<cluster url>

The admin username is "admin".

## Deploy OpenShift Service Mesh

### Installing the Operators

Before installing OpenShift Service Mesh, you first need to install the following operators:
* [OpenShift Elasticsearch Operator](https://docs.openshift.com/container-platform/4.7/service_mesh/v2x/installing-ossm.html#jaeger-operator-install-elasticsearch_installing-ossm)
* [Jaeger Operator](https://docs.openshift.com/container-platform/4.7/service_mesh/v2x/installing-ossm.html#jaeger-operator-install_installing-ossm)
* [Kiali Operator](https://docs.openshift.com/container-platform/4.7/service_mesh/v2x/installing-ossm.html#ossm-install-kiali_installing-ossm)

Once those are in place, you can install the Service Mesh operator:
* [OpenShift ServiceMesh Operator](https://docs.openshift.com/container-platform/4.7/service_mesh/v2x/installing-ossm.html#ossm-install-ossm-operator_installing-ossm)

### Creating a Service Mesh Control Plane

When using OpenShift Service Mesh 2.0, the 3Scale Istio Adapter requires *Mixer*.  Enabling Mixer seems spins up extra components that user more resources than the default `LimitRanges` applied to new projects in an RHPDS cluster.  If you are using RHPDS, you may run into this issue if you use `oc new-projct` to create your istio control plane namespace.  If you use the following instructions, you won't have this issue (creating a `namespace` directly bypasses the new project template).

Install a `ServiceMeshControlPlane` instance with the following features enabled:
* 3Scale Istio Adapter Addon
* Policy set to `Mixer` with checks enabled.
* Telemetry set to `Mixer`

To do this, run the following command:

```
oc apply -k resources/ossm/istio-system
```

Once the control plane in `istio-system` has fully deployed, create a shared `Gateway` in the `istio-gateway` namespace:

```
oc apply -k resources/ossm/istio-gateway
```

Finally, create a `ServiceMeshMemberRoll` in the `istio-system` namespace that includes the `istio-gateway` namespace.

```
oc apply -f resources/ossm/member-roll-gateway/default-servicemeshmemberroll.yaml -n istio-system
```

You're done with Service Mesh for now.

### Configuring the 3Scale Istio Adapter

By enabling the 3Scale Addon when deploying the Service Mesh Control Plane, the 3Scale Istio Adapter will be automatically deployed for you.  All that's left to do is configure it.

First, you will need some information about your 3Scale tenant, specifically:
* Admin portal URL (e.g https://3scale-admin.apps.<cluster domain>)
* Admin access token.

To find your access token, login to your Admin tenant and navigate to the "Account Settings" page.
* In 3Scale 2.9 you can find this by clicking on the "Gear" icon at the top-right of the screen.
* In 33Scale 2.10, click on the dropdown in the top nav of the screen and select "Account Settings"

The access key will be in large font in the middle of your screen.

Edit the file `handler.yaml` in `/resources/3scale/istio-adapter` and:
* Update `access_token` to be your actual access token (no quotes)
* Update `system_url` to the url of your admin portal.

For example, the fields should look something like:

```
  params:
    access_token: 3d0dcb02694b494ae0c8f63970e3e513
    system_url: https://3scale-admin.apps.example.com
```

Apply this configuration by running:

```
oc apply -k resources/3scale/istio-adapter
```

3Scale should now be integrated with OpenShift Service Mesh!

## Deploy an App

### Basic App Deployment

First, let's deploy a small app that exposes a simple API and uses the standard OpenShift Router:

```
oc apply -k resources/app
```

Once the app is up and running, you can hit the path `https://<route url>/camel/books`.  There should be two entries in the database:

```
echo "Books endpoint: http://`oc get route -n bookstore bookstore -o go-template='{{ .spec.host }}'`/camel/books"
```

Now that you know the app works, let's remove the route and use Service Mesh instead.

### Adding Service Mesh

Using the first application, first delete the `Route` to the app:

```
oc delete route bookstore -n bookstore
```

Next, patch the `ServiceMeshMemberRoll` to include the `bookstore` namespace:

```
oc patch servicemeshmemberroll default -n istio-system \
    --type "json" \
    -p '[{"op":"add","path":"/spec/members/-","value":"bookstore"}]'
```

Patch the app `Deployment` and database `DeploymentConfig` with the annotation that will instruct Service Mesh to inject the sidecar.

```
oc patch deployment bookstore -n bookstore --patch "$(cat resources/app-istio/istio-sidecar-patch.yaml)"

oc patch dc bookstoredb -n bookstore --patch "$(cat resources/app-istio/istio-sidecar-patch.yaml)"
```

Finally, create a `VirtualService` to route traffic through the Istio Ingress Gateway:

```
oc apply -f resources/app-istio/virtualservice.yaml -n bookstore
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
