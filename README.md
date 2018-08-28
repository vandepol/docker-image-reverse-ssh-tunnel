# docker-reverse-ssh-tunnel
This container setups up a reserve ssh tunnel between a NATed host and a public host. It allows the application server in the NATed host can be accessed through the public host, which helps the application poke a hole through its firewall.

Create Image
------------
To create the image `exposemyport-tunnel`, execute the following command:
```
	docker build . -t exposemyport-tunnel
```
Usage MANUAL (HELM under development)
-----
To deploy to Kubernetes use the following deployment.yaml:

```YAML
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: remoteconnect
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: remoteconnect
    spec:
      containers:
      - name: remoteconnect-app
        image: registry.ng.bluemix.net/dvp_node01/reverse-ssh-tunnel
        env:
        - name: ROOT_PASS
          value: password *** CHANGE TO UNIQUE PASSWORD ***
        ports:
        - containerPort: 22
          name: sshd-port
        - containerPort: 1080
          name: forwarding-port
```
Use for following service.yaml:
```YAML
apiVersion: v1
kind: Service
metadata:
  name: remoteconnect
  labels:
    apps: remoteconnect
spec:
  type: NodePort
  ports:
  - name: sshd
    port: 22
    targetPort: sshd-port
    protocol: TCP
  - name: forwarding
    port: 5432  *** CHANGE TO PORT YOU WANT TO FORWARD ***
    targetPort: forwarding-port
    protocol: TCP
  selector:
    app: remoteconnect
```

Determine the service ports used for the application:
```
$ kubectl get services
NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                       AGE
remoteconnect                      NodePort    172.21.142.22    <none>        22:31445/TCP,5432:30335/TCP   47m
```
One the machine with the port you want to expose run the following command

docker run  -e PUBLIC_HOST_ADDR=exposemyport.com -e PUBLIC_HOST_PORT=<PORT MAPPED TO 22> -e ROOT_PASS=<Password provided in deployment yaml>  -e PROXY_PORT=<Port to expose on local machine> -e PRIVATE_HOST_TUNNEL_PORT=1080 -p1080:1080 --name exposeport reverse-ssh-tunnel

<you can change PRIVATE_HOST_TUNNEL_PORT if you need to run more than one at a time.


example for postgres:
`docker run  -e PUBLIC_HOST_ADDR=184.172.236.212 -e PUBLIC_HOST_PORT=31445 -e ROOT_PASS=password -e PROXY_PORT=5432 -e PRIVATE_HOST_TUNNEL_PORT=1080 -p1080:1080 --name exposepostgres reverse-ssh-tunnel`

<!-- On NATed Host, run:

```
  docker run -d \
    -e PUBLIC_HOST_ADDR=<public_host_address> \
    -e PUBLIC_HOST_PORT=<public_host_port> \
    -e ROOT_PASS=<your_password> \
    -e PROXY_PORT=<NATed_service_port> \
    --net=host \
    tifayuki/reverse-ssh-tunnel
```
Parameters:
```
  <public_host_address> is the ip address or domain of your public host
  <public_host_port> is the same as <your sshd port> set on the public host
  <your_passorwd> is the same as <your passorwd> set on the public host
  <NATed_service_port> is the port that your service in NATed host listens to
```

Then, a user can access <public_host_address>:<forwording_port> to communicate the service running in the NATed host listened to port <PROXY_PORT>

Example
-------

Suppose you have a service running in the NATed host, listens to port `8080`, and there is also another public host with the ip address of `111.112.113.114`. You want users to access `111.112.113.114:80` to communicate with your NATed service, you could the do the following:

On public host(`111.112.113.114`):
```
  docker run -d -e ROOT_PASS=mypass -p 2222:22 -p 80:1080 tifayuki/reverse-ssh-tunnel
```
On NATed host:
```
  docker run -d -e PUBLIC_HOST_ADDR=111.112.113.114 -e PUBLIC_HOST_PORT=2222 -e ROOT_PASS=mypass -e PROXY_PORT=8080 --net=host tifayuki/reverse-ssh-tunnel
```

Then, `curl 111.112.113.114:80` will work -->
