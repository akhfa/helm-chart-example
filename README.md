# helm-chart-example
## Dependencies
Kubernetes cluster yang sudah aktif. Untuk environment local, dapat menggunakan minikube
1. Download minikube dari https://github.com/kubernetes/minikube/releases
2. Install virtualbox untuk virtualisasi VM kubernetes
3. Jalankan minikube. Untuk --memory dapat diset sesuai dengan memory yang dimiliki, untuk kenyamanan, beri minimal 2048
   ```
   minikube start --vm-driver=virtualbox --memory=4096
   ```

## Installing helm
1. Download helm dari `https://github.com/helm/helm/releases` sesuai dengan sistem operasi dan masukkan helm binary ke PATH environment masing-masing.
   ```
   wget https://storage.googleapis.com/kubernetes-helm/helm-v2.11.0-linux-amd64.tar.gz

   ```
2. Apply helm service account
   ```
   kubectl apply -f https://raw.githubusercontent.com/akhfa/helm-chart-example/master/install/helm-service-account.yaml
   ```
3. Set up TLS for helm
   ```
   mkdir helm-tls
   cd helm-tls
   openssl genrsa -out ./ca.key.pem 4096
   openssl req -key ca.key.pem -new -x509 -days 7300 -sha256 -out ca.cert.pem -extensions v3_ca
   openssl genrsa -out ./tiller.key.pem 4096
   openssl genrsa -out ./helm.key.pem 4096
   openssl req -key tiller.key.pem -new -sha256 -out tiller.csr.pem
   openssl req -key helm.key.pem -new -sha256 -out helm.csr.pem
   openssl x509 -req -CA ca.cert.pem -CAkey ca.key.pem -CAcreateserial -in tiller.csr.pem -out tiller.cert.pem -days 3650
   openssl x509 -req -CA ca.cert.pem -CAkey ca.key.pem -CAcreateserial -in helm.csr.pem -out helm.cert.pem  -days 3650
   ```
4. Install helm
   ```
   helm init --service-account tiller --tiller-tls --tiller-tls-cert ./tiller.cert.pem --tiller-tls-key ./tiller.key.pem --tiller-tls-verify --tls-ca-cert ca.cert.pem
   ```
   for kubernetes v1.16 (source: https://github.com/helm/helm/issues/6374#issuecomment-533427268)
   ```
   helm init --service-account tiller --tiller-tls --tiller-tls-cert ./tiller.cert.pem --tiller-tls-key ./tiller.key.pem --tiller-tls-verify --tls-ca-cert ca.cert.pem --override spec.selector.matchLabels.'name'='tiller',spec.selector.matchLabels.'app'='helm' --output yaml | sed 's@apiVersion: extensions/v1beta1@apiVersion: apps/v1@' | kubectl apply -f -
   ```
5. Copy cert to helm home dir
   ```
   cp ca.cert.pem $(helm home)/ca.pem
   cp helm.cert.pem $(helm home)/cert.pem
   cp helm.key.pem $(helm home)/key.pem
   ```
6. Amankan instalasi helm yang telah kita lakukan
   ```
   kubectl -n kube-system delete svc tiller-deploy
   kubectl -n kube-system patch deployment tiller-deploy --patch '
    spec:
    template:
        spec:
        containers:
            - name: tiller
            ports: []
            command: ["/tiller"]
            args: ["--listen=localhost:44134"]
    '
   ```

## Test helm to deploy nginx chart
1. Deploy nginx chart dari repo ini. Chart ini akan mendeploy nginx dengan nodeport 30080
   ```
   helm install --name=nginx https://github.com/akhfa/helm-chart-example/blob/master/nginx.tar.gz\?raw\=true
   ```
2. Cek deployment dan service yang ter-create di kubernetes
   ```
    $ kubectl get deployments.
    NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    nginx   1         1         1            1           16s
    $ kubectl get svc         
    NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
    kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        52d
    nginx        NodePort    10.105.137.190   <none>        80:30080/TCP   22s
   ```
3. Access menggunakan browser pada alamat 192.168.99.100:30080
