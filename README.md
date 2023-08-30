# orchestrate-with-kubernetes
```bash
$ ls
cleanup.sh  deployments  nginx  pods  services  tls


cat cleanup.sh 
kubectl delete pods healthy-monolith monolith secure-monolith
kubectl delete services monolith auth frontend hello
kubectl delete deployments auth frontend hello hello-canary hello-green
kubectl delete secrets tls-certs
kubectl delete configmaps nginx-frontend-conf nginx-proxy-conf


deployments
ls
auth.yaml  frontend.yaml  hello-canary.yaml  hello-green.yaml  hello.yaml

$ cat auth.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth
spec:
  selector:
    matchLabels:
      app: auth
  replicas: 1
  template:
    metadata:
      labels:
        app: auth
        track: stable
    spec:
      containers:
        - name: auth
          image: "kelseyhightower/auth:2.0.0"
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
          resources:
            limits:
              cpu: 0.2
              memory: "10Mi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 15
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /readiness
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            timeoutSeconds: 1

$ cat frontend.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  selector:
      matchLabels:
        app: frontend
  replicas: 1
  template:
    metadata:
      labels:
        app: frontend
        track: stable
    spec:
      containers:
        - name: nginx
          image: "nginx:1.9.14"
          lifecycle:
            preStop:
              exec:
                command: ["/usr/sbin/nginx","-s","quit"]
          volumeMounts:
            - name: "nginx-frontend-conf"
              mountPath: "/etc/nginx/conf.d"
            - name: "tls-certs"
              mountPath: "/etc/tls"
      volumes:
        - name: "tls-certs"
          secret:
            secretName: "tls-certs"
        - name: "nginx-frontend-conf"
          configMap:
            name: "nginx-frontend-conf"
            items:
              - key: "frontend.conf"
                path: "frontend.conf"

$ cat hello-canary.yaml 
apiVersion: extensions/v1
kind: Deployment
metadata:
  name: hello-canary
spec:
  replicas: 1
    selector:
    matchLabels:
      app: hello-canary
  template:
    metadata:
      labels:
        app: hello
        track: canary
        version: 2.0.0
    spec:
      containers:
        - name: hello
          image: kelseyhightower/hello:2.0.0
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
          resources:
            limits:
              cpu: 0.2
              memory: 10Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 15
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /readiness
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            timeoutSeconds: 1

$ cat hello-green.yaml 
apiVersion: extensions/v1
kind: Deployment
metadata:
  name: hello-green
spec:
  replicas: 3
    selector:
    matchLabels:
      app: hello-green
  template:
    metadata:
      labels:
        app: hello
        track: stable
        version: 2.0.0
    spec:
      containers:
        - name: hello
          image: kelseyhightower/hello:2.0.0
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
          resources:
            limits:
              cpu: 0.2
              memory: 10Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 15
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /readiness
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            timeoutSeconds: 1

$ cat hello.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  selector:
      matchLabels:
        app: hello
  replicas: 3
  template:
    metadata:
      labels:
        app: hello
        track: stable
        version: 1.0.0
    spec:
      containers:
        - name: hello
          image: "kelseyhightower/hello:1.0.0"
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
          resources:
            limits:
              cpu: 0.2
              memory: "10Mi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 15
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /readiness
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            timeoutSeconds: 1

cd nginx/
ls
frontend.conf  proxy.conf

$ cat frontend.conf
upstream hello {
    server hello.default.svc.cluster.local;
}

upstream auth {
    server auth.default.svc.cluster.local;
}

server {
    listen 443;
    ssl    on;

    ssl_certificate     /etc/tls/cert.pem;
    ssl_certificate_key /etc/tls/key.pem;

    location / {
        proxy_pass http://hello;
    }

    location /login {
        proxy_pass http://auth;


$ cat proxy.conf 
server {
  listen 443;
  ssl    on;

  ssl_certificate     /etc/tls/cert.pem;
  ssl_certificate_key /etc/tls/key.pem;

  location / {
    proxy_pass http://127.0.0.1:80;
  }
}

cd pods/
ls
healthy-monolith.yaml  monolith.yaml  secure-monolith.yaml


cat healthy-monolith.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: "healthy-monolith"
  labels:
    app: monolith
spec:
  containers:
    - name: monolith
      image: kelseyhightower/monolith:1.0.0
      ports:
        - name: http
          containerPort: 80
        - name: health
          containerPort: 81
      resources:
        limits:
          cpu: 0.2
          memory: "10Mi"
      livenessProbe:
        httpGet:
          path: /healthz
          port: 81
          scheme: HTTP
        initialDelaySeconds: 5
        periodSeconds: 15
        timeoutSeconds: 5
      readinessProbe:
        httpGet:
          path: /readiness
          port: 81
          scheme: HTTP
        initialDelaySeconds: 5
        timeoutSeconds: 1

cat monolith.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: monolith
  labels:
    app: monolith
spec:
  containers:
    - name: monolith
      image: kelseyhightower/monolith:1.0.0
      args:
        - "-http=0.0.0.0:80"
        - "-health=0.0.0.0:81"
        - "-secret=secret"
      ports:
        - name: http
          containerPort: 80
        - name: health
          containerPort: 81
      resources:
        limits:
          cpu: 0.2
          memory: "10Mi"

cat secure-monolith.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: "secure-monolith"
  labels:
    app: monolith
spec:
  containers:
    - name: nginx
      image: "nginx:1.9.14"
      lifecycle:
        preStop:
          exec:
            command: ["/usr/sbin/nginx","-s","quit"]
      volumeMounts:
        - name: "nginx-proxy-conf"
          mountPath: "/etc/nginx/conf.d"
        - name: "tls-certs"
          mountPath: "/etc/tls"
    - name: monolith
      image: "kelseyhightower/monolith:1.0.0"
      ports:
        - name: http
          containerPort: 80
        - name: health
          containerPort: 81
      resources:
        limits:
          cpu: 0.2
          memory: "10Mi"
      livenessProbe:
        httpGet:
          path: /healthz
          port: 81
          scheme: HTTP
        initialDelaySeconds: 5
        periodSeconds: 15
        timeoutSeconds: 5
      readinessProbe:
        httpGet:
          path: /readiness
          port: 81
          scheme: HTTP
        initialDelaySeconds: 5
        timeoutSeconds: 1
  volumes:
    - name: "tls-certs"
      secret:
        secretName: "tls-certs"
    - name: "nginx-proxy-conf"
      configMap:
        name: "nginx-proxy-conf"
        items:
          - key: "proxy.conf"
            path: "proxy.conf"

cd services/
ls
auth.yaml  frontend.yaml  hello-blue.yaml  hello-green.yaml  hello.yaml  monolith.yaml

$ cat auth.yaml 
kind: Service
apiVersion: v1
metadata:
  name: "auth"
spec:
  selector:
    app: "auth"
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 80

$ cat frontend.yaml 
kind: Service
apiVersion: v1
metadata:
  name: "frontend"
spec:
  selector:
    app: "frontend"
  ports:
    - protocol: "TCP"
      port: 443
      targetPort: 443
  type: LoadBalancer


cat frontend.yaml 
kind: Service
apiVersion: v1
metadata:
  name: "frontend"
spec:
  selector:
    app: "frontend"
  ports:
    - protocol: "TCP"
      port: 443
      targetPort: 443
  type: LoadBalancer


$ cat hello-blue.yaml 
kind: Service
apiVersion: v1
metadata:
  name: "hello"
spec:
  selector:
    app: "hello"
    version: 1.0.0
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 80


$ cat hello-green.yaml 
kind: Service
apiVersion: v1
metadata:
  name: hello
spec:
  selector:
    app: hello
    version: 2.0.0
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

cat hello.yaml 
kind: Service
apiVersion: v1
metadata:
  name: "hello"
spec:
  selector:
    app: "hello"
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 80


cat monolith.yaml 
kind: Service
apiVersion: v1
metadata:
  name: "monolith"
spec:
  selector:
    app: "monolith"
    secure: "enabled"
  ports:
    - protocol: "TCP"
      port: 443
      targetPort: 443
      nodePort: 31000
  type: NodePort


cd tls
ls
ca-key.pem  ca.pem  cert.pem  key.pem

cat ca-key.pem 
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA6krl+/g69Et06/cU6601EjypCM5vhB1ghXdhYuqCNZZLWPMB
/B55GkNnHMUPfMCMs6W6ItHVjNquaUabCLbnnXKmdEIWfq8NWKnNy8vUX2QIdKfq
Tigc4pijrcuJekbFNdFtpnCx6ifTdPZMqnsGP2dY32rdVQdZ0DuPp3aJwJNTBAwA
g/naoHLvyWLrZB5FHkxN61iBl78jllLXKv9qdRRg+h3GWA95ObpQlKfo8gSyfqFS
ZlCvVeFhaNbcxd4ZiXLi6H1laWjybcKfUd6cmCw1U4AVnEnmXZGIsW9zv6/qmj7J
0MNEz65ivr+QDZPzZv63RzijJs2pYHl9oUq/7wIDAQABAoIBAQC2lZHvL/65nQhM
T6x1EfF2+eD9JOuQ+NfcizFQxdKdcjfb5N0aHqFfz0FPEV9FaET+R1vsgLw8Xbtn
/YcaXnfXop6HoW0oYsEy5HmlpX4mrK1ORAF70RTZnfyIl0LXEMnlbAVYnSB5i3nl
/3+1p9QxmxeOXRiJiAX9Gj2UUvN9J5SXgPGfH+94t8l7qY/kvDvJQDWQ6DLCPBca
tUjCEmaJVsbE5ekKh6ENNZxTgIUp7BON3BnUuM9ud3Gmyf3dOlZB6rz5mooNH/41
G5tUqau5ppixo9VgCfsPjy1eKzHInzYppw44Lq2tVPZ+A429qB6tKkOIotcud3zY
/VYufh9JAoGBAP7sRqCxYb/QQG4Eniqc4SP5Ec5RXYbm5i7Sk8HGbahsnguqQZJf
PwuoamMEhP7SJI2/FwJm6y/lRN7Dqo1fuTpyYng0vBsBOP7etVjIiHMoGoP+jKWV
uivICkQRuhiu9LBwaCE2nsC4LsOXFmcNJRkXtO3ygK2HjZTeXWYpnkelAoGBAOtI
Twe2t5wiwsODs7KtQ4NAWIYaHpG34Cwc6SAh+S+7QRDJ4msyW8pwVQSW7xuTML23
zDzQ4gVbDbNUbfmKOPa/XJI9ECSZkc/+vWUrzfeT1hK2UyeqsmEZdFYitJ5/oF8l
VhSPB9Sd0YknzQfF8ZICSuQEg0UJmxRKUgeUM/UDAoGBAORWItUgzVuQX4WsITgu
GQOtvyM8gjepbpiWCb9RyztHPzFXqTBAnCoHCnPywmW1OQS2GxgNs6/M/qlCPewv
x6vwdP8SzUKrD7BLL8h8pqvvSgDc6oIO4RkCLx/VeQlO/OFlbgAB+qTI1SpglLJt
dcNKFsfjpRrKBilIHAS8Vof5AoGAJvWeQIS8+pm27nEMfHW8TCuHfQ0uKqrr7+IJ
qEx32rODHqiPWXjJQkg/i7cCeOpyk7evlhJwmrptFljQrRV6QUGGrqB139meD3b7
HZmXTXupYwfV1SeqyfFRFkJA7k3r3FVuX5EfltFbNP7mMHdSfP7sL72fjvr8Nuvn
kWG1CMkCgYANW0BiYWmxcw4LXjYVrrcgkrrJevL6nQ4ELxeb5+ic60hoTFo7X37D
kvIoCc1OWLyJ/bT12ZgW45SqxZgC+JNQ2XhvorPE6Bq33h1tZzULHEKyde2MyqLW
ze1zfkHPseS5xbXwMir7JfYPB7n2Tt9j+vynyQ8nFsGvRSJgYiJvhg==
-----END RSA PRIVATE KEY-----

cat ca.pem 
-----BEGIN CERTIFICATE-----
MIIDOzCCAiOgAwIBAgIQby02MxEEyeNyWhqHw4a3zDANBgkqhkiG9w0BAQsFADAV
MRMwEQYDVQQKEwpLdWJlcm5ldGVzMB4XDTE2MDQyNjA0MDkwOFoXDTE3MDQyNjA0
MDkwOFowFTETMBEGA1UEChMKS3ViZXJuZXRlczCCASIwDQYJKoZIhvcNAQEBBQAD
ggEPADCCAQoCggEBAOpK5fv4OvRLdOv3FOutNRI8qQjOb4QdYIV3YWLqgjWWS1jz
AfweeRpDZxzFD3zAjLOluiLR1YzarmlGmwi2551ypnRCFn6vDVipzcvL1F9kCHSn
6k4oHOKYo63LiXpGxTXRbaZwseon03T2TKp7Bj9nWN9q3VUHWdA7j6d2icCTUwQM
AIP52qBy78li62QeRR5MTetYgZe/I5ZS1yr/anUUYPodxlgPeTm6UJSn6PIEsn6h
UmZQr1XhYWjW3MXeGYly4uh9ZWlo8m3Cn1HenJgsNVOAFZxJ5l2RiLFvc7+v6po+
ydDDRM+uYr6/kA2T82b+t0c4oybNqWB5faFKv+8CAwEAAaOBhjCBgzAOBgNVHQ8B
Af8EBAMCAqQwEwYDVR0lBAwwCgYIKwYBBQUHAwEwDwYDVR0TAQH/BAUwAwEB/zAd
BgNVHQ4EFgQUDxJum8kqt4i67A/qALy2h/ON4iwwHwYDVR0jBBgwFoAUDxJum8kq
t4i67A/qALy2h/ON4iwwCwYDVR0RBAQwAoIAMA0GCSqGSIb3DQEBCwUAA4IBAQC8
VaLgNIFw66LOQoU5QAoqBJ8i8/L5i0b9cJZoM/YPEGd44OK2wMQEh1vaVsXYhDe2
6MZcomDzcq4XD7/OsBtdrFEw7F7prwWbVvjlgLqhOdlAtNQyTMFd/Jm3M6pI7NTf
GiOdLdyMyy/sY719sh34IpetTE9W/K0wKSnNJG011qV+LSOAR0g7H3OZyJN+MqVp
WurY/umW7PJF0Xo5YR4jSWf/rdaqFLrTAt4Xn6jIBZeW41E5ABxhvaUA7EcSIBYP
MFF8YS+e/Clg2FeM6EikABTXFBWGv23UlCRALaduYGwYH+M86bNgO45Ba1JJkP6M
4m0nKCrFlqDnSUyaCsZs
-----END CERTIFICATE-----

cat cert.pem 
-----BEGIN CERTIFICATE-----
MIIDbjCCAlagAwIBAgIQWqMGAhd/4EOo4hFRrvXhrDANBgkqhkiG9w0BAQsFADAV
MRMwEQYDVQQKEwpLdWJlcm5ldGVzMB4XDTE2MDQyNjA0MDkwOFoXDTE3MDQyNjA0
MDkwOFowLTETMBEGA1UEChMKS3ViZXJuZXRlczEWMBQGA1UEAxMNKi5leGFtcGxl
LmNvbTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMpXqpuNLl6q9eD0
fifuAGjnI0TlvkSdETUwMkA0HMXeGl3a7tfhgkd4gxdOJPQpmwTz1l84ckzi1wPo
MrDDvhAwyAVzvg6rRoZ/jccZQ2DHfaaUkBc3fabQw38stLKbyMenCGm3GmhGcAOW
GrJ2yADwqfELTY/M4MSXSFhvxsOPpNCv6v2kCLwMcHMwHMkgdBlZKwDr1KPeHUd7
YEo8T4CYqNrYpckqwH/kuecDSyau2iowkce17LfUDpM1GKeENAiN6Mz5e8yPK1a8
EUEVLFWzZMdxcdIPZ8ZPDkTgn1qH8134FQOzHLtxHBDPRkXa9vs9iieBGBYJBtO6
gTe8wLECAwEAAaOBoTCBnjAOBgNVHQ8BAf8EBAMCBaAwEwYDVR0lBAwwCgYIKwYB
BQUHAwEwDAYDVR0TAQH/BAIwADAdBgNVHQ4EFgQUNC2nZTkEz0njbyYs1jkPUy9k
O7wwHwYDVR0jBBgwFoAUDxJum8kqt4i67A/qALy2h/ON4iwwKQYDVR0RBCIwIIIN
Ki5leGFtcGxlLmNvbYIJbG9jYWxob3N0hwR/AAABMA0GCSqGSIb3DQEBCwUAA4IB
AQC3/YnlZaq0CVWSfQBnaO4dPc05aN36GgDhRjxJZ8pAoklhJA28BlvyhAwJPJZv
AtxquUmZPYXBbhqtOaMosaLKYWvvaN5IzV+rfQ8sWUN/1JdgeEPku4U19aiDtj5z
VWqS9C3cU6MTAUP0pvn+vq1moOnMEyGiPA0wEngvO/Bjhg9MFwHByY2ISEgutuD9
9Iv7sxkx3M4VoQxRfGtHLYiVwqiS1YcbwaGfrRyBThBKoGbUUa8nViFAxr9KPZO+
0hxuqdBDfk942lWFfqN8O8GHp5O0KrZTej9vhgBhgx1XkbL5MeKAV6oHLgHl+zj5
h2j/WHCfgs5+or21+Q9M9+WK
-----END CERTIFICATE-----

cat key.pem 
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAyleqm40uXqr14PR+J+4AaOcjROW+RJ0RNTAyQDQcxd4aXdru
1+GCR3iDF04k9CmbBPPWXzhyTOLXA+gysMO+EDDIBXO+DqtGhn+NxxlDYMd9ppSQ
Fzd9ptDDfyy0spvIx6cIabcaaEZwA5YasnbIAPCp8QtNj8zgxJdIWG/Gw4+k0K/q
/aQIvAxwczAcySB0GVkrAOvUo94dR3tgSjxPgJio2tilySrAf+S55wNLJq7aKjCR
x7Xst9QOkzUYp4Q0CI3ozPl7zI8rVrwRQRUsVbNkx3Fx0g9nxk8OROCfWofzXfgV
A7Mcu3EcEM9GRdr2+z2KJ4EYFgkG07qBN7zAsQIDAQABAoIBACarnHp//+WtzLIC
Z/3fmYpy6iWntrZMQlak8GWe0ATszqMzTURK3+gi2wLgN2XGcc7/fu/RzN5u1+Ly
RIXN0wwrFn8cQK1zBFZ+GC194YekeJoWeHdHbqcr7MDoXVxpM3UcshnqGYzmMVAu
JsoGs3Cijgf4PgmGgUpxEy17p0QGX6AmeqhSzeBwc6pNfJvPeXljJKBfMXTnkxn8
eFpCaZVVMd3MDuAFRajgEoyO428MAGYpjIScuV4Vutd7QDa39hax5JMpc2yNUXPQ
eIHLrj21aHX3bgfh1WE3yhclYf3pGarpOuW27N4UrmeGyb7jAa80CMWsqAILRru7
vEHh50ECgYEA00hRBJuF04MLkHsk5/oX5TllIA/zEsrYgoHYI3agWUqLZMUR/ujE
tNDjdCf5RG8lf9ANcU7eFhQMCZdDiSH2dwwmv6J2/RzUsPJ+675vktl3AoTVD7KA
glsuPSChik8k/1PI7ZaYhTs2UxCOVB7hOXsv9qjSoIHgFMZDnkswTnkCgYEA9Sr2
NvVU4OLI1CPCjAltIhnDUMqsJ6crlJFICAR2qeBQcf3SqAUAh/NRZ9NodIRozgOE
bHDjKYD98Hq5v2QHqVnZ3BGNlNJrPEt33I7DcJ6Z6OfNVcWMDL4Gm5JsjnJl2AR7
2HNXaB91qQzmzcb6YfL3r7TwSZ4GtuwJb2O0lfkCgYBLBCEn9qQ0bhHcEa0P5F85
lwBNuvv+DyGCbOG17bePHIWTmNkD3deBr60in9LENoZk9BThxzPZOPLxMNDczr84
k4rqfZ+rzOHDlcX0o9/vjuDPdyRC94jjP8aSE5Tni6RCN5heqxqqK1Tldzphqbkj
9JYaCOUH8jUCi0aU3HNhWQKBgFAQGZvU/kT6io8MponIwkTymOAXb6T7aLX5w8Yq
fv327Q5sz5BjIctD4H/BgEkcvIUajPJE40o4f7U6vtILvpzFZOoDKXNCTBbCpn/2
d0id4rE2kc3C13uJyuqfJKhYH34t6KvE7vRn4aq1NeJZaob2K4DL2/SOkK7H4kTo
EJ8xAoGBANGyX2sECsld/+KOWH5xMrAZo2qNkL+TgRQ3o8NHcUcvAT3C23tYstoN
7LpKSmZxba7IWnV7bYUPXqHVLpNIZAuuMFHOxTnU7hxrp9oVpJVJt+MABR2/1Kay
O0JSjXP6b58VllICqOMjyR/HbtoBwg9qe9LfxFdZXvJOEWg5ErqC
-----END RSA PRIVATE KEY-----
```
