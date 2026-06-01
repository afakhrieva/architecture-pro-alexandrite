# Jaeger в Minikube с сервисами

## Описание
Развертывание Jaeger в Minikube с двумя сервисами, которые:
1. Взаимодействуют между собой
2. Отправляют трейсы в Jaeger

## Требования
- Minikube
- kubectl
- Docker

## Установка

### 1. Запуск Minikube 
```bash
minikube start --addons=ingress 
```
Ingress нужен для вызовов

### 2. Установка cert-manager - обновила версию
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.20.2/cert-manager.yaml
```

### 3. Развертывание Jaeger
```bash
kubectl create namespace observability
kubectl create -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.51.0/jaeger-operator.yaml -n observability
kubectl apply -f k8s/jaeger-instance.yaml
```

**ЭТО НЕ РАБОТАЕТ**
```bash
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  55s                default-scheduler  Successfully assigned observability/jaeger-operator-777fb677c7-wzkll to minikube
  Normal   Pulling    54s                kubelet            spec.containers{jaeger-operator}: Pulling image "quay.io/jaegertracing/jaeger-operator:1.47.0"
  Normal   Pulled     37s                kubelet            spec.containers{jaeger-operator}: Successfully pulled image "quay.io/jaegertracing/jaeger-operator:1.47.0" in 17.503s (17.503s including waiting). Image size: 303021466 bytes.
  Normal   Created    36s                kubelet            spec.containers{jaeger-operator}: Container created
  Normal   Started    36s                kubelet            spec.containers{jaeger-operator}: Container started
  Normal   Pulling    23s (x2 over 36s)  kubelet            spec.containers{kube-rbac-proxy}: Pulling image "gcr.io/kubebuilder/kube-rbac-proxy:v0.13.1"
  Warning  Failed     22s (x2 over 35s)  kubelet            spec.containers{kube-rbac-proxy}: Failed to pull image "gcr.io/kubebuilder/kube-rbac-proxy:v0.13.1": Error response from daemon: manifest for gcr.io/kubebuilder/kube-rbac-proxy:v0.13.1 not found: manifest unknown: Failed to fetch "v0.13.1"
  Warning  Failed     22s (x2 over 35s)  kubelet            spec.containers{kube-rbac-proxy}: Error: ErrImagePull
  Normal   BackOff    8s (x3 over 34s)   kubelet            spec.containers{kube-rbac-proxy}: Back-off pulling image "gcr.io/kubebuilder/kube-rbac-proxy:v0.13.1"
  Warning  Failed     8s (x3 over 34s)   kubelet            spec.containers{kube-rbac-proxy}: Error: ImagePullBackOff
```
https://github.com/kubernetes-sigs/kubebuilder/discussions/3907#discussion-6670554

Удалось установить через helm
```bash
# Добавьте репозиторий Jaeger
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
# Обновите его
helm repo update
# Установите оператор из Helm-чарта
helm install jaeger-operator jaegertracing/jaeger-operator \
  --namespace observability \
  --create-namespace \
  --set rbac.clusterRole=true # это обязательно
```

Запуск
```
kubectl apply -f k8s/jaeger-instance.yaml -n observability
```

### 4. Сборка и деплой сервисов
```bash
# Сборка образов
minikube image build -t service-a:latest services/service-a/
minikube image build -t service-b:latest services/service-b/

# Развертывание (!!! добавила -n observability)
kubectl apply -f k8s/services.yaml -n observability
```

## Проверка работы

### Доступ к Jaeger UI
```bash
kubectl port-forward svc/simplest-query 16686:16686
```
Откройте в браузере: http://localhost:16686

### Тестирование сервисов
```bash
# Вызов service-a, который вызывает service-b
kubectl exec -it $(kubectl get pods -l app=service-a -o jsonpath='{.items[0].metadata.name}') -- wget -qO- http://service-a:8080
```
-выдает фигню, делаю так
```bash
kubectl run test-pod --image=alpine/curl -it --rm --restart=Never -n observability -- sh

/ # curl http://service-a:8080 
```

## Структура проекта
- `services/service-a/` - Исходный код service-a
- `services/service-b/` - Исходный код service-b  
- `k8s/services.yaml` - Конфигурация Kubernetes для сервисов
- `jaeger-instance.yaml` - Конфигурация Jaeger