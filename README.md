# dockerSwarm

dockerswarm은 k8s와 k3s들과 비교했을 때 아키텍처의 간단함으로 배우기 쉽고, 적은 리소스 사용등의 이유로 사용하고자 합니다.<br>
dockerswarm은 docker 자체 오케스트레이션 도구입니다.<br>
dockerswarm으로 보다 쉽게 배포 환경에 대해 이해하고자 합니다.<br>
rolling, blue-green, canary 의 배포 전략을 구현해 보았습니다.<br>

## 배포 전략
### rolling
Rolling 배포는 새로운 버전의 애플리케이션을 점진적으로 배포하면서, 이전 버전의 인스턴스를 차례차례 새로운 버전으로 교체하는 방식입니다.<br>

신버전에 문제가 있을시 rollback이 힘들다는 단점이 있습니다.

### blue-green
blue 구버전 green 신버전으로 리소스를 2배로 사용하여 신버전이 준비 될때까지 구버전을 배포하며 신버전이 완료될 경우 해당 서비스를 제공하는 전략입니다.<br>
blue를 삭제하지 않는 한 rollback에 강한 장점을 지니고 있습니다. 다만 리소스를 2배로 사용하므로 자원 낭비가 심하다는 단점이 있습니다.


### canary
canary전략은 비율을 나누어 특정 소수(랜덤, 특정 ip유저들..)에게 신버전의 서비스를 제공한 후 성능의 보장이 될 경우 점진적으로 서비스를 늘려가는 전략입니다.<br>
위험성이 다른 전략에 비해 현저히 낮지만 소수 사용자의 피드백에 의존해야 한다는 단점을 지니고 있습니다.


## 실습

### image build
#### pyapp
일전에 사용했던 python-light-web을 재사용하였습니다.

#### nginx
```bash
/etc/nginx/nginx.conf
```
```bash
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
        upstream flask_app {
                server 10.0.2.22:5000 max_fails=2 fail_timeout=10s; 
                server 10.0.2.23:5000 max_fails=2 fail_timeout=10s;
        }

        server {
                listen 80;

                location / {
                        proxy_pass http://flask_app;
                        proxy_set_header Host $host;
                        proxy_set_header X-Real-IP $remote_addr;
                        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                        proxy_set_header X-Forwarded-Proto $scheme;

                }

                location ~* \.(html|css|js|png|jpg|jpeg|gif|ico)$ {
                        return 404;
                }
        }
}

```

사용중인 컨테이너 build
```bash
docker build commit <컨테이너ID> <저장할 이미지명>
```

사용할 이미지들
```bash
username@managerN:~$ docker images
REPOSITORY            TAG         IMAGE ID       CREATED        SIZE
dkac0012/pyapp        v1          9f4e1516330a   34 hours ago   175MB
dkac0012/nginx        v2          dc99173a2160   34 hours ago   294MB
dkac0012/pyapp        v2          c8d9e516e8aa   4 days ago     175MB
```


### docker swarm 

manager Node 등록
```bash
username@managerN:~$ docker swarm init
Swarm initialized: current node is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token "토큰 값" ip:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

worker Node 등록
```bash
  docker swarm join --token "토큰 값" ip:2377
```

service 등록
```bash
pyapp
docker service create --name pyapp --replicas 4 --constraint "node.hostname != workerN3" --publish 5000:5000 --update-delay 10s --update-parallelism 1 --restart-condition any dkac0012/pyapp:v1

nginx
docker service create --name nginx --replicas 2 --constraint "node.hostname == workerN3" --publish 80:80 --restart-condition any dkac0012/nginx:v2
```

service 확인
```bash
username@managerN:~$ docker service ls
ID             NAME      MODE         REPLICAS   IMAGE               PORTS
ok2nloiiq3mf   nginx     replicated   2/2        dkac0012/nginx:v2   *:80->80/tcp
mqqrbg5ty60a   pyapp     replicated   4/4        dkac0012/pyapp:v1   *:5000->5000/tcp
```

### 배포

#### rolling 

```bash
username@managerN:~$ docker service update --image dkac0012/pyapp:v2 pyapp

위에 서비스를 생성하며 넣어두었던 --update-delay 10s --update-parallelism 1 옵션으로 10초를 간격으로 1개씩 업데이트가 된다. 이 과정에서 서비스의 신버전과 구버전이 동시에 제공된다.
```

#### blue-green

```bash
username@managerN:~$ docker service create --name green-pyapp --replicas 4 --constraint "node.hostname != workerN3" --publish 5001:5001 --update-delay 10s --update-parallelism 1 --restart-condition any dkac0012/pyapp:v2
```
다음 커맨드를 활용해 서비스를 하나 더 띄워두고 nginx의 설정파일의 proxy_pass 부분을 다음과 같이 고칩니다.
```bash
 proxy_pass http://green-pyapp;
```

<br>

**버전 2 배포 성공 화면** <br><br>

![image](https://github.com/user-attachments/assets/7b164f73-d911-4782-aa08-a2b24b81f47f)


#### canary

```bash
username@managerN:~$ docker service create --name canary-pyapp --replicas 4 --constraint "node.hostname != workerN3" --publish 5001:5001 --update-delay 10s --update-parallelism 1 --restart-condition any dkac0012/pyapp:v2
```

위와 같이 canary 버전을 소규모로 배포한다. 버전에 대한 성능이 확신될때 마다 비중을 규모에 맞춰 바꿔가면 된다.
```bash
docker service update --replicas 3 canary-pyapp
or
docker service scale canary-pyapp=3
```

nginx 설정
가중치를 넣어 특정 랜덤유저들은 canary서비스를 이용하도록 한다.
```bash
    upstream flask_app {
        server pyapp:5000 weight=90;
        server canary-pyapp:5000 weight=10;
    }
```
