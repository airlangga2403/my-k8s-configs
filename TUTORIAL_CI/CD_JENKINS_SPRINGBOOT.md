Tentu, ini adalah konten `README.md` yang sama seperti sebelumnya, disajikan sebagai blok kode biasa agar mudah Anda salin.

````markdown
# Tutorial CI/CD Lengkap: Spring Boot dengan Jenkins, Kubernetes, & Argo CD

Dokumen ini menjelaskan langkah-langkah untuk membangun pipeline CI/CD (Continuous Integration & Continuous Deployment) yang sepenuhnya otomatis. Setiap kali Anda melakukan `push` kode ke repositori Spring Boot, Jenkins akan secara otomatis membuat Docker image, mendorongnya ke registry, memperbarui manifestasi deployment di repositori Git, dan Argo CD akan men-deploy versi terbaru tersebut ke cluster Kubernetes.

## Alur Kerja (Workflow)

![Alur Kerja CI/CD](https://i.imgur.com/gA3fHxq.png)

1.  **Developer** melakukan `git push` ke branch `master` di repositori kode aplikasi.
2.  **GitHub** mengirimkan notifikasi (via Webhook) ke Jenkins.
3.  **Jenkins** secara otomatis memulai build pipeline:
    a. Mengambil kode sumber terbaru.
    b. Mengompilasi aplikasi dan membuat file `.jar` menggunakan Maven.
    c. Membangun Docker image baru dari file `.jar` tersebut.
    d. Mendorong (push) Docker image baru ke Docker Hub dengan tag unik (commit hash).
    e. Meng-clone repositori konfigurasi Kubernetes.
    f. Mengedit file `deployment.yaml` untuk menunjuk ke tag image yang baru.
    g. Melakukan `commit` dan `push` perubahan `deployment.yaml` kembali ke repositori konfigurasi.
4.  **Argo CD** yang terus memantau repositori konfigurasi, mendeteksi adanya perubahan baru.
5.  **Argo CD** secara otomatis menerapkan `deployment.yaml` yang baru ke **Cluster Kubernetes**, yang kemudian akan menarik (pull) image Docker terbaru dari Docker Hub dan menjalankan versi aplikasi yang baru.

---

## 1. Prasyarat

Sebelum memulai, pastikan Anda memiliki:
* Sebuah cluster Kubernetes yang aktif.
* `kubectl` terinstal dan terkonfigurasi untuk mengakses cluster Anda.
* Akun GitHub.
* Akun Docker Hub.
* Helm (opsional, untuk instalasi Jenkins/Argo CD yang lebih mudah).

---

## 2. Struktur Repositori (GitOps)

Kita akan menggunakan dua repositori Git:
1.  **Repositori Aplikasi**: Berisi kode sumber aplikasi Spring Boot Anda, beserta `Dockerfile` dan `Jenkinsfile`.
    * Contoh: `https://github.com/airlangga2403/kai-test-kafka.git`
2.  **Repositori Konfigurasi**: Berisi semua file manifestasi Kubernetes (`.yaml`) untuk men-deploy aplikasi Anda. Ini adalah *single source of truth* untuk Argo CD.
    * Contoh: `https://github.com/airlangga2403/my-k8s-configs.git`

---

## 3. Instalasi Jenkins di Kubernetes

Kita akan men-deploy Jenkins Controller ke dalam cluster K8s.

**File YAML yang Dibutuhkan:**

Simpan file-file berikut dan terapkan ke cluster Anda menggunakan `kubectl apply -f <nama-file.yaml>`.

#### `01-namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
````

#### `02-rbac.yaml`

(Memberikan izin agar Jenkins bisa membuat Pod agen di dalam cluster)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-sa
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins-role
  namespace: jenkins
rules:
- apiGroups: [""]
  resources: ["pods", "pods/exec", "pods/log", "secrets"]
  verbs: ["get", "list", "watch", "create", "delete", "patch", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-role-binding
  namespace: jenkins
subjects:
- kind: ServiceAccount
  name: jenkins-sa
roleRef:
  kind: Role
  name: jenkins-role
  apiGroup: rbac.authorization.k8s.io
```

#### `03-pvc.yaml`

(Penyimpanan permanen untuk data Jenkins)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

#### `04-deployment.yaml`

(Deployment utama untuk Jenkins Controller)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins-deployment
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins-sa
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts-jdk17
        env:
        - name: JAVA_OPTS
          value: "-Dorg.jenkinsci.plugins.workflow.cps.CpsVmExecutorService.USE_WEB_SOCKET=true"
        ports:
        - containerPort: 8080
          name: web
        - containerPort: 50000
          name: agent
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
      volumes:
      - name: jenkins-home
        persistentVolumeClaim:
          claimName: jenkins-pvc
```

#### `05-service.yaml`

(Mengekspos Jenkins UI ke luar cluster)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  namespace: jenkins
spec:
  type: NodePort
  selector:
    app: jenkins
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 32005 # Ganti jika port ini sudah terpakai
    name: web
```

**Terapkan ke Cluster:**

```bash
kubectl apply -f 01-namespace.yaml
kubectl apply -f 02-rbac.yaml
kubectl apply -f 03-pvc.yaml
kubectl apply -f 04-deployment.yaml
kubectl apply -f 05-service.yaml
```

-----

## 4\. Konfigurasi Awal Jenkins

1.  **Dapatkan Password Admin:**
    ```bash
    kubectl exec -it $(kubectl get pods -n jenkins -l app=jenkins -o jsonpath='{.items[0].metadata.name}') -n jenkins -- cat /var/jenkins_home/secrets/initialAdminPassword
    ```
2.  **Akses Jenkins UI:** Buka `http://<IP-Node-Anda>:32005` di browser. Masukkan password di atas.
3.  **Install Plugin:** Pilih "Install suggested plugins".
4.  **Buat Akun Admin** Anda sendiri.
5.  **Konfigurasi Kredensial:**
      * Pergi ke **Manage Jenkins** -\> **Credentials** -\> **System** -\> **Global credentials**.
      * **Docker Hub:** Klik **Add Credentials**.
          * **Kind:** `Username with password`
          * **Username:** Username Docker Hub Anda (contoh: `airlangga22`)
          * **Password:** Password Docker Hub Anda.
          * **ID:** `dockerhub-creds` (harus sama persis)
      * **GitHub:** Klik **Add Credentials** lagi.
          * **Kind:** `Username with password`
          * **Username:** Username GitHub Anda (contoh: `airlangga2403`)
          * **Password:** Buat **Personal Access Token (PAT)** dari GitHub dengan scope `repo` dan `workflow`, lalu masukkan token tersebut di sini.
          * **ID:** `github-token` (harus sama persis)

-----

## 5\. Konfigurasi Cloud Kubernetes di Jenkins

Beritahu Jenkins cara membuat agen build di dalam cluster K8s.

1.  Pergi ke **Manage Jenkins** -\> **Plugins** -\> **Available Plugins**. Cari dan install `Kubernetes`.
2.  Pergi ke **Manage Jenkins** -\> **Nodes and Clouds** -\> **Configure Clouds**.
3.  Klik **`Add a new cloud`** dan pilih **`Kubernetes`**.
4.  Isi formulir:
      * **Name:** `kubernetes`
      * **Kubernetes URL:** Biarkan kosong (Jenkins akan mendeteksi secara otomatis dari dalam Pod).
      * **Kubernetes Namespace:** `jenkins`
      * **Jenkins URL:** `http://jenkins-service.jenkins.svc.cluster.local:8080`
      * Centang (✓) **`Enable WebSocket`**.
5.  Klik **`Test Connection`**. Seharusnya muncul pesan "Connection successful".
6.  Klik **Save**.

-----

## 6\. Setup Repositori Aplikasi

#### `Dockerfile`

Gunakan multi-stage build untuk membuat image yang kecil dan efisien.

```dockerfile
# Stage 1: Build environment
FROM maven:3.6-openjdk-17 AS build-env
WORKDIR /app
COPY . ./
# Gunakan cache maven agar build lebih cepat
RUN --mount=type=cache,target=/root/.m2 mvn clean package -Dmaven.test.skip=true

# Stage 2: Final image
FROM openjdk:17.0
WORKDIR /app
# Gunakan wildcard (*) agar tidak bergantung pada versi spesifik
COPY --from=build-env /app/target/*.jar app.jar
ENTRYPOINT ["java","-jar","app.jar"]
```

#### `Jenkinsfile`

Ini adalah jantung dari pipeline Anda. Salin `Jenkinsfile` yang sudah kita perbaiki bersama dan simpan di root repositori aplikasi Anda.

-----

## 7\. Setup Repositori Konfigurasi K8s

Di dalam repositori `my-k8s-configs`, buat sebuah direktori yang namanya sama dengan `APP_NAME` di `Jenkinsfile` (contoh: `kai-test-kafka`). Di dalamnya, buat file-file berikut:

#### `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kai-test-kafka
```

#### `deployment.yaml`

Perhatikan blok `resources` dan placeholder `image`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kai-test-kafka-deployment
  namespace: kai-test-kafka
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kai-test-kafka
  template:
    metadata:
      labels:
        app: kai-test-kafka
    spec:
      imagePullSecrets:
      - name: dockerhub-creds
      containers:
      - name: app
        # PENTING: Jenkins akan mengganti baris ini secara otomatis
        image: airlangga22/kai-test-kafka:initial
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
      volumes:
      - name: config-volume
        configMap:
          name: kai-test-kafka-config
```

#### `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kai-test-kafka-service
  namespace: kai-test-kafka
spec:
  type: NodePort
  selector:
    app: kai-test-kafka
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30820 # Pastikan port ini unik di cluster Anda
    name: http
```

#### `configmap.yaml`

Penting untuk mengatur `context-path` agar URL Anda berfungsi.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kai-test-kafka-config
  namespace: kai-test-kafka
data:
  application.yml: |
    server:
      servlet:
        context-path: /kai-test-kafka # <--- PENTING
      port: 8080
    # ... sisa konfigurasi Spring Anda ...
```

-----

## 8\. Setup Argo CD & Aplikasi

1.  **Install Argo CD** di cluster Anda (ikuti dokumentasi resminya).
2.  **Buat Secret Docker Hub:** Agar Argo CD bisa men-deploy aplikasi Anda, K8s harus bisa menarik image private. Buat secret di namespace aplikasi Anda:
    ```bash
    kubectl create secret docker-registry dockerhub-creds \
      --namespace=kai-test-kafka \
      --docker-server=[https://index.docker.io/v1/](https://index.docker.io/v1/) \
      --docker-username=airlangga22 \
      --docker-password=<PASSWORD_DOCKERHUB_ANDA>
    ```
3.  **Buat Aplikasi di Argo CD:**
    Buat file `argo-application.yaml` dan terapkan ke cluster.
    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: kai-test-kafka
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: '[https://github.com/airlangga2403/my-k8s-configs.git](https://github.com/airlangga2403/my-k8s-configs.git)'
        path: kai-test-kafka # Path ke direktori manifestasi Anda
        targetRevision: HEAD
      destination:
        server: [https://kubernetes.default.svc](https://kubernetes.default.svc)
        namespace: kai-test-kafka
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
    ```
    Terapkan file tersebut:
    ```bash
    kubectl apply -f argo-application.yaml
    ```

-----

## 9\. Membuat Job Multibranch Pipeline & Webhook

1.  **Buat Job di Jenkins:**
      * Di dashboard Jenkins, klik **New Item**.
      * Beri nama (misal: `proyek-springboot`).
      * Pilih **Multibranch Pipeline** dan klik OK.
2.  **Konfigurasi Sumber Branch:**
      * Di halaman konfigurasi, pergi ke **Branch Sources**.
      * Klik **Add source** -\> **GitHub**.
      * **Credentials:** Pilih kredensial GitHub Anda (`github-token`).
      * **Repository HTTPS URL:** Masukkan URL repo aplikasi Anda (`https://github.com/airlangga2403/kai-test-kafka.git`).
      * Klik **Save**. Jenkins akan otomatis memindai repo dan membuat job untuk setiap branch yang memiliki `Jenkinsfile`.
3.  **Konfigurasi Webhook di GitHub:**
      * Buka repo aplikasi Anda di GitHub -\> **Settings** -\> **Webhooks**.
      * **Add webhook**.
      * **Payload URL:** `http://<IP-Node-Anda>:32005/github-webhook/`
      * **Content type:** `application/json`
      * Pilih **"Just the `push` event."**
      * Klik **Add webhook**.

-----

## 10\. Troubleshooting

  * **Build Gagal di Jenkins:** Periksa "Console Output" dari build yang gagal untuk melihat pesan error spesifik.
  * **Pod Status `ImagePullBackOff`:**
      * Pastikan secret `dockerhub-creds` ada di namespace aplikasi (`kai-test-kafka`).
      * Pastikan password di dalam secret sudah benar. Jika salah, hapus (`kubectl delete secret ...`) dan buat ulang.
  * **Pod Status `CrashLoopBackOff`:**
      * Periksa log pod dengan: `kubectl logs <nama-pod> -n kai-test-kafka`.
      * Kemungkinan besar ada error di aplikasi Spring Anda (gagal koneksi DB, Kafka, dll).
  * **Aplikasi Tidak Bisa Diakses dari Luar:**
      * Pastikan `NodePort` Anda (misal: `30820`) sudah diizinkan di **Firewall / Security Group** cloud provider Anda.

<!-- end list -->

```
```