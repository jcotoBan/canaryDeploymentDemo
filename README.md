# canaryDeploymentDemo

Just a simple canary deployment for beginners like me :) 

Deployment strategies on kubernetes – Canary

The main idea behind canary strategy is to deploy new software only to a small subset of users and then increase this percentage of users.

It provides the following advantages:

→ It is less risky since new features are rolled out only to a small percentage of users.

→ It lets you keep the pace with new feature adoption and receive timely feedback without disruption of the production current release.

→ It only requires one production environment, which makes it cheaper than other strategies like blue/green for example.



How does it work:

<img width="529" alt="image" src="https://user-images.githubusercontent.com/73861871/208133358-e935748d-38fd-41a2-acba-bf6d9fd720c0.png">


For a canary deployment we will require several instances, as well as a load balancer. The idea is to add new instances (for example pods) or route more percentage of the traffic to the new software progressively with a load balancer as a front door, so that some of the user base get to see these new features. 

As the new changes are adopted or fixed according to the feedback received the number of instances with the new release are increased while the old ones start to be decommissioned. Container technologies like kubernetes makes this particularly smooth and easy to deploy due to its portable and flexible nature.
A canary deployment in practice with kubernetes.

In this post, two ways of setting up a deployment will be shown. One is more basic, while the other is more sophisticated.

# Basic deployment

The most rudimentary way to apply a canary strategy is by having separate deployments with a certain amount of pods each and a service that routes to each of the deployments. In order to balance the load accordingly more replicas will be added to one deployment (this will be the beta or current release) while the other will be left with less replicas (the alpha one). To differentiate the apps, a different color and title are added to a simple webpage:

<img width="362" alt="image" src="https://user-images.githubusercontent.com/73861871/208133458-6027e0d9-3e7f-4f17-991e-8acc73ceed58.png">

<img width="408" alt="image" src="https://user-images.githubusercontent.com/73861871/208133466-79b483e1-33f9-4cde-bdaa-927c1960fc31.png">
 
For this setup we will only need to apply the following files:

alpha_App.yaml
beta_App.yaml
canaryService.yaml

Some important notes on the files:

The environment is set to allocate 10 pods in total, 7 for the beta version and 3 for the alpha. This number is defined by the replicas field on the spec section of the deployment. 

There is an environment variable set called APP_COLOR, it defines the background color of the html index, and you can customize it as required. (Shoutout to kodekloud for the app!)

Notice how the deployment’s pod templates have an extra label called deployment: canary, this label will be used by the service in order to route the traffic to the deployments.

For ease of testing, a nodeport kind of service will be used, which means we can access the deployments from the ip of the nodes (any of the nodes you have set on your cluster)  plus the nodePort specified, in this case 31000.

139.144.188.177:31000 for example.

To Deploy the objects:

```
kubectl apply -f alpha_App.yaml
```

Or, if you want to use a custom namespace (default will be used if not specified)

kubectl apply -f alpha_App.yaml -n=canaryns

<img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208134071-f89e5350-d7a1-48bd-8275-aa9b35e4a2e2.png">


 When the two deployments are created, you can verify them with the kubectl get deploy command:

 <img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208134097-2e44858e-5e95-483e-94b2-e1813bdf7cae.png">


Then you should see 7 beta pods and 3 alpha with kubectl get pods command (make sure to edit the replica numbers):

<img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208134131-a738d20b-55e8-4ebc-b229-a8e56c2b6d91.png">

 
Then, we proceed to apply the service as well:

 <img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208134186-fdbacb93-dd2f-4c1f-8cc2-13908e5074dc.png">


And then with the command kubectl get svc canary, we will verify it:

 <img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208134201-30cd8313-72d2-4d81-876e-0b226cb6a4bc.png">


TYPE should be NodePort, and the port binding should be 8080:31000.

Then, we will verify the endpoints for the canary deployment, there should be 10 (one per pod):

 <img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208134238-2357db59-a470-40dc-9b5c-cf9341b196cb.png">


If the endpoints are not correctly displayed the service’s routing wont work.

Then, if everything at this point has set correctly, you can check any of your node ips, go to your browser and verify the service is working:

 

 <img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208134263-a97876a3-311c-4018-968a-44ec3fef027a.png">
<img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208134281-1caa54b2-b207-471e-a430-3e40f1d87ff1.png">



To control the number of replicas you can edit the files and apply them again, or edit the runtime deployment with kubectl edit deploy command:


<img width="416" alt="image" src="https://user-images.githubusercontent.com/73861871/208134309-dd126d5c-da08-494f-b3e4-fcab512f0124.png">

<img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208134358-859ec5dd-6329-43f5-811a-8225f410add7.png">



# A more sophisticated way of canary deployment with istio.

<img width="381" alt="image" src="https://user-images.githubusercontent.com/73861871/208134546-4940f89f-e005-4262-b22f-50cf4f864ca2.png">


By using the previous method, you may have noticed that it is not a very efficient way to distribute the traffic, as well the distribution is not exactly made the way expected. A tool like Istio can help us to set up more advanced traffic distribution rules.

Istio is a service mesh that allows you to add capabilities to a microservices infrastructure, details will be avoided (you can visit https://istio.io/ if curious), but for now the only capability we will care about is traffic distribution. 

Just as we setup a deployment and a service, we will do it with with this lab as well.


Nothing changes that much until this point, except for the fact that each deployment will have the same number of replicas since the traffic distribution will be performed by Istio.

Now, we will need to install Istio on our cluster. We will do it with the help of kubernetes package manager, helm.

How to install helm: https://helm.sh/docs/intro/install/
Configure the Istio repo:

helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

Create a namespace for the istio setup:

kubectl create namespace istio-system

Install base chart:

helm install istio-base istio/base -n istio-system

Install the Istio discovery chart which deploys the istiod service:

helm install istiod istio/istiod -n istio-system --wait

Install the ingress gateway:

kubectl create namespace istio-ingress
helm install istio-ingressgateway istio/gateway -n istio-ingress

To verify the istio base installation:

 <img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208134636-7b04403e-b431-467d-91bf-a553e4f00b4e.png">


Verify the gateway was installed and is running (it is a service), you can run command:

kubectl get svc istio-ingressgateway -n=istio-ingress

 <img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208134708-9beb7cd5-c91f-4630-8633-12239bb55a96.png">


Then, we will need to create 3 new objects for the gateway to work and route the traffic correctly. These objects are specific to the Istio API.

Destination Rule: Is just a way to tell Istio the routes for our application. See how the alpha and beta labels are specified as part of the canary host (in this case, canary is the name of the service)


Gateway: It just specifies a load balancer for the ingress gateway, in this case port 80 on http protocol will route to the ingress gateway


Virtual Service: Defines a set of traffic routing rules to apply when a host is addressed. Each routing rule defines matching criteria for traffic of a specific protocol. If the traffic is matched, then it is sent to a named destination service.

It uses the canary gateway we created.

As you can see on the file, we set a weight on the hosts, so 90% goes to beta release and 10% go to alpha no matter the number of replicas for each deployment. So 9 times out of 8, we should see the blue beta website.

After you have applied these resources, you can verify them with kubectl:

<img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208134929-85444eaf-9880-4f87-92c4-463e55289076.png">

<img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208134940-17aac112-c8c9-4674-b322-3fec010faad0.png">

 <img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208134949-f55c046c-2d74-4f32-8573-56c1ad887602.png">


Then, how do we access the service?

If you are running a cloud native cluster, like Linode kubernetes engine, you should get an external ip when checking the ingress gateway service:

 <img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208134964-4a560538-eab9-4536-82e0-6b36f9fe4c0e.png">


In this case, it was runned on a manually created cluster, so the service was edited to be type node port. Which means it can be accessed on a node public ip.

So, what should you expect to see? Even though each deployment has the same number of replicas, 90% of the time we should get the traffic routed to our beta release:

<img width="468" alt="image" src="https://user-images.githubusercontent.com/73861871/208135084-886f799f-04dd-4bb7-8e98-25a2da0b1836.png">




