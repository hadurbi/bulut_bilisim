Google Kubernetes Engine (GKE) ile Güvenli Web Uygulaması Dağıtımı ve CI/CD Yönetimi Proje Raporu 

Bu projede Go dili ile geliştirilmiş basit bir web uygulaması Docker kullanılarak container haline getirilmiş ve Google Kubernetes Engine (GKE) ortamında çalıştırılmıştır. 

Projede yüksek erişilebilirlik, ölçeklenebilirlik ve otomatik dağıtım süreçlerini göstermek amacıyla Kubernetes Deployment, Service, Persistent Volume (PV), Persistent Volume Claim (PVC), NetworkPolicy ve CI/CD pipeline yapıları kullanılmıştır. 

CI/CD süreçleri Google Cloud Build üzerinden otomatikleştirilmiş ve uygulama güncellemeleri Rolling Update mekanizması ile kesintisiz şekilde canlı ortama aktarılmıştır. 

1. Mimari Akışı 

Projenin sıfırdan canlıya alınma süreci şu kronolojik adımlarla gerçekleştirilmiştir: 

Dizin Standardizasyonu: Dosya karmaşasını ve derleme hatalarını önlemek adına Google Cloud Shell üzerinde projenin kök dizini olan kubernetes-engine-samples/quickstarts/hello-app klasörüne geçiş yapılmıştır. 

Deklaratif Manifestoların İnşa Edilmesi: Tüm kod, altyapı ve otomasyon dosyaları (main.go, proje.yaml, cloudbuild.yaml) herhangi bir sözdizimi hatası barındırmayacak şekilde bu kök dizinde sıfırdan oluşturulmuştur. 

Altyapının Kümede Deklare Edilmesi: kubectl apply -f proje.yaml komutu çalıştırılarak kalıcı veri diskleri (PV/PVC), 3 replikalı pod yapısı, harici yük dengeleyici (LoadBalancer) ve ağ güvenlik kuralları GKE üzerinde aktif edilmiştir. 

CI/CD Pipeline Otomasyonunun Başlatılması: gcloud builds submit --config cloudbuild.yaml . komutu ile bulut boru hattı tetiklenmiştir. Kod bulutta derlenmiş, Docker imajı üretilmiş ve kümedeki podlar kesintisiz olarak güncellenmiştir. 

Canlı Güncelleme ve Sıfır Kesinti (Zero-Downtime) Testi: Kaynak kod üzerinde yapılan bir metin değişikliğinin, sistemin çalışma sürekliliğini bozmadan Rolling Update mekanizmasıyla canlıya nasıl yansıtıldığı harici IP adresi üzerinden doğrulanmıştır. 

Projenin kod seviyesinden başlayarak Google Cloud Platform (GCP) üzerinde konumlanan canlı üretim ortamına kadar olan tam otomatik döngüsü ve boru hattı (pipeline) topolojisi aşağıda şematize edilmiştir:
Şema Bileşenleri ve Boru Hattı Analizi 

Projemizin akış diyagramı, geleneksel manuel dağıtım süreçlerini tamamen eleyen ve altyapıyı kod olarak yöneten (Infrastructure as Code) modern bir CI/CD (Sürekli Entegrasyon / Sürekli Dağıtım) boru hattını doğrulamaktadır:  

Geliştirici Katmanı (Developer): Süreç, kaynak kod üzerinde yerel veya bulut terminalinde bir değişiklik yapılması ve komut satırı üzerinden bu değişikliğin tetiklenmesiyle başlar.  

Tetikleme Mekanizması (Detect New Commit): Google Cloud Build mimarisi, proje kök dizininde gerçekleşen tetiklemeyi anlık olarak yakalar ve boru hattındaki adımları sırasıyla koşturmaya başlar.  

Sürekli Entegrasyon (Build & Push Container Image): Dockerfile yönergeleri doğrultusunda Go uygulaması Multi-Stage mimariyle derlenir. Sürüm çakışmalarını önlemek adına her bültene özgü dinamik $BUILD_ID çevre değişkeni atanarak paketlenen imaj, Google Artifact Registry havuzuna güvenli bir şekilde itilir.  

Sürekli Dağıtım (Apply Manifest & Update GKE Cluster): İmaj havuzuna push işleminin tamamlanmasının ardından pipeline, projenin kalbi olan proje.yaml altyapı bildirimlerini Kubernetes API'sine deklare eder (kubectl apply). Son adımda kubectl set image komutu yürütülerek Google Kubernetes Engine (GKE) kümesindeki canlı podlar, kullanıcı trafiğinde hiçbir kesintiye yol açmadan Rolling Update mekanizmasıyla güncellenir.
2. Güncel Kod Dosyaları ve Satır Satır Teknik Analizleri 

2.1. Uygulama Katmanı (main.go) 

Go dilinin güçlü standart kütüphanesi (net/http) kullanılarak yazılmış, minimum kaynak tüketen HTTP web sunucu kodudur.
Go 

package main 
 
import ( 
	"fmt" 
	"log" 
	"net/http" 
	"os" 
) 
 
func main() { 
	// 1. HTTP İstek Yönlendiricisinin (Mux) Tanımlanması 
	mux := http.NewServeMux() 
	mux.HandleFunc("/", hello) 
 
	// 2. Çevre Değişkeninden (Environment Variable) Port Kontrolü 
	port := os.Getenv("PORT") 
	if port == "" { 
		port = "8080" // Port tanımlanmamışsa varsayılan olarak 8080 atanır 
	} 
 
	log.Printf("Server listening on port %s", port) 
	// 3. HTTP Sunucusunun Belirtilen Portta Dinlemeye Başlaması 
	log.Fatal(http.ListenAndServe(":"+port, mux)) 
} 
 
func hello(w http.ResponseWriter, r *http.Request) { 
	log.Printf("Serving request: %s", r.URL.Path) 
	 
	// 4. Dinamik Yük Dengeleme Doğrulaması (Kritik Satır) 
	host, _ := os.Hostname()  
	fmt.Fprintf(w, "Sunum provasi basariyla tamamlandi\n") 
	fmt.Fprintf(w, "Version: 1.0.0\n") 
	fmt.Fprintf(w, "Hostname: %s\n", host)  
}
Kritik Kod Açıklaması: host, _ := os.Hostname() satırı, tarayıcıdan gelen isteği yanıtlayan podun o anki işletim sistemi düzeyindeki benzersiz ağ adını (konteyner ID'sini) çeker. Sunum esnasında sayfa yenilendikçe bu değerin değişmesi, Load Balancer'ın (Yük Dengeleyici) trafiği podlar arasında adilce dağıttığının en somut kanıtıdır.
2.2. Paketleme Reçetesi (Dockerfile)
Bulut Konteyner Havuzu ve Dinamik Etiketleme Analizi 


Dinamik Sürüm Yönetimi ($BUILD_ID): Görselde yer alan imajların tag (etiket) kısımlarında sabit bir sürüm numarası (v1, v2 vb.) yerine, cloudbuild.yaml tarafından otomatik üretilen benzersiz karma değerler (Örn: hello-app:3595... veya hello-app:g72a...) yer almaktadır. 

Mühendislik Avantajı: Bu yapı, Kubernetes kümesinde imaj önbellekleme hatalarını ve ImagePullBackOff kilitlenmelerini tamamen engeller. Küme, her dağıtımda benzersiz bir etiket gördüğü için imaj havuzundan en güncel konteyner paketini çeker. 

Depolama ve Güvenlik: Dockerfile dosyasında uygulanan Multi-Stage Build optimizasyonu sayesinde, her bir imaj katmanı bulut üzerinde minimum boyutta (yaklaşık 2-3 MB) yer kaplayacak şekilde Artifact Registry'de depolanmakta ve GKE kümesine saniyeler içinde dağıtılmaya hazır halde bekletilmektedir. 

Uygulamanın siber güvenlik ve performans standartlarına uygun olarak paketlenmesini sağlayan Multi-Stage Build (Çok Aşamalı Derleme) mimarisidir.
Dockerfile 

# --- 1. AŞAMA: DERLEME (BUILD STAGE) --- 
FROM golang:1.26.2 AS builder 
WORKDIR /app 
COPY go.mod ./ 
RUN go mod download 
COPY *.go ./ 
# Go kaynak kodunun CGO bağımlılığı olmadan static binary olarak derlenmesi 
RUN CGO_ENABLED=0 GOOS=linux go build -o /hello-app 
 
# --- 2. AŞAMA: ÜRETİM (PRODUCTION STAGE) --- 
FROM gcr.io/distroless/static-debian12 
COPY --from=builder /hello-app /hello-app 
USER nonroot:nonroot 
CMD ["/hello-app"]
Kritik Kod Açıklaması: FROM gcr.io/distroless/static-debian12 katmanı projenin kurumsal kalitesini belirler. İçinde terminal (bash), paket yöneticisi (apt) gibi siber saldırganların sızdığında kullanabileceği hiçbir araç barındırmayan, sadece 2 MB boyutunda minimal bir taban imaj kullanılmıştır. Ayrıca USER nonroot komutuyla uygulamanın root yetkileri kısıtlanmıştır.
2.3. Kubernetes Altyapı Manifestosu (proje.yaml) 

Sistemin veri kalıcılığını, replikasyonunu, dış dünyaya açılmasını ve izolasyonunu sağlayan bütünleşik deklaratif dosyadır.
YAML 

# --- PERSISTENT VOLUME (FİZİKSEL DEPOLAMA TAAHHÜDÜ) --- 
apiVersion: v1 
kind: PersistentVolume 
metadata: 
  name: bulut-pv 
spec: 
  capacity: 
    storage: 1Gi 
  accessModes: 
    - ReadWriteOnce 
  persistentVolumeReclaimPolicy: Retain 
  storageClassName: standard 
  gcePersistentDisk: 
    pdName: bulut-disk # GCP üzerinde manuel oluşturulan disk adı 
    fsType: ext4 
--- 
# --- PERSISTENT VOLUME CLAIM (DİSK TALEP KATMANI) --- 
apiVersion: v1 
kind: PersistentVolumeClaim 
metadata: 
  name: bulut-pvc 
spec: 
  storageClassName: standard 
  accessModes: 
    - ReadWriteOnce 
  resources: 
    requests: 
      storage: 1Gi 
--- 
# --- DEPLOYMENT (YÜKSEK ERİŞİLEBİLİRLİK VE REPLİKASYON) --- 
apiVersion: apps/v1 
kind: Deployment 
metadata: 
  name: hello-deployment 
  labels: 
    app: hello-app 
spec: 
  replicas: 3 # Aynı anda yan yana çalışan pod sayısı (Scaling) 
  strategy: 
    type: RollingUpdate # Sıfır kesintiyle güncelleme stratejisi 
    rollingUpdate: 
      maxUnavailable: 1 # Güncelleme sırasında en fazla 1 pod çevrimdışı olabilir 
      maxSurge: 1       # Güncelleme sırasında en fazla 1 yeni pod fazladan açılabilir 
  selector: 
    matchLabels: 
      app: hello-app 
  template: 
    metadata: 
      labels: 
        app: hello-app 
    spec: 
      containers: 
      - name: hello-app 
        image: gcr.io/google-samples/hello-app:1.0 
        ports: 
        - containerPort: 8080 
        volumeMounts: 
        - mountPath: /data # Konteyner içinde kalıcı verinin yazılacağı dizin 
          name: depolama-alani 
      volumes: 
      - name: depolama-alani 
        persistentVolumeClaim: 
          claimName: bulut-pvc 
--- 
# --- SERVICE (DIŞ DÜNYAYA AÇMA VE YÜK DENGELEME) --- 
apiVersion: v1 
kind: Service 
metadata: 
  name: hello-service 
spec: 
  type: LoadBalancer # GCP üzerinde harici statik IP adresi tahsis eder 
  ports: 
  - port: 80 
    targetPort: 8080 
  selector: 
    app: hello-app 
--- 
# --- NETWORK POLICY (SİBER GÜVENLİK VE ERİŞİM KONTROLÜ) --- 
apiVersion: networking.k8s.io/v1 
kind: NetworkPolicy 
metadata: 
  name: bulut-network-policy 
spec: 
  podSelector: 
    matchLabels: 
      app: hello-app # Bu politikanın uygulanacağı hedef pod etiketleri 
  policyTypes: 
  - Ingress 
  ingress: 
  - from: 
    - ipBlock: 
        cidr: 0.0.0.0/0 # Dış dünyadaki kullanıcıların web sitesine erişim izni
Kritik Kod Açıklaması: replicas: 3 ve strategy: RollingUpdate katmanları projenin omurgasıdır. Kod güncellendiğinde 3 pod birden kapatılmaz; sistem eski sürümü ayakta tutarken yeni podları tek tek sırayla devreye alır, trafiği kaydırır ve güncellemeyi sıfır kesintiyle tamamlar.
2.4. Otomatik CI/CD Yapılandırması (cloudbuild.yaml) 

Google Cloud Build alt yapısında çalışan, insan hatasını sıfırlayan Sürekli Entegrasyon boru hattıdır.
YAML 

steps: 
  # 1. Adım: Docker imajını benzersiz BUILD_ID ile inşa etme (Çakışma Önleme) 
  - name: 'gcr.io/cloud-builders/docker' 
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/hello-app:$BUILD_ID', '.'] 
 
  # 2. Adım: Üretilen imajı Google Container Registry havuzuna güvenli şekilde pushlama 
  - name: 'gcr.io/cloud-builders/docker' 
    args: ['push', 'gcr.io/$PROJECT_ID/hello-app:$BUILD_ID'] 
 
  # 3. Adım: GKE Kümesine bağlanıp canlı mimarideki imajı anında güncelleme 
  - name: 'gcr.io/cloud-builders/kubectl' 
    args: 
      - 'set' 
      - 'image' 
      - 'deployment/hello-deployment' 
      - 'hello-app=gcr.io/$PROJECT_ID/hello-app:$BUILD_ID' 
    env: 
      - 'CLOUDSDK_COMPUTE_ZONE=us-central1-a' 
      - 'CLOUDSDK_CONTAINER_CLUSTER=bulut-cluster'
Kritik Kod Açıklaması: Sabit bir imaj etiketi (v2 gibi) kullanmak yerine bulut yerel çevre değişkeni olan $BUILD_ID kullanılmıştır. Bu sayede her komut koştuğunda benzersiz bir imaj kimliği üretilir. Kubernetes de gelen imajın yepyeni bir sürüm olduğunu anlar ve imaj kilitlenme hatalarına (ImagePullBackOff) düşmeden canlı sistemi günceller