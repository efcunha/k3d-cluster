# Ingress Controller

## Disable traefik in K3D

`cluster-cfg.yml`
```yaml
...
  k3s:
    extraServerArgs:
      - --tls-san=127.0.0.1
      - --no-deploy=traefik
...
```
## Deploy Ingress

### Deploy with Kubectl
```sh 
$> kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/cloud/deploy.yaml
```

### Deploy with Helm


Install helm repo
```sh
$> helm repo add bitnami https://charts.bitnami.com/bitnami
$> helm repo update
$> kubectl create namespace ingress
```

Crie um namespace para entrada e crie um segredo com o certificado de servidor padrão
Eu uso [mkcert](https://github.com/FiloSottile/mkcert) para gerar esses certificados assinados e instalar a raiz da CA no meu computador.
Neste caso, o certificado corresponde aos domínios `*.fuf.me`.
Consulte o documento mkcert para obter mais informações.

```sh
$> kubectl --namespace ingress create secret tls nginx-server-certs --key fuf.me-key.pem --cert fuf.me.pem
```

Crie um arquivo para substituir os valores padrão no leme

`ingress-values.yml`
```yaml
extraArgs:
  default-ssl-certificate: "ingress/nginx-server-certs"
```

```sh
$> helm install --namespace ingress -f ingress-values.yaml ingress bitnami/nginx-ingress-controller 
```

## Test Ingress

`nginx-ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  rules:
  - host: "nginx.fuf.me"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: nginx
            port:
              number: 80
```

```sh
$> kubectl create deployment nginx --image=nginx
$> kubectl create service clusterip nginx --tcp=80:80
$> kubectl apply -f  nginx-ingress.yaml
```

### Custom domain certificates

Se definirmos a configuração de TLS no ingresso para outros domínios, você precisará criar um segredo com valores de certificado no namespace onde você vai implantar o ingresso. 

```sh
$> kubectl create secret tls example-certs --key example.com.key --cert example.com.pem
```

e use-o no arquivo yaml de entrada

```yml
piVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  annotations:
    ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
      - "nginx.example.com" 
    - secretName: example-certs    
  rules:
    - host: "nginx.example.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
```

### Generating certificates on the fly
```sh
$> openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=fuf.me' -keyout ca.key -out ca.crt
$> openssl req -out nginx.fuf.me.csr -newkey rsa:2048 -nodes -keyout nginx.fuf.me.key -subj "/CN=nginx.fuf.me/O=some organization"
$> openssl x509 -req -days 365 -CA ca.crt -CAkey ca.key -set_serial 0 -in nginx.fuf.me.csr -out nginx.fuf.me.crt

$> nginx.fuf.me.key > nginx.fuf.me.pem
$> cat nginx.fuf.me.crt >> nginx.fuf.me.pem
$> cat nginx.fuf.me.csr >> nginx.fuf.me.pem
$> kubectl create secret tls nginx-server-certs --key nginx.fuf.me.key --cert nginx.fuf.me.pem
```


## Deploy Cert Manager with Ingress

### Install Cert Manager

```sh
# Instale os recursos CustomResourceDefinition separadamente
$> kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.8/deploy/manifests/00-crds.yaml

# Crie o namespace para cert-manager
$> kubectl create namespace cert-manager

# Rotule o namespace cert-manager para desabilitar a validação de recursos
$> kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true

# Adicione o repositório Jetstack Helm
$> helm repo add jetstack https://charts.jetstack.io

# Atualize o cache do repositório do gráfico Helm local
$> helm repo update

# Instale o gráfico Helm do cert-manager
$> helm install \
  --namespace cert-manager --create-namespace \
  cert-manager jetstack/cert-manager
```

### Create Issuers

#### let`s encrypt issuers

`lets-encrypt-dev-issuer.yml`
```yml
cat <<EOF | kubectl create -f -
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-dev
  namespace: cert-manager
spec:
  acme:
    # The ACME server URL
    server: hhttps://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: user@example.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx
EOF          
```

`lets-encrypt-prod-issuer.yml`
```yml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: user@example.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx
```

#### CA Issuers

# Generate a CA private key
$ openssl genrsa -out ca.key 2048

# Create a self signed Certificate, valid for 10yrs with the 'signing' option set
$ openssl req -x509 -new -nodes -key ca.key -subj "/CN=${COMMON_NAME}" -days 3650 -reqexts v3_req -extensions v3_ca -out ca.crt

kubectl create secret tls ca-key-pair \
   --cert=ca.crt \
   --key=ca.key \
   --namespace=default

```yml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: ca-issuer
  namespace: default
spec:
  ca:
    secretName: ca-key-pair
```   

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: ca-issuer
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    kind: Issuer
  commonName: example.com
  organization:
  - Example CA
  dnsNames:
  - example.com
  - www.example.com
```

este arquivo gera um segredo no namespace com example-com-tls
este segredo é o segredo que será usado no arquivo de ingress


```yml
piVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  annotations:
    ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
      - "nginx.example.com" 
    - secretName: example-com-tls    
  rules:
    - host: "nginx.example.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
```

https://docs.cert-manager.io/en/release-0.8/index.html

https://cloud.google.com/kubernetes-engine/docs/how-to/ingress-multi-ssl

cat <<EOF | helm install --namespace ingress -f - ingress bitnami/nginx-ingress-controller
extraArgs:
  default-ssl-certificate: "ingress/nginx-server-certs"
EOF

cat <<EOF > tmp-ingress-${CLUSTER_NAME}-values.yaml
extraArgs:
  default-ssl-certificate: "ingress/nginx-server-certs"
EOF

cat <<EOF > tmp-ingress-values.yaml
extraArgs:
  default-ssl-certificate: "ingress/nginx-server-certs
EOF

kubectl apply  \
--namespace=kubernetes-dashboard \
--enable-insecure-login \
--insecure-bind-address=0.0.0.0 \
-f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml

--name my-release

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubeapps
  namespace: kubeapps
  annotations:
    ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
    - host: kubeapps.${CLUSTER_DOMAIN}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kubeapps-internal-dashboard
                port:
                  number: 8080
EOF