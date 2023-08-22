### 도커설치
	1. sudo apt-get update
    2. sudo apt-get install ca-certificates curl gnupg
    3. sudo install -m 0755 -d /etc/apt/keyrings
    4. curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    5. sudo chmod a+r /etc/apt/keyrings/docker.gpg
    6. echo \
        "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
        "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
        sudo tee /etc/apt/sources.list.d/docker.list > /dev/null     
    7. sudo apt-get update     
    8. sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    9. docker -v

### 도커명령(자주씀)

    1. docker cp (로컬 파일을 컨테이너 파일로 복사)
    2. docker inspect (자세한 내용확인)
    3. watch -n 1 docker ps -a (터미널에서 변경사항 수시확인)
    4. docker run --restart (재시작 정책설정)
    5. docker stats (서버자원 사용량 확인)
    6. docker run -it -d -v --name 이름 --restart=설정 컨테이너이름  

### 베이그란트로 손쉽게 초기 세팅 해보기

    # -*- mode: ruby -*-
    # vi: set ft=ruby :
    Vagrant.configure("2") do |config|
    config.vm.box = "ubuntu/focal64"

    config.vm.define "ubuntu" do |ubuntu|
        ubuntu.vm.hostname = "mz-server"
        ubuntu.vm.provider "virtualbox" do |vb|
        vb.name = "docker-server"
        vb.cpus = 4
        vb.memory = 8192
        end
        ubuntu.vm.network "forwarded_port", guest: 80, host: 80
        ubuntu.vm.network "private_network", ip: "192.168.33.10"
        ubuntu.vm.provision "shell", inline: <<-SCRIPT
        sudo apt-get update -y
        sudo apt-get install -y ca-certificates curl gnupg
        sudo install -m 0755 -d /etc/apt/keyrings
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
        sudo chmod a+r /etc/apt/keyrings/docker.gpg
        echo \
            "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
            "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
            sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
        sudo apt-get update -y
        sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
        sudo usermod -a -G docker vagrant
        docker run -it -d -p 80:80 --name nginx-server nginx
        docker cp /vagrant/env/sample2/ nginx-server:/usr/share/nginx/html

### 도커허브 로그인 토큰 등록
    1.env/docker_token 파일을 생성하여 토큰번호를 등록 합시다       

### 도커 스토리지 관리 해보자
    1.바인드마운트 호스트 파일시스템에서 직접 관리(내가 지정)
    2.볼륨마운트 도커 시스템에서 간접 관리(/var/lib/docker/volumes)
    3.tmpfs 메모리에만 저장, 호스트에 저장안함,성능이 좋고 빠름(시스템 메모리)
    4.볼륨마운트 커맨드
    5.볼륨 크리에이트 하고 컨테이너에 마운트하기
    6.docker run -d -p (8080:80) -v web_src_vol:/usr/local/apache2/htdocs --name 컨테이너이름 이미지(httpd)
