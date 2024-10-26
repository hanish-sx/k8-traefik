# k8-traefik
kubernetes traefik routing and middleware

## Reading
https://doc.traefik.io/traefik/getting-started/quick-start-with-kubernetes/


## 02-traefik-service.yaml
```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: traefik-deployment
  namespace: traefik
  labels:
    app: traefik

spec:
  replicas: 1
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-account
      containers:
        - name: traefik
          image: traefik:v3.1
          args:
            - --api.insecure
            # - --providers.kubernetesingress
            # --entryPoints.web.Address=:8000
            - --providers.kubernetescrd           # <----- required for recognizing our middleware ---->
          ports:
            - name: web
              containerPort: 80
            - name: dashboard
              containerPort: 8080

```

## 02-traefik.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: traefik-dashboard-service
  namespace: traefik
spec:
  type: LoadBalancer
  ports:
    - port: 8080
      targetPort: dashboard
  selector:
    app: traefik
---
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-service
  namespace: traefik
spec:
  type: LoadBalancer
  ports:
    - targetPort: web
      port: 8088       # <-------------- this is out apps service port, not the apps container port --->
  selector:
    app: traefik

```

## 03-whoami-services.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: traefik 
spec:
  ports:
    - name: web
      port: 8088        # <-------------------- port we are planning to expose ------->
      #port: 80
      targetPort: web

  selector:
    app: whoami

```

## 04-whoami-ingress.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami-ingress
  namespace: traefik
spec:
  rules:
  - http:
  # - host: traefik.example.com      # <----------- if we set host, we have to use curl -H 'HOST: traefik.example.com' ip:80/path to get to our app --------->
  #  http:
      paths:
      - path: /who
        pathType: Prefix
        backend:
          service:
            name: whoami
            port:
                name: web
```

## Test
```
$ curl -iv localhost/who
*   Trying 127.0.0.1:80...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 80 (#0)
> GET /who HTTP/1.1
> Host: localhost
> User-Agent: curl/7.68.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 401 Unauthorized
HTTP/1.1 401 Unauthorized
< Content-Type: text/plain
Content-Type: text/plain
< Www-Authenticate: Basic realm="traefik"
Www-Authenticate: Basic realm="traefik"
< Date: Sat, 26 Oct 2024 00:23:23 GMT
Date: Sat, 26 Oct 2024 00:23:23 GMT
< Content-Length: 17
Content-Length: 17

< 
401 Unauthorized
* Connection #0 to host localhost left intact
```

```
$ curl  -u user:password localhost/who
Hostname: whoami-57b48994d9-j5p4s
IP: 127.0.0.1
IP: ::1
IP: 10.42.0.49
IP: fe80::8c6c:f4ff:fee0:f073
RemoteAddr: 10.42.0.8:42578
GET /who HTTP/1.1
Host: localhost
User-Agent: curl/7.68.0
Accept: */*
Accept-Encoding: gzip
Authorization: Basic dXNlcjpwYXNzd29yZA==
X-Forwarded-For: 10.42.0.7
X-Forwarded-Host: localhost
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: traefik-67f6c94c47-5ldpz
X-Real-Ip: 10.42.0.7
```
