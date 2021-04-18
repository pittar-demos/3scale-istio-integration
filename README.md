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

4. In the new 3Scale project, create the 3Scale Operator and an `ObjectBucketClaim`:

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

The admin portal for the 3Scale tenant will be available at: `https://3scale-admin.apps.<cluster domain>`

The admin username is "admin".

### Deploy OpenShift Service Mesh

#### Installing the Operators

Before installing OpenShift Service Mesh, you first need to install the following operators:
* [OpenShift Elasticsearch Operator](https://docs.openshift.com/container-platform/4.7/service_mesh/v2x/installing-ossm.html#jaeger-operator-install-elasticsearch_installing-ossm)
* [Jaeger Operator](https://docs.openshift.com/container-platform/4.7/service_mesh/v2x/installing-ossm.html#jaeger-operator-install_installing-ossm)
* [Kiali Operator](https://docs.openshift.com/container-platform/4.7/service_mesh/v2x/installing-ossm.html#ossm-install-kiali_installing-ossm)

Once those are in place, you can install the Service Mesh operator:
* [OpenShift ServiceMesh Operator](https://docs.openshift.com/container-platform/4.7/service_mesh/v2x/installing-ossm.html#ossm-install-ossm-operator_installing-ossm)

#### Creating a Service Mesh Control Plane

When using OpenShift Service Mesh 2.0, the 3Scale Istio Adapter requires *Mixer*.  Enabling Mixer seems spins up extra components that use more resources than the default `LimitRanges` applied to new projects in an RHPDS cluster allows.  If you are using RHPDS, you may run into this issue if you use `oc new-projct` to create your Istio control plane namespace.  If you use the following instructions, you won't have this issue (creating a `namespace` directly bypasses the new project template).

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

#### Configuring the 3Scale Istio Adapter

By enabling the 3Scale Addon when deploying the Service Mesh Control Plane, the 3Scale Istio Adapter will be automatically deployed for you.  All that's left to do is configure it.

First, you will need some information about your 3Scale tenant, specifically:
* Admin portal URL (e.g `https://3scale-admin.apps.<cluster domain>`)
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

**Important:** One thing that's easy to overlook is *the configuration of the `ports` of your `Service` is very important when using Istio*!  In order for Istio to select the correct `port` on your `Service` you need to either *prefix the port name with the correct protocol name*, OR *add a `appProtocol` field specifying the protocol*.

I find the `appProtocol` field to be be more obvious and explicit than a naming convention, so that's what I've used.  You can see this by looking at the service object for the [application](https://github.com/pittar-demos/3scale-istio-integration/blob/main/resources/app/bookstore-svc.yaml#L20) and the [database](https://github.com/pittar-demos/3scale-istio-integration/blob/main/resources/app/bookstoredb-service.yaml#L11).

For example:

```
spec:
  selector:
    app: bookstore
    deploymentconfig: bookstore
  ports:
  - appProtocol: http
    name: 8080-tcp
    port: 8080
```

### Adding Service Mesh

Using the same application, first delete the `Route` to the app:

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

Finally, create a `VirtualService` to route traffic through the Istio Ingress Gateway to the bookstore app:

```
oc apply -f resources/app-istio/virtualservice.yaml -n bookstore
```

You should now be able to hit your API endpoint through the Istio Ingress Gateway instead of it's original (and now deleted) Route.  You can find this url by running the following command:

```
echo "Istio Ingress Gateway URL: http://`oc get route -l maistra.io/gateway-name=default-gateway -o go-template='{{range .items}}{{ .spec.host }}{{end}}'`/camel/books"
```

This should confirm that your application is properly configured and working with OpenShift Service Mesh.  If you like, you can also login to Kiali to see the network graph for your application.

## Create a New Product in 3Scale

Finally, it's time to create a new product in 3Scale!

Log back into your 3Scale admin portal.  From the main dashboard:

1. Click **Create Product**.
2. Give it a name (e.g. Bookstore Istio) and a unique system name and click **Create Product**.
3. On the next screen, make note of the *Service ID*.  It will be in the bottom-right panel.  It will also be at the end of the current URL in your browser.  In this example, the *Service ID* is **3**.

The next few steps are a work around for a 3Scale bug!  Hopefully this will be resolved soon (still and issue in 3Scale 2.10.0):

4. Select *any* backend for the app.  It doesn't matter because it won't be used.
5. Go to *Settings* and select "APIcast 3Scale Managed".  Scroll to the bottom and click **Update Product**.
6. Click on *Configuration* and promote your product all the way to Prodution.
7. Click on **Settings** again. This time change the "Deployment" to **Istio**.  Leave authentication as **API Key**.
6. Click **Update Product**.

Now you need an **Application Plan** and a registered **Application**.

1. From the left nav, select **Applications -> Application Plans**.
2. Click the green **Create Application Plan** link near the right side of the screen.
3. Give your plan a Name and System Name (e.g. "Istio Basic" and "istio_basic") and click **Create Application Plan**.
4. Click the **Publish** link.

Next, sign up for this plan as the default *Developer* account.

1. From the drop down list at the top of the screen, select **Audience**.
2. Click on the *Developer* account.
3. Click on the `Application` bread crumb at the top of the screen (it will say `1 Application` if this is a new tenant).
4. Click the green **Create Application** link near the top-right of the screen.
5. Select the `Bookstore Istio -> Istio Basic` plan from the drop down.
6. Give your app a name and description and click **Create Application**
7. **Take note of your User Key!!**

You now have a new 3Scale product, but there is still some config to do to your application `Deployment` to complete the configuration.

## Add 3Scale Istio Adapter Labels to App Deployment

The final step is to add two new labels to the application `Deployment`.  These are:

```
service-mesh.3scale.net/credentials : threescale
service-mesh.3scale.net/service-id: '3'
```

The first label tells the 3Scale Istio Adapter to use the handler named `threescale` for credential requests.  

The second label tells the adapter which 3Scale service this application belongs to.  **The value for the `service-id` label should be the ID of the 3Scale service you just created**.

Update this patch to match the Service ID from your 3Scale service then apply it:

```
oc patch deployment bookstore -n bookstore \
    --type "json" \
    -p '[{"op":"add","path":"/spec/template/metadata/labels/service-mesh.3scale.net~1credentials","value":"threescale"},
         {"op":"add","path":"/spec/template/metadata/labels/service-mesh.3scale.net~1service-id","value":"3"}]'
```

This should trigger a rollout of your application.  

## Testing Your Application

Once has finished rolling out, try hitting the same "books" endpoint through the same Istio Ingress Gateway URL again.  If everything is working, you should get the following error:

```
UNAUTHENTICATED:threescale.handler.istio-system:no auth credentials provided or provided in invalid location
```

This means 3Scale is protecting your API!  Add the `user_key` parameter to the end of your url:

```
<endpoint url>?user_key=<your application key>
```

Now you should be authenticated and getting results!

Your appication is now integrated with OpenShift Service Mesh (Istio) and being managed by 3Scale.