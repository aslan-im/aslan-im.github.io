# Converting a Docker Compose File to Kubernetes Objects: Jellyfin Example
Today, we will convert a Docker Compose file into Kubernetes objects using Jellyfin as an example.

Initially, the Docker Compose file looked like this:
```yaml
version: '3.5'
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    hostname: jellyfin
    user: '1000:1000'
    volumes:
      - /storage/config:/config
      - /storage/cache:/cache
      - /storage/media:/media
    restart: 'unless-stopped'
    ports:
      - 8096:8096
    extra_hosts:
      - "host.docker.internal:host-gateway"
```
Based on the Compose file, we will need the following Kubernetes objects:
	•	Namespace
	•	Deployment: For managing the pod with the application.
	•	ClusterIP Service: To make the application accessible within the cluster.
	•	Ingress: To make the application accessible via a domain name.

## Namespace and Deployment
Namespace can be created by one command:
```bash
$ kubectl create namespace jellyfin
```

Next we need to create the core manifest for deployment:
```bash
$ kubectl -n jellyfin create deploy jellyfin --image=jellyfin/jellyfin:10.10.3
```

Now we have the deployment.yaml file
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: jellyfin
  name: jellyfin
  namespace: jellyfin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jellyfin
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: jellyfin
    spec:
      containers:
      - image: jellyfin/jellyfin:10.10.3
        name: jellyfin
        resources: {}
status: {}
```

Ports,  Environment Variables, Volumes and Volume Mounts need to be added to the deployment in container section like this:
```yaml
containers:
- name: jellyfin
  image: jellyfin/jellyfin:10.10.3
  ports:
  - containerPort: 8096 # port for accessing webui
    name: http-tcp
    protocol: TCP

  env:
  - name: JELLYFIN_PublishedServerUrl
    value: "http://<domain>:80" # change to your domain
  - name: PGID
    value: "1000"
  - name: PUID
    value: "1000"
  - name: TZ
    value: "Asia/Almaty" # change to your timezone

  volumeMounts:
  - mountPath: "/config" # path to config 
    name: jellyfin-config
  - mountPath: "/cache" # path to cache
    name: jellyfin-cache
  - mountPath: "/media" # path to media
    name: media
  
  resources: {}
```

Also volumes need to be added under spec
```yaml
spec:
...
  volumes:
	- name: jellyfin-config
	  hostPath:
		path: /storage/jellyfin/config # path on the server
	- name: jellyfin-cache
	  hostPath:
		path: /storage/jellyfin/cache # path on the server
	- name: media
	  hostPath:
		path: /storage/media # path on the server
```
For this example, I am using hostPath, which utilizes the host node’s filesystem for storing data and accessing media. If you need to specify the exact path to the media files, don’t forget to include it in the media volume path.

If you have more than one node and your media files are attached to a specific machine, you can add affinity properties to the pod spec:
```yaml
spec:
  ...
  affinity:
	nodeAffinity:
	  requiredDuringSchedulingIgnoredDuringExecution:
		nodeSelectorTerms:
		  - matchExpressions:
			- key: kubernetes.io/hostname
			  operator: In
			  values:
				- <node name> # specify the node name where you want pod to run
  ...
```

So, the complete deployment manifest will look like this:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: jellyfin
  name: jellyfin
  namespace: jellyfin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jellyfin
  strategy: {}
  template:
    metadata:
      labels:
        app: jellyfin
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
            - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
              - <node name> # specify the node name where you want pod to run
      containers:
      - image: jellyfin/jellyfin:10.10.3 # version can be changed 
        name: jellyfin
        resources: {}
        ports:
        - containerPort: 8096 # port for accessing webui
          name: http-tcp
          protocol: TCP

        env:
        - name: JELLYFIN_PublishedServerUrl
          value: "http://<domain>:80" # change to your domain
        - name: PGID
          value: "1000"
        - name: PUID
          value: "1000"
        - name: TZ
          value: "Asia/Almaty" # change to your timezone

        volumeMounts:
        - mountPath: "/config" # path to config 
          name: jellyfin-config
        - mountPath: "/cache" # path to cache
          name: jellyfin-cache
        - mountPath: "/media" # path to media
          name: media
      volumes:
      - name: jellyfin-config
        hostPath:
          path: /storage/jellyfin/config # path on the server
      - name: jellyfin-cache
        hostPath:
          path: /storage/jellyfin/cache # path on the server
      - name: media
        hostPath:
          path: /storage/media # path on the server
```

## Service
For making the app available in the cluster we need to expose the deployment using ClusterIP service. Core manifest can be create using this command:
```bash
$ kubectl k -n jellyfin expose deployment jellyfin \
	--name jellyfin-svc \
	--target-port 8096 --port 8096 \
	--dry-run=client -o yaml > service.yaml
```

It will create the following manifest:
```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: jellyfin
  name: jellyfin-svc
  namespace: jellyfin
spec:
  ports:
  - port: 8096
    protocol: TCP
    targetPort: 8096
  selector:
    app.kubernetes.io/name: jellyfin
status:
  loadBalancer: {}
```

## Ingress
The last part is Ingress that can be created using the following command
```bash
 $ kubectl create ingress jellyfin-ingress --class=nginx \
	  --rule="<your domain>/*=jellyfin-svc:80" \
	  --dry-run=client -o yaml > ingress.yaml# do not forget to specify the domain
```

As a result we got the following manifest:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jellyfin-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: <your domain>
    http:
      paths:
      - backend:
          service:
            name: jellyfin-svc
            port:
              number: 80
        path: /
        pathType: Prefix
status:
  loadBalancer: {}
```
## Deploying the resources
Finally, we have two options: either run the manifests one by one or save all the manifests in a single file, separated by three dashes (---).

If you choose to run the manifests individually, follow the same order in which the manifests were created.

## Conclusion
By following these steps, you can successfully convert a Docker Compose file into Kubernetes objects for Jellyfin. This allows you to leverage Kubernetes' powerful orchestration capabilities for managing your media server.