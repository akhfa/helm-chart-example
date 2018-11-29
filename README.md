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
3. Install helm
   ```
   helm init --service-account tiller
   ```
4. Amankan instalasi helm yang telah kita lakukan
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
