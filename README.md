# Ref
- https://github.com/docker/awesome-compose/tree/master/django
- https://github.com/pitimon/dockerswarm-inhoure

# Wakatime
- https://wakatime.com/@spcn05/projects/acotlctecj

# Revert proxy
- https://spcn05-django.xops.ipv9.me/

# Setup VM

1. สร้าง VM
    - Ubuntu 22.04
    - 2 core CPU
    - 2 GB of ram
    - 32 GB HDD

2. Clone VM to template

3. ทำการสร้าง VM จาก template มาทั้งหมด 3 node
    - manager
    - worker1
    - worker2

4. ทำการเปลี่ยนค่าชื่อ hostname ทุกโหนด
```
hostnamectl set-hostname "name"
```
5. รีเซ็ทค่า machine-id ทุกโหนดเพื่อไม่ให้ ip ชนกัน
```
cp /dev/null /etc/machine-id
rm /var/lib/dbus/machine-id
ln -s /etc/machine-id /var/lib/dbus/machine-id
```

# Docker swarm
1. ทำการกำหนดเครื่อง manager
```
docker swarm init
```
 - โดยจะได้คำสั่งและคีย์มาเพื่อนำไปสร้าง worker

2. นำคำสั่งที่ได้มาไปรันบนเครื่อง worker

3. ทำการติดตั้ง Portainer ทั้ง 3 node เพื่อใช้เป็นตัว monitoring และทำให้ง่ายต่อการจัดการ
```
curl -L https://downloads.portainer.io/ce2-17/portainer-agent-stack.yml -o portainer-agent-stack.yml
docker stack deploy -c portainer-agent-stack.yml portainer
```

# Traefik
1. ทำการแก้ไขไฟล์ host บนเครื่องคอมพิวเตอร์
    - Windows C:\Windows\System32\drivers\etc\hosts
    - Linux/Mac /etc/hosts
    โดยเพิ่ม
    ```
    172.31.1.xxx portainer.cpedemo.local
    172.31.1.xxx traefik.cpedemo.local
    172.31.1.xxx swarmpit.cpedemo.local
    ```
2. ทำการติดตั้ง traefik เพื่อใช้ในการ dashboard ดู http routers
    ```
    docker network create --driver=overlay traefik-public

    export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}') 
    echo $NODE_ID

    docker node update --label-add traefik-public.traefik-public-certificates=true $NODE_ID

    export EMAIL=<email>
    export DOMAIN=<domain>
    export USERNAME=admin
    export PASSWORD=<password>
    export HASHED_PASSWORD=$(openssl passwd -apr1 $PASSWORD)
    echo $HASHED_PASSWORD

    docker stack deploy -c traefik-host.yml traefik
    ```

3. ตรวจสอบการใช้งานโดยเข้าไปที่ https://traefik.cpedemo.local/

# Swarmpit
1. ทำการติดตั้ง swarmpit เพื่อใช้ในการ dashboard ดูรายละเอียดของ node

    ```
    export DOMAIN=swarmpit.cpedemo.local

    กำหนดค่า lable ให้กับ db-data
    export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')
    docker node update --label-add swarmpit.db-data=true $NODE_ID

    กำหนดค่า lable ให้กับ influx-data
    export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')
    docker node update --label-add swarmpit.influx-data=true $NODE_ID

    docker stack deploy -c swarmpit.yml swarmpit
    ```

2. ตรวจสอบการใช้งานโดยเข้าไปที่ http://swarmpit.cpedemo.local/

# Docker hub images
1. ทำการล็อกอินบัญชี docker ที่ node
    ```
    docker login
    ```

2. ตรวจสอบชื่อ images ทั้งหมดที่มีเพื่อใช้ในการ push ลง docker hub
    ```
    docker images
    ```

3. สร้าง image ใหม่โดยใช้ image ตัวเดิม
![image](https://user-images.githubusercontent.com/109062980/224478761-bcd976a2-af33-477a-bb35-fe123a4ab5b6.png)

    ```
    docker tag swarm02-web-django:latest centurynine/swarm02-web-django:0311
    ```

4. push image เข้าสู่ docker hub
    ```
    docker push centurynine/swarm02-web-django:0311
    ```
![image](https://user-images.githubusercontent.com/109062980/224478839-2b88fc0a-9143-424f-a7ef-5cb1347362cb.png)

# Portainer rev-proxy deploy stack
1. เข้าไปที่หน้าเว็บ https://portainer.ipv9.me

2. ทำการ Add stack โดยใช้ image ที่ได้ push ลงไปยัง docker
    ```
    version: '3.7'
    services:
        web-django:
            image: centurynine/swarm02-web-django:0311
            networks:
            - webproxy
            logging:
            driver: json-file
            options:
                "max-size": "10m"
                "max-file": "5"
            deploy: 
            replicas: 1 
            labels:
                - traefik.docker.network=webproxy
                - traefik.enable=true
                - traefik.http.routers.${APPNAME}-https.entrypoints=websecure
                - traefik.http.routers.${APPNAME}-https.rule=Host("${APPNAME}.xops.ipv9.me")
                - traefik.http.routers.${APPNAME}-https.tls.certresolver=default
                - traefik.http.services.${APPNAME}.loadbalancer.server.port=8000
            
    volumes:
    app:

    networks:
    webproxy:
        external: true
    ```

3. กำหนด environment variables 
    - APPNAME = spcn05-django
![image](https://user-images.githubusercontent.com/109062980/224478785-52b07259-8a83-4f58-9258-54e95152a509.png)

4. ทำการ deploy stack
![image](https://user-images.githubusercontent.com/109062980/224479246-3c6a35a3-8f92-45da-b95d-cd70bc8a5873.png)

5. ตรวจสอบว่าทำการ deply ได้ไหมที่ลิ้งค์ https://spcn05-django.xops.ipv9.me/
