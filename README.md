# Домашнее задание к занятию «Helm» - Kadancev Vladimir

### Цель задания

В тестовой среде Kubernetes необходимо установить и обновить приложения с помощью Helm.

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение, например, MicroK8S.
2. Установленный локальный kubectl.
3. Установленный локальный Helm.
4. Редактор YAML-файлов с подключенным репозиторием GitHub.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Инструкция](https://helm.sh/docs/intro/install/) по установке Helm. [Helm completion](https://helm.sh/docs/helm/helm_completion/).

------

### Задание 1. Подготовить Helm-чарт для приложения

1. Необходимо упаковать приложение в чарт для деплоя в разные окружения. 
2. Каждый компонент приложения деплоится отдельным deployment’ом или statefulset’ом.
3. В переменных чарта измените образ приложения для изменения версии.

## Решение:

Чарт [myapp](myapp/) упаковывает приложение из двух компонентов. Каждый компонент деплоится отдельным Deployment и имеет свой Service:

| Компонент | Образ по умолчанию | Ресурсы |
|-----------|--------------------|---------|
| `frontend` | `nginx:alpine` | Deployment + Service (порт 80) |
| `backend` | `wbitt/network-multitool:minimal` | Deployment + Service (порт 8080) |

Файлы чарта:

- [myapp/Chart.yaml](myapp/Chart.yaml): описание чарта (`version: 0.1.0`, `appVersion: "1.0.0"`);
- [myapp/values.yaml](myapp/values.yaml): переменные чарта;
- [myapp/templates/_helpers.tpl](myapp/templates/_helpers.tpl): общие метки;
- [myapp/templates/frontend-deployment.yaml](myapp/templates/frontend-deployment.yaml), [myapp/templates/frontend-service.yaml](myapp/templates/frontend-service.yaml);
- [myapp/templates/backend-deployment.yaml](myapp/templates/backend-deployment.yaml), [myapp/templates/backend-service.yaml](myapp/templates/backend-service.yaml).

### Как устроен чарт

- **Образ и версия вынесены в переменные.** В `values.yaml` для каждого компонента заданы `image.repository`, `image.tag` и `image.pullPolicy`.
  Версия приложения меняется переменной `frontend.image.tag` (при установке через `--set` или в своём файле values).

  ```yaml
  frontend:
    replicaCount: 1
    image:
      repository: nginx
      tag: alpine
      pullPolicy: IfNotPresent
    service:
      port: 80

  backend:
    replicaCount: 1
    image:
      repository: wbitt/network-multitool
      tag: minimal
      pullPolicy: IfNotPresent
    port: 8080
    service:
      port: 8080
  ```

- **Несколько копий не конфликтуют.** Имена ресурсов начинаются с имени релиза (`{{ .Release.Name }}-frontend`), а в селекторах Deployment и Service
  есть метки `app.kubernetes.io/instance` (имя релиза) и `app.kubernetes.io/component`. Поэтому два релиза в одном namespace
  не пересекаются по именам и не подхватывают чужие поды.
- Backend слушает порт, заданный `backend.port` (через переменную окружения `HTTP_PORT`).

### Проверка чарта без установки

```bash
find myapp -type f | sort
helm lint myapp
helm template test myapp | head -40
helm template test myapp --set frontend.image.tag=1.27-alpine | grep image:
```

Структура чарта, `helm lint` и начало отрисованных манифестов:

![структура, lint, template](img/task1-lint-template.png)

Смена переменной `frontend.image.tag` меняет образ в отрисованных манифестах (изменение версии через переменные):

![helm template --set](img/task1-template-set.png)

------
### Задание 2. Запустить две версии в разных неймспейсах

1. Подготовив чарт, необходимо его проверить. Запуститe несколько копий приложения.
2. Одну версию в namespace=app1, вторую версию в том же неймспейсе, третью версию в namespace=app2.
3. Продемонстрируйте результат.

## Решение:

Три релиза одного чарта: две версии в namespace `app1` и третья в namespace `app2`.
Релизы называются по-разному, поэтому в одном namespace они не конфликтуют.

```bash
helm install app-v1 ./myapp -n app1 --create-namespace --set frontend.image.tag=1.26-alpine
helm install app-v2 ./myapp -n app1 --set frontend.image.tag=1.27-alpine
helm install app-v3 ./myapp -n app2 --create-namespace --set frontend.image.tag=alpine
```

| Релиз | Namespace | Образ frontend |
|-------|-----------|----------------|
| `app-v1` | `app1` | `nginx:1.26-alpine` |
| `app-v2` | `app1` | `nginx:1.27-alpine` |
| `app-v3` | `app2` | `nginx:alpine` |

Установка трёх релизов:

![helm install](img/task2-install.png)

### Результат

```bash
helm list -A
kubectl get pods -n app1
kubectl get pods -n app2
```

В `app1` работают поды двух релизов (`app-v1-*` и `app-v2-*`), в `app2` работают поды `app-v3-*`.
В списке релизов также видны `traefik` и `kubernetes-dashboard`: это аддоны MicroK8s, установленные ранее.

![helm list и поды](img/task2-list-pods.png)

Образы и сервисы. В колонке `IMAGES` видны разные версии frontend:

```bash
kubectl get deploy -n app1 -o wide
kubectl get deploy -n app2 -o wide
kubectl get svc -n app1
kubectl get svc -n app2
```

![deployments и services](img/task2-deploy-svc.png)

Проверка работы приложения и применённых значений. Запрос из временного пода в `app1` к Service `app-v1-frontend`
возвращает `HTTP/1.1 200 OK` с заголовком `Server: nginx/1.26.3`, что соответствует тегу `1.26-alpine`:

```bash
kubectl run curl-test -n app1 --image=wbitt/network-multitool:minimal --image-pull-policy=IfNotPresent --rm -it --restart=Never -- curl -sI app-v1-frontend
helm get values app-v1 -n app1
```

![curl и helm get values](img/task2-curl-values.png)
### Правила приёма работы

1. Домашняя работа оформляется в своём Git репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl`, `helm`, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.
