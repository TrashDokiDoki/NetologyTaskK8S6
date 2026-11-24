# Домашнее задание к занятию «Настройка приложений и управление доступом в Kubernetes»

## Задание 1. Работа с ConfigMaps

### 1. Создать Deployment с двумя контейнерами

Создан Deployment `web-app`, содержащий два контейнера:

- `nginx`;
- `multitool`.

Для контейнера `multitool` HTTP-порт изменён на `8080`, чтобы избежать конфликта с nginx, использующим порт `80`.

Проверка состояния приложения:

```
kubectl get pods
```

Результат:

```
web-app-bf4cdf585-jvc6z   2/2   Running
```

[Манифест deployment.yaml](deployment.yaml)

---

### 2. Подключить веб-страницу через ConfigMap

Создан ConfigMap `web-content`, содержащий файл `index.html`.

Проверка:

```
kubectl get configmap web-content
```

Результат:

```
NAME          DATA
web-content   1
```

ConfigMap монтируется в контейнер nginx в каталог:

```
/usr/share/nginx/html
```

[Манифест configmap-web.yaml](configmap-web.yaml)

---

### 3. Проверить доступность веб-страницы

Для проверки выполнен проброс порта:

```
kubectl port-forward deployment/web-app 8080:80
```

В другом терминале выполнен запрос:

```
curl http://localhost:8080
```

Получен ответ:

```
<!DOCTYPE html>
<html>
<head>
  <title>Netology Kubernetes</title>
</head>
<body>
  <h1>Привет от Kubernetes ConfigMap!</h1>
  <p>Домашнее задание 2.3</p>
</body>
</html>
```

<img width="1429" height="584" alt="image" src="https://github.com/user-attachments/assets/f380ebc7-e3cc-4fdc-b79e-d24e53bb21f0" />

**Результат:** веб-страница успешно передана в контейнер nginx через ConfigMap и доступна по HTTP.

---

# Задание 2. Настройка HTTPS с Secrets

### 1. Сгенерировать SSL-сертификат

Для HTTPS был создан самоподписанный сертификат:

```
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=netology.local/O=Netology"
```

---

### 2. Создать TLS Secret

TLS Secret создан командой:

```
kubectl create secret tls netology-tls \
  --cert=tls.crt \
  --key=tls.key
```

Проверка:

```
kubectl get secret netology-tls
```

Результат:

```
NAME           TYPE                DATA
netology-tls   kubernetes.io/tls   2
```

Для хранения структуры Secret в репозитории используется шаблон:

[Манифест secret-tls.yaml](secret-tls.yaml)

---

### 3. Создать Service и настроить Ingress

Для приложения создан Service `web-app-svc`.

[Манифест service-web.yaml](service-web.yaml)

Создан Ingress `web-app-ingress` с доменным именем:

```
netology.local
```

и TLS Secret:

```
netology-tls
```

Проверка:

```
kubectl get ingress
kubectl describe ingress web-app-ingress
```

Ingress успешно использует TLS:

```
TLS:
  netology-tls terminates netology.local
```

Маршрут:

```
netology.local / → web-app-svc:80
```

[Манифест ingress-tls.yaml](ingress-tls.yaml)

<img width="519" height="331" alt="image" src="https://github.com/user-attachments/assets/a83ca221-4c91-48ad-9f84-6fb4b92c6006" />

---

### 4. Проверить HTTPS-доступ

Для проверки выполнена команда:

```
curl -k --resolve netology.local:443:10.0.2.15 https://netology.local/
```

Получен ответ:

```
<!DOCTYPE html>
<html>
<head>
  <title>Netology Kubernetes</title>
</head>
<body>
  <h1>Привет от Kubernetes ConfigMap!</h1>
  <p>Домашнее задание 2.3</p>
</body>
</html>
```

Параметр `-k` используется потому, что сертификат самоподписанный.

<img width="744" height="171" alt="image" src="https://github.com/user-attachments/assets/7e51d066-6cd7-4c54-b92d-7ca6fa9b83ef" />

**Результат:** приложение доступно по HTTPS через Ingress с использованием TLS Secret.

---

# Задание 3. Настройка RBAC

### 1. Включить RBAC

RBAC в MicroK8s включён командой:

```
microk8s enable rbac
```

---

### 2. Создать сертификат пользователя

Создан пользователь Kubernetes `developer`.

Приватный ключ:

```
openssl genrsa -out developer.key 2048
```

Запрос на сертификат:

```
openssl req -new \
  -key developer.key \
  -out developer.csr \
  -subj "/CN=developer"
```

Подписание сертификата CA Kubernetes:

```
sudo openssl x509 -req \
  -in developer.csr \
  -CA /var/snap/microk8s/current/certs/ca.crt \
  -CAkey /var/snap/microk8s/current/certs/ca.key \
  -CAcreateserial \
  -out developer.crt \
  -days 365
```

Получены файлы:

```
developer.key
developer.csr
developer.crt
```

---

### 3. Создать Role и RoleBinding

Создана Role `pod-reader`, которая разрешает:

- просмотр Pod;
- получение списка Pod;
- наблюдение за Pod;
- просмотр логов Pod.

[Манифест role-pod-reader.yaml](role-pod-reader.yaml)

Также создан RoleBinding `developer-pod-reader`, связывающий пользователя `developer` с Role `pod-reader`.

[Манифест rolebinding-developer.yaml](rolebinding-developer.yaml)

Проверка:

```
kubectl get role pod-reader
kubectl get rolebinding developer-pod-reader
```

Результат:

```
pod-reader
developer-pod-reader   Role/pod-reader
```

---

### 4. Проверить права пользователя

Проверены разрешённые операции:

```
kubectl auth can-i get pods --as=developer
kubectl auth can-i list pods --as=developer
kubectl auth can-i get pods/log --as=developer
```

Результат:

```
yes
yes
yes
```

Также проверена запрещённая операция:

```
kubectl auth can-i delete pods --as=developer
```

Результат:

```
no
```

Таким образом, пользователь может просматривать Pod и их логи, но не может удалять Pod.

Дополнительно выполнено:

```
kubectl get pods --as=developer
```

Команда успешно вывела список Pod:

```
web-app-bf4cdf585-jvc6z   2/2   Running
...
```

<img width="600" height="334" alt="image" src="https://github.com/user-attachments/assets/56ee21d1-e304-4246-bfc4-69f8fc6b626a" />

**Результат:** пользователь `developer` получил ограниченный доступ только для просмотра Pod и их логов, без возможности удаления ресурсов.
