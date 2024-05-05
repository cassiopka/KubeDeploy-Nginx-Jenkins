# KubeDeploy-Nginx-Jenkins
## Автоматическое развертывание Nginx с помощью Minikube, Helm и Jenkins
### Требования 
* Git
* VirtualBox
* Minikube
* Kubernetes
* Jenkins
* Helm

### Операционная Система
    Fedora 39

### Установка Git

```bash
sudo dnf install git
```

### Установка VirtualBox
Инструкция по установке: https://www.virtualbox.org/wiki/Linux_Downloads

1. Создаем и открываем файл 

```bash   
sudo nano /etc/yum.repos.d/virtualbox.repo
```

2. Добавляем следующие строчки, сохраняем и выходим из nano.

```repo
[virtualbox]
name=Fedora $releasever - $basearch - VirtualBox
baseurl=http://download.virtualbox.org/virtualbox/rpm/fedora/$releasever/$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://www.virtualbox.org/download/oracle_vbox_2016.asc
```

3. Проверяем наличие VirtualBox в yum

```bash
sudo yum repolist
```

4. Устанавливаем VirtualBox

```bash
sudo yum install VirtualBox
```

5. Проверяем установку 

```bash
VBoxManage --version
```

    7.0.16_rpmfusionr162802

### Установка Minikube

Инструкция по установке: https://minikube.sigs.k8s.io/docs/start/

1. Устанавливаем RPM пакет

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
sudo rpm -Uvh minikube-latest.x86_64.rpm
```

2. Проверяем установку 

```bash
minikube version
```

    minikube version: v1.33.0
    commit: 86fc9d54fca63f295d8737c8eacdbb7987e89c67

### Установка Kubectl

1. Отключение подкачки. Kubernetes настроен на выдачу ошибки установки при обнаружении подкачки.

```bash
sudo systemctl stop swap-create@zram0
sudo dnf remove zram-generator-defaults
sudo reboot now
```

2. Отключение брандмауэра. Kubeadm выдаст предупреждение об установке, если брандмауэр запущен. Текущий список портов и протоколов, используемых кластером Kubernetes: https://kubernetes.io/docs/reference/networking/ports-and-protocols/.

```bash
sudo systemctl disable --now firewalld
```

3. Установка дополнительных утилит iptables и iproute-tc

* iptables - это утилита для управления таблицами правил Netfilter, которые используются ядром Linux для фильтрации и перенаправления сетевого трафика. iptables позволяет администраторам создавать, удалять и изменять правила для управления входящим и исходящим трафиком.

* iproute-tc - это утилита для управления трафиком в Linux с использованием механизма Traffic Control (tc). iproute-tc позволяет администраторам создавать и управлять очередями, классами и фильтрами для управления трафиком в сети.


```bash
sudo dnf install iptables iproute-tc
```

4. Настройка переадресации IPv4 и фильтры моста: https://kubernetes.io/docs/setup/production-environment/container-runtimes/

```bash
sudo cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

5. Загрузите модули overlay и bridge filter.
* overlay - это модуль ядра, который реализует сетевой драйвер overlay. Сетевой драйвер overlay используется для создания виртуальных сетевых интерфейсов, которые могут быть использованы для создания сетей перекрытия (overlay networks). Сети перекрытия используются для соединения нескольких физических сетей в одну логическую сеть.

* br_netfilter - это модуль ядра, который реализует поддержку Netfilter для мостов (bridges). Netfilter - это фреймворк для фильтрации и перенаправления сетевого трафика в ядре Linux. br_netfilter позволяет администраторам использовать Netfilter для фильтрации и перенаправления трафика на мостах.

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

6. Добавление необходимых параметров sysctl

```bash
# sysctl params required by setup, params persist across reboots
sudo cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

7. Применение параметров sysctl без перезагрузки.

```bash
sudo sysctl --system
```

8. Установка Kubernetes

```bash
sudo dnf install kubernetes-client kubernetes-node kubernetes-kubeadm
```

9. Извлечение необходимых образов системных контейнеров для Kubernetes

```bash
sudo kubeadm config images pull
```

10. Запуск и включение kubelet. Kubelet будет находиться в аварийном цикле до тех пор, пока кластер не будет инициализирован.

```bash
sudo systemctl enable --now kubelet
```

11. Инициализация кластера

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16



    Your Kubernetes control-plane has initialized successfully!

    To start using your cluster, you need to run the following as a regular user:

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

    Alternatively, if you are the root user, you can run:

        export KUBECONFIG=/etc/kubernetes/admin.conf
```

12. Следуем инструкциям из вывода 

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

13. Разрешение машине control plane также запускать модули для приложений. В противном случае в кластере потребуется более одной машины.

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

14. Установка flannel в кластер, чтобы обеспечить сетевое взаимодействие кластера.

```bash
kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
```

15. Проверка

```bash
kubectl version

    Client Version: v1.30.0
    Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

### Установка helm

1. Установка из официального репозитория

```bash
sudo dnf install helm
```

1. Проверка

```bash
helm version

    version.BuildInfo{Version:"v3.14.4", GitCommit:"81c902a123462fd4052bc5e9aa9c513c4c8fc142", GitTreeState:"clean", GoVersion:"go1.21.9"}
```


### Создание и клонирование репозитория

```bash
gh repo clone cassiopka/KubeDeploy-Nginx-Jenkins
cd KubeDeploy-Nginx-Jenkins
```

### Развертывание minikube

1. Запуск локального кластера Kubernetes
* --vm-driver=virtualbox - указывает, что Minikube должен использовать VirtualBox в качестве драйвера виртуальной машины.
* --nat-nic-type=Am79C973 - указывает тип сетевого адаптера NAT, который будет использоваться виртуальной машиной. Am79C973 - это тип адаптера, который поддерживает сетевые карты с разделенным DMA.
* --host-only-nic-type=Am79C973 - указывает тип сетевого адаптера Host-only, который будет использоваться виртуальной машиной. Am79C973 - это тип адаптера, который поддерживает сетевые карты с разделенным DMA.
* --cpus 2 - указывает количество виртуальных процессоров, которые будут использоваться виртуальной машиной.
* --memory 2048 - указывает количество памяти, которое будет использоваться виртуальной машиной. 2048 - это количество памяти в мегабайтах.
* --disk-size 10g - указывает размер виртуального диска, который будет использоваться виртуальной машиной. 10g - это размер диска в гигабайтах.

```bash
minikube start --vm-driver=virtualbox --nat-nic-type=Am79C973 --host-only-nic-type=Am79C973 --cpus 2 --memory 2048 --disk-size 10g

```


### Написание Helm чарта

1. Генерация базовых наборов файлов helm чарта, включая манифесты Kubernetes, шаблоны и значения по умолчанию

```bash
helm create nginx-chart
```

2. Редактирование файла **nginx-chart/values.yaml**

```bash
nano nginx-chart/values.yaml
```

```YAML
# Default values for nginx-chart.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: nginx
  # Overrides the image tag whose default is the chart appVersion.
  tag: latest # Ставим тег для последней версии образа

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Automatically mount a ServiceAccount's API credentials?
  automount: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

podAnnotations: {}
podLabels: {}

podSecurityContext: {}
  # fsGroup: 2000

securityContext: {}
  # capabilities:
  #   drop:
  #   - ALL
  # readOnlyRootFilesystem: true
  # runAsNonRoot: true
  # runAsUser: 1000
service:
  type: NodePort
  port: 32080 # Ставим порт для нода 
  targetProt: 80
ingress:
  enabled: false
  className: ""
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

resources: {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

livenessProbe:
  httpGet:
    path: /
    port: http
readinessProbe:
  httpGet:
    path: /
    port: http

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80

# Additional volumes on the output Deployment definition.
volumes: []
# - name: foo
#   secret:
#     secretName: mysecret
#     optional: false

# Additional volumeMounts on the output Deployment definition.
volumeMounts: []
# - name: foo
#   mountPath: "/etc/foo"
#   readOnly: true

nodeSelector: {}

tolerations: []

affinity: {}

```

3. Редактирование файла **nginx-chart/templates/deployment.yaml**

```bash
nano nginx-chart/templates/deployment.yaml
```


```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "nginx-chart.fullname" . }}
  labels:
    {{- include "nginx-chart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "nginx-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
	{{- include "nginx-chart.labels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "nginx-chart.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80 # Ставим значение 80 
              protocol: TCP
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- with .Values.volumeMounts }}
          volumeMounts:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with .Values.volumes }}
      volumes:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}

```

4. Редактирование файла **nginx-chart/templates/service.yaml**

```bash
nano nginx-chart/templates/service.yaml
```

```YAML
apiVersion: v1
kind: Service
metadata:
  name: {{ include "nginx-chart.fullname" . }}
  labels:
    {{- include "nginx-chart.labels" . | nindent 4 }}

# Ставим значения из nginx-chart/values.yaml
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort:  {{ .Values.service.targetPort }}
      protocol: TCP
      name: http
  selector:
    {{- include "nginx-chart.selectorLabels" . | nindent 4 }}

spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }} 
      targetPort:  {{ .Values.service.targetPort }}
      protocol: TCP
      name: http

```

5. Запуск чарта

```bash
helm install nginx nginx-chart

    NAME: nginx
    LAST DEPLOYED: Sun May  5 20:18:28 2024
    NAMESPACE: default
    STATUS: deployed
    REVISION: 1
    NOTES:
    1. Get the application URL by running these commands:
        export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services nginx-nginx-chart)
        export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
        echo http://$NODE_IP:$NODE_PORT

```
6. Проверка 

```bash
kubectl get svc

    NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
    kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP           89m
    nginx-nginx-chart   NodePort    10.108.218.43   <none>        32080:32514/TCP   8m21s
```

```bash
kubectl get deploy

    NAME                READY   UP-TO-DATE   AVAILABLE   AGE
    nginx-nginx-chart   1/1     1            1           8m28s
```

```bash
kubectl get pods

    NAME                                 READY   STATUS    RESTARTS   AGE
    nginx-nginx-chart-67b8ffb98d-46g5j   1/1     Running   0          8m33s
```

7.Запуск nginx

```bash
kubectl port-forward service/nginx-nginx-chart 8081:32080

    Forwarding from 127.0.0.1:8081 -> 32080
    Forwarding from [::1]:8081 -> 32080
    Handling connection for 8081
```
8. Перейдем по адресу **localhost:8081** и увидим приветственную страницу nginx


### Развертывание Jenkins в Minikube

1. Добавление репозитория Jenkins 

```bash
helm repo add jenkins https://charts.jenkins.io
```
2. Обновление репозиториев 

```bash
helm repo update
```

3. Установка Jenkins

```bash
helm install jenkins jenkins/jenkins


    NAME: jenkins LAST DEPLOYED: Fri May 5 20:10:05 2024 NAMESPACE: jenkins STATUS: deployed REVISION: 1 NOTES:

        Get your 'admin' user password by running: kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
        Get the Jenkins URL to visit by running these commands in the same shell: echo http://127.0.0.1:8080 kubectl --namespace jenkins port-forward svc/jenkins 8080:8080
        Login with the password from step 1 and the username: admin
        Configure security realm and authorization strategy
        Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http://127.0.0.1:8080/configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos

    For more information on running Jenkins on Kubernetes, visit: https://cloud.google.com/solutions/jenkins-on-container-engine For more information about Jenkins Configuration as Code, visit: https://jenkins.io/projects/jcasc/ NOTE: Consider using a custom image with pre-installed plugins

```

4. Следуя инструкциям, получаем пароль

```bash
kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
```

5. Запускаем Jenkins локально

```bash
kubectl port-forward service/jenkins 8080:8080
```


6. Заходим в браузер по адресу localhost:8081
7. Вводим логин *admin* и пароль из пункта 4.

    
### Написание Jenkins Job, который устанавливает написанный чарт для установки Nginx на кластер Minikube 

1. Пушим чарт в репозиторий

```bash
git add nginx-chart/
git commit -m 'add chart'
git push
```

2. Заходим на главную страницу Jenkins
3. Вводим имя *"Install Jenkins"*
4. Выбираем *"Создать задачу со свободной конфигурацией"*
5. В настройках переходим к разделу *Управление исходным кодом* и выбираем *Git*
6. В поле вставляем репозиторий **https://github.com/cassiopka/KubeDeploy-Nginx-Jenkins.git**
7. Переходим к разделу *Шаги сборки* и выбираем *Выполнить команду shell*
8. Добавляем команды для установки helm и 

```bash
# Скачиваем архив Helm
echo "Downloading Helm..."
curl -fsSL -o helm.tar.gz https://get.helm.sh/helm-v3.7.1-linux-amd64.tar.gz

# Распаковываем архив Helm
echo "Extracting Helm archive..."
tar -zxvf helm.tar.gz

# Создаем директорию для установки Helm
echo "Creating directory for Helm installation..."
mkdir -p /home/jenkins/bin

# Перемещаем исполняемый файл Helm в директорию /home/jenkins/bin
echo "Moving Helm binary to /home/jenkins/bin..."
mv linux-amd64/helm /home/jenkins/bin/helm

# Добавляем путь к исполняемому файлу Helm в переменную PATH
echo "Adding Helm binary to PATH..."
export PATH=$PATH:/home/jenkins/bin

# Проверяем версию установленного Helm
echo "Checking Helm version..."
helm version
скрипта
# Устанавливаем чарт Nginx
echo "Installing Nginx chart..."
helm install nginx nginx-chart

# Оповещение о завершении скрипта
echo "Script execution completed."
```

9. Сохраняем
10. Сборщик выключен, из-за предупреждения, убираем ограничения памяти в настройках jenkins
11. Скачиваем утилиту jenkins-cli.jar для доступа к jenkins
12. Запускаем сборку 

```bash
java -jar jenkins-cli.jar -s http://192.168.59.105:32000/ -auth admin:$(kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password) build Install-Nginx -s -v
```
13. Видим уведомление об успешном завершении команд

```bash
Downloading Helm...
Extracting Helm archive...
Creating directory for Helm installation...
Moving Helm binary to /home/jenkins/bin...
Adding Helm binary to PATH...
Checking Helm version...
version.BuildInfo{Version:"v3.14.4", GitCommit:"81c902a123462fd4052bc5e9aa9c513c4c8fc142", GitTreeState:"clean", GoVersion:"go1.21.9"}
Installing Nginx chart...
NAME: nginx
LAST DEPLOYED: Tue Nov 23 15:30:14 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
    NOTES:
    1. Get the application URL by running these commands:
    export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=nginx,app.kubernetes.io/instance=nginx" -o jsonpath="{.items[0].metadata.name}")
    echo "Visit http://127.0.0.1:8080 to use your application"
    Script execution completed.
```
