# Container-mysql-data-Dump
컨테이너의 Mysql 데이터를 덤프화 및 자동화

### 개요 🚩
컨테이너에서 MySQL 데이터 덤프를 자동화하고 **새로운 데이터베이스 컨테이너에 반영**하고자 한다. 이때 `docker-compose`의 `volume` 기능을 사용 한다.

<h3>1. docker-compose.yml 파일 수정</h3>

먼저 백업을 위해 `volume을` 추가하고, 새로운 데이터 컨테이너 `newmysqldb` 추가를 위해 `docker-compose.yml`을 수정해준다.

```yaml
services:
  db:
    container_name: mysqldb
    image: mysql:latest
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: # password 입력
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    networks:
      - spring-mysql-net
    volumes:
      - mysqldata:/var/lib/mysql
      - ./backup:/backup  # 백업 디렉터리 공유
    healthcheck:
      test: ['CMD-SHELL', 'mysqladmin ping -h 127.0.0.1 -u root --password=$$MYSQL_ROOT_PASSWORD']
      interval: 10s
      timeout: 2s
      retries: 100

  app:
    container_name: springbootapp
    build:
      context: .
      dockerfile: ./Dockerfile
    ports:
      - "8088:8088"
    environment:
      MYSQL_HOST: db  # DB 호스트 설정
    depends_on:
      db:
        condition: service_healthy
    networks:
      - spring-mysql-net

  newmysqldb:
    container_name: newmysqldb
    image: mysql:latest
    environment:
      MYSQL_ROOT_PASSWORD: # password 입력
    volumes:
      - newmysqldata:/var/lib/mysql
      - ./backup:/backup  # 동일한 백업 디렉터리 공유
    networks:
      - spring-mysql-net

networks:
  spring-mysql-net:
    driver: bridge

volumes:
  mysqldata:
  newmysqldata:
```

**db**와 **newmysqldb** 서비스에서 `/backup` 디렉토리를 공유하는 볼륨이 추가되었다. 이를 통해 MySQL 백업 파일 공유 가능하다. <br>
**newmysqldb** 서비스는 새로 생성된 데이터베이스 컨테이너로, 덤프 파일을 이 컨테이너에 반영할 수 있다.

<h3>2. mysql 데이터 덤프 및 newmysqldb에 반영</h3>

`mysqldb`에서 데이터 백업 및 덤프 생성 
```bash
docker exec mysqldb mysqldump -u root -p'root' --all-databases > ./backup/mysqldump.sql
```

<br>

생성된 폴더 및 sql 파일 확인
<div align="center">
  <img src ="https://github.com/user-attachments/assets/ac099c55-35f7-4f6e-9787-75829a1999d9" width ="50%">
</div>

<br>

`newmysqldb`에 덤프 파일 반영한다.
```bash
docker exec -i newmysqldb mysql -u root -p'root' < ./backup/mysqldump.sql
```

<br>

**※ 트러블 슈팅 🙄**

해당 bash가 Permission denied 되는것을 확인 <br>

<div align="center">
  <img src ="https://github.com/user-attachments/assets/3593a631-0d06-456c-89b6-1e6c9aa298c9" width ="70%">
</div>

이를 해결하기 위해 권한을 수정해 주었다.

```bash
# 백업 디렉토리 권한 설정
sudo chmod 777 ./backup

# 소유자 권한 변경
sudo chown -R $USER:$USER ./backup
```

### 결과 확인 👩‍💻

<div align="center">
  <img src ="https://github.com/user-attachments/assets/7b34f602-2e7e-43f4-8ef6-1e8215c56203" width ="55%">
  <img src ="https://github.com/user-attachments/assets/78314add-e13f-41a7-af5d-215307e00a88" width ="35%">
</div>


