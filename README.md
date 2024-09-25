# Nginx 사용법 정리

이 레포지토리는 CICD를 연습하며 만든 저장소입니다.

제가 개발한 웹 서비스의 프론트엔드 배포 구조에 대해서 리뷰해 보도록 하겠습니다. 내용은 AWS EC2를 활용하여 Nginx를 설치, Github Actions를 통해 HTML CSS JS 리소스들을 EC2로 전송, Nginx 설정 파일을 작성하여 엔드포인트 설정으로 구성되어있습니다. 

---

## 사용방법

1. 인스턴스를 대여
   대여 후, 인증키와 public ip를 보관해주세요.


2. EC2에 Nginx 설치
   EC2에 접속 후, 아래의 커맨드를 한 번에 실행시키면 쉽게 설치할 수 있습니다.
   
````
sudo apt update
sudo apt install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
````


3. Github Actions로 리소스 배포
    Github 리포지토리의 .github/workflows 디렉토리에 YAML 파일을 생성합니다. 아래는 예시입니다.

<details>
   <summary>예시 workflow</summary>

````
name: Deploy to Nginx

on:
  workflow_dispatch: ## 수동 트리거
  pull_request:
    branches: [ "main" ]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Add EC2 Host Key
        env:
          host: ${{ secrets.REMOTE_SERVER }}
          PEM_KEY: ${{ secrets.PEM_KEY}}
        run: |
          mkdir -p ~/.ssh
          echo "$PEM_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -H $host >> ~/.ssh/known_hosts  

      # SCP로 서버로 전송
      - name: Deploy to server
        env:
          PEM_KEY: ${{ secrets.PEM_KEY }}  # PEM 키를 GitHub Secrets에 저장
          username: ${{ secrets.SSH_USER }}
          host: ${{ secrets.REMOTE_SERVER }}
          key: ${{ secrets.PEM_KEY }}
          target_path: ${{ secrets.target_path }}
        run: |
          echo "${{ secrets.PEM_KEY }}" > private_key.pem
          chmod 600 private_key.pem
          scp -i private_key.pem -r src/main/* $username@$host:$target_path

      - name: Nginx restart
        env:
          PEM_KEY: ${{ secrets.PEM_KEY }}  # PEM 키를 GitHub Secrets에 저장
          username: ${{ secrets.SSH_USER }}
          host: ${{ secrets.REMOTE_SERVER }}
          key: ${{ secrets.PEM_KEY }}
        run: |
          echo "${{ secrets.PEM_KEY }}" > private_key.pem
          chmod 600 private_key.pem
          
          ssh -i private_key.pem "$username@$host" "
            sudo systemctl restart nginx
          "
          echo 'key를 제거합니다.'
          ssh-keygen -R $host
````
</details>

4. Nginx 설정파일 수정
   Nginx 설정파일은 보통 **/etc/nginx/sites-enables 디렉토리의 default 파일**입니다. 이 파일에 어떤 요청에 대해서 어떤 리소스를 반환할 것인지 설정합니다. 예를 들어, / 요청에 대하여 home.html을, /monitoring 요청에 대하여 monitoring.html을 반환한다고 가정하겠습니다. 아래는 예시입니다. **자신이 받을 요청과 요청에 대해서 반환할 리소스 디렉토리 구조와 파일명에 맞게 root의 경로와 index의 파일명을 변경해주세요.** **반드시, js 파일과 css 파일도 설정해야합니다.** 그렇지 않으면 html만 반환이 됩니다.

<details>
   <summary>예시 nginx 설정 파일</summary>

````
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        index index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                root /home/ubuntu/resources/templates;
                index home.html;
                try_files $uri $uri/ =404;
        }

        location /monitor {
                root /home/ubuntu/resources/templates;
                index monitor.html;
                try_files $uri $uri/ =404;
        }

        location /message {
                root /home/ubuntu/resources/templates;
                index message.html;
                try_files $uri $uri/ =404;
        }

        location /css/ {
                alias /home/ubuntu/resources/static/css/;
        }

        location /js/ {
                alias /home/ubuntu/resources/static/js/;
        }
}
````
</details>
   
4. Nginx 사용자 추가
  /etc/ngnix에 설정 파일에서 user 란에 ubuntu를 추가하여 static files에 ngnix가 접근할 수 있도록 허용하였다.
