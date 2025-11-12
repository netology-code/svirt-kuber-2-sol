# Эталонное решение: Kubernetes ч.2

## Задание 1: Развертывание кластера

### Пошаговое решение

#### Подготовка всех узлов (master + 2 workers)

**1. Отключение swap:**
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

**2. Настройка модулей ядра:**
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

**3. Настройка сетевых параметров:**
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

**4. Установка containerd:**
```bash
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

**5. Установка kubeadm, kubelet, kubectl:**
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

#### Инициализация Control Plane (только на master)

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --upload-certs

# ⚠️ ВАЖНО: Сохраните вывод команды! Будет содержать команду join для worker узлов
```

**Настройка kubectl:**
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Установка CNI плагина (Calico):**
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

# Ожидание готовности узлов
kubectl get nodes -w
# STATUS должен стать Ready
```

#### Присоединение Worker узлов

На каждом worker узле выполните команду из вывода `kubeadm init`:
```bash
sudo kubeadm join <control-plane-host>:<control-plane-port> \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

#### Проверка кластера

```bash
# На master узле
kubectl get nodes
kubectl get pods -A
kubectl cluster-info

```

### Отчет по заданию 1

**Выбранный CNI:** Calico
**Обоснование:** 
- Поддержка Network Policies для безопасности
- Хорошая производительность
- Широкое использование в production
- Простая установка

**CIDR ranges:**
- Pod network: 10.244.0.0/16
- Service network: 10.96.0.0/12 (по умолчанию)

---

## Задание 2: Helm практика

### Структура чарта

```
helm-chart/
├── Chart.yaml
├── values.yaml
├── README.md
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── ingress.yaml
    └── _helpers.tpl
```

### Chart.yaml

```yaml
apiVersion: v2
name: myapp
description: A Helm chart for Kubernetes
type: application
version: 1.0.0
appVersion: "1.21"
```

### values.yaml

```yaml
# Количество реплик
replicaCount: 3

# Образ приложения
image:
  repository: nginx
  tag: "1.21"
  pullPolicy: IfNotPresent

# Конфигурация приложения (ConfigMap)
config:
  app_name: "myapp"
  environment: "production"
  log_level: "info"

# Секреты (Secret)
secret:
  database_password: "changeme"
  api_key: "secret-api-key"

# Service конфигурация
service:
  type: ClusterIP
  port: 80
  targetPort: 80

# Ingress конфигурация
ingress:
  enabled: false
  className: "nginx"
  annotations: {}
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

# Resource limits и requests
resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

# Health checks
livenessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5

# Labels и annotations
labels: {}
annotations: {}

# Node selector
nodeSelector: {}

# Tolerations
tolerations: []

# Affinity
affinity: {}
```

### Команды для работы с чартом

```bash
# 1. Создание чарта
helm create myapp-chart
# Удалить стандартные файлы и заменить на наши

# 2. Проверка синтаксиса
helm lint ./helm-chart

# 3. Предпросмотр манифестов
helm template ./helm-chart

# 4. Dry-run установка
helm install myapp ./helm-chart --dry-run --debug

# 5. Установка чарта
helm install myapp ./helm-chart

# 6. Установка с кастомными значениями
helm install myapp ./helm-chart -f custom-values.yaml

# 7. Проверка установки
helm list
helm status myapp
kubectl get pods -l app.kubernetes.io/instance=myapp
kubectl get svc -l app.kubernetes.io/instance=myapp

# 8. Обновление чарта
helm upgrade myapp ./helm-chart
helm upgrade myapp ./helm-chart --set replicaCount=5

# 9. История релизов
helm history myapp

# 10. Rollback
helm rollback myapp 1

# 11. Удаление
helm uninstall myapp
```

---

## Задание 3: Документация

### Архитектура кластера

**Схема кластера:**


**Сетевая топология:**

### Архитектурные решения

**1. Выбор CNI плагина: Calico**
- **Причина:** Поддержка Network Policies для изоляции трафика
- **Преимущества:** 
  - Безопасность на уровне сети
  - Хорошая производительность
  - Широкое использование в production

**2. Количество реплик: 3**
- **Причина:** Обеспечение высокой доступности
- **Обоснование:** При падении одной реплики остаются 2 работающие

**3. Resource limits:**
- **CPU:** requests 100m, limits 200m
- **Memory:** requests 128Mi, limits 256Mi
- **Причина:** Предотвращение истощения ресурсов узлов

**4. Health checks:**
- **Liveness probe:** Проверка каждые 10 секунд после 30 секунд задержки
- **Readiness probe:** Проверка каждые 5 секунд после 5 секунд задержки
- **Причина:** Быстрое обнаружение проблем и корректная маршрутизация трафика

### Production readiness

**Что нужно добавить для production:**

**1. Мониторинг и логирование:**
- Prometheus + Grafana для метрик
- ELK Stack или Loki для логов
- Alertmanager для алертов

**2. Backup стратегия:**
- Регулярные backup etcd (ежедневно)
- Backup конфигураций через GitOps
- Velero для backup приложений

**3. Безопасность:**
- RBAC для контроля доступа
- Network Policies для изоляции трафика
- Pod Security Policies/Pod Security Standards
- Шифрование etcd at rest
- Regular security updates

**4. Автоскейлинг:**
- Horizontal Pod Autoscaler (HPA)
- Vertical Pod Autoscaler (VPA)
- Cluster Autoscaler (для облачных провайдеров)

**5. Disaster recovery:**
- Регулярные backup etcd
- Документированный процесс восстановления
- Тестирование восстановления
- Multi-zone deployment для HA

**6. Дополнительно:**
- Ingress Controller (nginx-ingress или traefik)
- Certificate Manager (cert-manager) для TLS
- Resource Quotas и Limit Ranges
- Pod Disruption Budgets