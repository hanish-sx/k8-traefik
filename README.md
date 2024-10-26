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

