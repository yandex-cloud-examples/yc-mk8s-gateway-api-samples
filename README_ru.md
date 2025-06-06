# Сценарии использования Gateway API в Managed Kubernetes

В этой папке содержатся примеры манифестов K8s для настройки [шлюзов](https://gateway-api.sigs.k8s.io/) Kubernetes для продакш-среды (Prod) и среды разработки (Dev). Среды развернуты в разных пространствах имен. Шлюзы настроены для привязки только к определенным пространствам имен для целей изоляции. TLS-секреты размещены в отдельном пространстве имен.

## Предварительные условия

- Кластер Managed Kubernetes в Yandex Cloud.
- Установленный и инициализированный [YC CLI](https://yandex.cloud/docs/cli/operations/install-cli).
- Публичный домен, [делегированный в Cloud DNS](https://yandex.cloud/docs/dns/concepts/dns-zone#public-zones) Yandex Cloud.

## Установка Gateway API

- Установите Gateway API из Marketplace. В консоли перейдите в Managed Service for Kubernetes → Имя нужного кластера → Marketplace → Gateway API → Использовать.
- Заполните все необходимые поля и установите Gateway API.

## Пример шлюзов и окружений Prod и Dev

### Создайте зарезервированные IP-адреса

```bash
yc vpc address create --name=prod --labels reserved=true --external-ipv4 zone=ru-central1-b # change to zone of your cluster
yc vpc address create --name=dev --labels reserved=true --external-ipv4 zone=ru-central1-b # change to zone of your cluster
```

### Добавьте записи Prod и Dev в вашу зону

```bash
yc dns zone add-records --name my-domain --record '*.prod.example.com 60 A  <ip_address>'
yc dns zone add-records --name my-domain --record '*.dev.example.com 60 A  <ip_address>'
```

### При необходимости создайте TLS-сертификат и отдельное пространство имен для TLS-секретов

Создайте пространство имен для TLS-секретов:

```bash
kubectl create namespace gateway-api-tls-secrets
```

Создайте сертификат для шлюза Prod:

```bash
openssl req -x509 \
    -newkey rsa:4096 \
    -keyout gateway-key-prod.pem \
    -out gateway-cert-prod.pem \
    -nodes \
    -days 365 \
    -subj '/CN=*.prod.example.com'

kubectl create -n gateway-api-tls-secrets secret tls gateway-prod-tls \
    --cert=gateway-cert-prod.pem \
    --key=gateway-key-prod.pem
```

Создайте сертификат для шлюза Dev:

```bash
openssl req -x509 \
    -newkey rsa:4096 \
    -keyout gateway-key-dev.pem \
    -out gateway-cert-dev.pem \
    -nodes \
    -days 365 \
    -subj '/CN=*.dev.example.com'

kubectl create -n gateway-api-tls-secrets secret tls gateway-dev-tls \
    --cert=gateway-cert-dev.pem \
    --key=gateway-key-dev.pem
```

### Настройте IP-адреса и домены

В YAML-файлах укажите реальные IP-адреса вместо <ip_address> и замените `example.com` на ваши доменные имена.

### Примените манифесты

```bash
kubectl apply prod-gw.yaml
kubectl apply prod-app.yaml
kubectl apply prod-route.yaml

kubectl apply dev-gw.yaml
kubectl apply dev-app.yaml
kubectl apply dev-route.yaml
```

### Проверьте работоспособность приложений

Проверьте работоспособность приложений на `app.prod.example.com` и `app.dev.example.com`.
