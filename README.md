# Домашнее задание к занятию «Запуск приложений в K8S»

### Цель задания

В тестовой среде для работы с Kubernetes, установленной в предыдущем ДЗ, необходимо развернуть Deployment с приложением, состоящим из нескольких контейнеров, и масштабировать его.

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым git-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) Deployment и примеры манифестов.
2. [Описание](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) Init-контейнеров.
3. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod

1. Создать Deployment приложения, состоящего из двух контейнеров — nginx и multitool. Решить возникшую ошибку.
2. После запуска увеличить количество реплик работающего приложения до 2.
3. Продемонстрировать количество подов до и после масштабирования.
4. Создать Service, который обеспечит доступ до реплик приложений из п.1.
5. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложений из п.1.

------

### Ответы:

```
ubuntu@server-1:~/kuber$ micro apply -f one-deploy.yaml
deployment.apps/nginx created
ubuntu@server-1:~/kuber$ micro get pods
NAME                     READY   STATUS   RESTARTS     AGE
nginx-567dcfb566-fpnx7   1/2     Error    1 (8s ago)   17s
```
Ошибка связана с тем, что оба приложения слушают 80 порт. Для исправления ошибки я нашел 3 способа, поменять порт nginx в конфиге (не самый лучший, но я решил его описать), поменять порт multitool (думаю что это правильный способ в рамках этого задания), и 3й способ это сделать отдельные деплойменты с отдельными подами под каждое приложение. Третий способ логически правильный, но я его не использовал, т.к. в задании указано создать деплоймент с подом, состоящим из двух контейнеров.

### 1 способ:    

```
ubuntu@server-1:~/kuber$ micro exec -it nginx-567dcfb566-fpnx7 -c nginx -- sed -i 's/80/8080/' /etc/nginx/conf.d/default.conf
ubuntu@server-1:~/kuber$ micro exec -it nginx-567dcfb566-fpnx7 -c nginx -- nginx -s reload
2023/10/11 12:43:39 [notice] 43#43: signal process started
ubuntu@server-1:~/kuber$ micro get pods
NAME                     READY   STATUS    RESTARTS      AGE
nginx-567dcfb566-fpnx7   2/2     Running   4 (51s ago)   111s
```
Здесь был изменен порт nginx    
```
ubuntu@server-1:~/kuber$ micro scale deployment nginx --replicas=2
deployment.apps/nginx scaled
ubuntu@server-1:~/kuber$ micro get deploy
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/2     2            1           13m
ubuntu@server-1:~/kuber$ micro get pods
NAME                     READY   STATUS    RESTARTS      AGE
nginx-567dcfb566-fpnx7   2/2     Running   4 (12m ago)   13m
nginx-567dcfb566-9fg2q   2/2     Running   0             7s
```
Увеличено количество реплик до 2    
```
ubuntu@server-1:~/kuber$ micro exec -it nginx-567dcfb566-9fg2q -c nginx -- sed -i 's/80/8080/' /etc/nginx/conf.d/default.conf
ubuntu@server-1:~/kuber$ micro exec -it nginx-567dcfb566-9fg2q -c nginx -- nginx -s reload
2023/10/11 12:57:05 [notice] 39#39: signal process started
ubuntu@server-1:~/kuber$ micro get pods
NAME                     READY   STATUS    RESTARTS      AGE
nginx-567dcfb566-fpnx7   2/2     Running   4 (14m ago)   15m
nginx-567dcfb566-9fg2q   2/2     Running   4 (46s ago)   107s
```
Во второй реплике так же вручную поменян порт nginx. Естественно данный метод не подходит

### 2 способ:

В манифесте указываем переменную HTTP_PORT равную 8080, теперь multitool слушает 8080 и ошибки не будет изначально
```
env:
    - name: HTTP_PORT
    value: "8080"

```
```
ubuntu@server-1:~/kuber$ micro apply -f one-final-deploy.yaml
deployment.apps/nginx-final created
ubuntu@server-1:~/kuber$ micro get pods
NAME                           READY   STATUS              RESTARTS      AGE
nginx-567dcfb566-fpnx7         2/2     Running             4 (25h ago)   25h
nginx-567dcfb566-9fg2q         2/2     Running             4 (25h ago)   25h
mult                           1/1     Running             0             25h
nginx-init-9f6bf5c97-6zwps     1/1     Running             0             5h38m
nginx-final-756dcb66b6-ql66r   0/2     ContainerCreating   0             4s
ubuntu@server-1:~/kuber$ micro get deploy
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
nginx         2/2     2            2           25h
nginx-init    1/1     1            1           5h38m
nginx-final   1/1     1            1           14s
ubuntu@server-1:~/kuber$ micro get pods
NAME                           READY   STATUS    RESTARTS      AGE
nginx-567dcfb566-fpnx7         2/2     Running   4 (25h ago)   25h
nginx-567dcfb566-9fg2q         2/2     Running   4 (25h ago)   25h
mult                           1/1     Running   0             25h
nginx-init-9f6bf5c97-6zwps     1/1     Running   0             5h38m
nginx-final-756dcb66b6-ql66r   2/2     Running   0             30s

```
Увеличиваем количество реплик:    
```
ubuntu@server-1:~/kuber$ micro apply -f one-final-deploy.yaml
deployment.apps/nginx-final configured
ubuntu@server-1:~/kuber$ micro get pods
NAME                           READY   STATUS    RESTARTS      AGE
nginx-567dcfb566-fpnx7         2/2     Running   4 (25h ago)   26h
nginx-567dcfb566-9fg2q         2/2     Running   4 (25h ago)   25h
mult                           1/1     Running   0             25h
nginx-init-9f6bf5c97-6zwps     1/1     Running   0             5h40m
nginx-final-756dcb66b6-ql66r   2/2     Running   0             2m1s
nginx-final-756dcb66b6-fstss   2/2     Running   0             7s

```
Создан сервис:    
```
ubuntu@server-1:~/kuber$ micro get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.152.183.1     <none>        443/TCP   2d5h
nginx-svc    ClusterIP   10.152.183.130   <none>        80/TCP    25h

```

Создан отдельный под с multitool:    
```
ubuntu@server-1:~/kuber$ micro describe pods mult
Name:             mult
Namespace:        default
Priority:         0
Service Account:  default
Node:             server-1/10.0.3.17
Start Time:       Wed, 11 Oct 2023 13:04:38 +0000
Labels:           app=nginx
Annotations:      cni.projectcalico.org/containerID: fb3a486b5c253e074dfc645d43cc7da885ae3dbf0fdb400c71612d6564134b0e
                  cni.projectcalico.org/podIP: 10.1.125.224/32
                  cni.projectcalico.org/podIPs: 10.1.125.224/32
Status:           Running
IP:               10.1.125.224
IPs:
  IP:  10.1.125.224
Containers:
  mult:
    Container ID:   containerd://7a87440c8423fc421cbc6f988f5bae2f593d1958b89957e1a94b96f7ffe96b2a
    Image:          wbitt/network-multitool
    Image ID:       docker.io/wbitt/network-multitool@sha256:d1137e87af76ee15cd0b3d4c7e2fcd111ffbd510ccd0af076fc98dddfc50a735
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 11 Oct 2023 13:04:40 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-44bwg (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-44bwg:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>

```
Проверка доступности сервисов с отдельного пода:

```
ubuntu@server-1:~/kuber$ micro get pods -o wide
NAME                           READY   STATUS    RESTARTS      AGE     IP             NODE       NOMINATED NODE   READINESS GATES
nginx-567dcfb566-fpnx7         2/2     Running   4 (26h ago)   26h     10.1.125.222   server-1   <none>           <none>
nginx-567dcfb566-9fg2q         2/2     Running   4 (25h ago)   25h     10.1.125.223   server-1   <none>           <none>
mult                           1/1     Running   0             25h     10.1.125.224   server-1   <none>           <none>
nginx-init-9f6bf5c97-6zwps     1/1     Running   0             5h44m   10.1.125.230   server-1   <none>           <none>
nginx-final-756dcb66b6-ql66r   2/2     Running   0             6m24s   10.1.125.233   server-1   <none>           <none>
nginx-final-756dcb66b6-fstss   2/2     Running   0             4m30s   10.1.125.234   server-1   <none>           <none>
ubuntu@server-1:~/kuber$ micro exec mult -- curl 10.1.125.222:8080
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   615  100   615    0     0   2201      0 --:--:-- --:--:-- --:--:--  2204
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
ubuntu@server-1:~/kuber$ micro exec mult -- ping -c 4 10.1.125.223
PING 10.1.125.223 (10.1.125.223) 56(84) bytes of data.
64 bytes from 10.1.125.223: icmp_seq=1 ttl=63 time=0.275 ms
64 bytes from 10.1.125.223: icmp_seq=2 ttl=63 time=0.109 ms
64 bytes from 10.1.125.223: icmp_seq=3 ttl=63 time=0.134 ms
64 bytes from 10.1.125.223: icmp_seq=4 ttl=63 time=0.121 ms

--- 10.1.125.223 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3076ms
rtt min/avg/max/mdev = 0.109/0.159/0.275/0.067 ms
ubuntu@server-1:~/kuber$ micro exec mult -- curl 10.1.125.233
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   615  100   615    0     0   168k      0 --:--:-- --:--:-- --:--:--  200k
ubuntu@server-1:~/kuber$ micro exec mult -- ping -c 4 10.1.125.234
PING 10.1.125.234 (10.1.125.234) 56(84) bytes of data.
64 bytes from 10.1.125.234: icmp_seq=1 ttl=63 time=0.180 ms
64 bytes from 10.1.125.234: icmp_seq=2 ttl=63 time=0.117 ms
64 bytes from 10.1.125.234: icmp_seq=3 ttl=63 time=0.079 ms
64 bytes from 10.1.125.234: icmp_seq=4 ttl=63 time=0.125 ms

--- 10.1.125.234 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3061ms
rtt min/avg/max/mdev = 0.079/0.125/0.180/0.036 ms

```


------

### Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения.
2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox.
3. Создать и запустить Service. Убедиться, что Init запустился.
4. Продемонстрировать состояние пода до и после запуска сервиса.


### Ответы:

```
ubuntu@server-1:~/kuber$ micro get pods -w
NAME                           READY   STATUS     RESTARTS      AGE
nginx-567dcfb566-fpnx7         2/2     Running    4 (26h ago)   26h
nginx-567dcfb566-9fg2q         2/2     Running    4 (25h ago)   25h
mult                           1/1     Running    0             25h
nginx-final-756dcb66b6-ql66r   2/2     Running    0             12m
nginx-final-756dcb66b6-fstss   2/2     Running    0             10m
nginx-init-9f6bf5c97-dkvbb     0/1     Init:0/1   0             10s
nginx-init-9f6bf5c97-dkvbb     0/1     PodInitializing   0             32s
nginx-init-9f6bf5c97-dkvbb     1/1     Running           0             34s

ubuntu@server-1:~/kuber$ micro get pods
NAME                           READY   STATUS    RESTARTS      AGE
nginx-567dcfb566-fpnx7         2/2     Running   4 (26h ago)   26h
nginx-567dcfb566-9fg2q         2/2     Running   4 (25h ago)   26h
mult                           1/1     Running   0             25h
nginx-final-756dcb66b6-ql66r   2/2     Running   0             15m
nginx-final-756dcb66b6-fstss   2/2     Running   0             13m
nginx-init-9f6bf5c97-dkvbb     1/1     Running   0             3m11s

```
![alt text](https://github.com/bonanzza-web/kuber-homeworks1.3/blob/main/docs/2023-10-12_17-54-18.png)

Сервис пересоздавать не стал, т.к. есть с первого задания, под с инит контейнером доступен:
```
ubuntu@server-1:~/kuber$ micro get pods -o wide
NAME                           READY   STATUS    RESTARTS      AGE     IP             NODE       NOMINATED NODE   READINESS GATES
nginx-567dcfb566-fpnx7         2/2     Running   4 (26h ago)   26h     10.1.125.222   server-1   <none>           <none>
nginx-567dcfb566-9fg2q         2/2     Running   4 (26h ago)   26h     10.1.125.223   server-1   <none>           <none>
mult                           1/1     Running   0             25h     10.1.125.224   server-1   <none>           <none>
nginx-final-756dcb66b6-ql66r   2/2     Running   0             17m     10.1.125.233   server-1   <none>           <none>
nginx-final-756dcb66b6-fstss   2/2     Running   0             15m     10.1.125.234   server-1   <none>           <none>
nginx-init-9f6bf5c97-dkvbb     1/1     Running   0             5m12s   10.1.125.235   server-1   <none>           <none>
ubuntu@server-1:~/kuber$ micro exec mult -- ping -c 4 10.1.125.235
PING 10.1.125.235 (10.1.125.235) 56(84) bytes of data.
64 bytes from 10.1.125.235: icmp_seq=1 ttl=63 time=0.177 ms
64 bytes from 10.1.125.235: icmp_seq=2 ttl=63 time=0.097 ms
64 bytes from 10.1.125.235: icmp_seq=3 ttl=63 time=0.121 ms
64 bytes from 10.1.125.235: icmp_seq=4 ttl=63 time=0.109 ms

--- 10.1.125.235 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3055ms
rtt min/avg/max/mdev = 0.097/0.126/0.177/0.030 ms

```
Логи пода:
```
ubuntu@server-1:~/kuber$ micro logs nginx-init-9f6bf5c97-dkvbb
Defaulted container "nginx" out of: nginx, nginx-init-app (init)
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2023/10/12 14:53:27 [notice] 1#1: using the "epoll" event method
2023/10/12 14:53:27 [notice] 1#1: nginx/1.25.2
2023/10/12 14:53:27 [notice] 1#1: built by gcc 12.2.0 (Debian 12.2.0-14)
2023/10/12 14:53:27 [notice] 1#1: OS: Linux 5.15.0-86-generic
2023/10/12 14:53:27 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 65536:65536
2023/10/12 14:53:27 [notice] 1#1: start worker processes
2023/10/12 14:53:27 [notice] 1#1: start worker process 29
2023/10/12 14:53:27 [notice] 1#1: start worker process 30

```
Логи контейнеров в поде:
```
ubuntu@server-1:~/kuber$ micro logs nginx-init-9f6bf5c97-dkvbb -c nginx
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2023/10/12 14:53:27 [notice] 1#1: using the "epoll" event method
2023/10/12 14:53:27 [notice] 1#1: nginx/1.25.2
2023/10/12 14:53:27 [notice] 1#1: built by gcc 12.2.0 (Debian 12.2.0-14)
2023/10/12 14:53:27 [notice] 1#1: OS: Linux 5.15.0-86-generic
2023/10/12 14:53:27 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 65536:65536
2023/10/12 14:53:27 [notice] 1#1: start worker processes
2023/10/12 14:53:27 [notice] 1#1: start worker process 29
2023/10/12 14:53:27 [notice] 1#1: start worker process 30
ubuntu@server-1:~/kuber$ micro logs nginx-init-9f6bf5c97-dkvbb -c nginx-init-app
Hello world
Todo esta bien

```
Файлы находятся в каталоге (https://github.com/bonanzza-web/kuber-homeworks1.3/tree/main/docs)


------

### Правила приема работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать файлы манифестов и ссылки на них в файле README.md.

------
