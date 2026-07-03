### EC2 서버 생성 외부연결한후
### 서버 패키지 및 디렉터리 구성
```
# 서버 패키지 업데이트 및 빌드 도구 설치
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3-pip python3-venv python3-dev git build-essential libpq-dev nginx

# 프로젝트 디렉토리 생성 및 가상환경 활성화
mkdir ~/myproject && cd ~/myproject
python3 -m venv venv
source venv/bin/activate

# 필수 패키지 임시 설치 및 최초 프로젝트 생성
pip install django gunicorn
django-admin startproject config .
python manage.py migrate

# settings.py의 ALLOWED_HOSTS에 EC2 퍼블릭 IP 등록 (최초 1회 편집)
nano config/settings.py
# ALLOWED_HOSTS = ['<EC2 퍼블릭 IP>', 'localhost', '127.0.0.1']
```

### Gunicorn 서비스 생성 (/etc/systemd/system/gunicorn.service)
```
sudo nano /etc/systemd/system/gunicorn.service
```
아래 내용 입력
```
[Unit]
Description=gunicorn daemon for Django Chatbot
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/myproject
ExecStart=/home/ubuntu/myproject/venv/bin/gunicorn \
          --workers 3 \
          --bind unix:/home/ubuntu/myproject/gunicorn.sock \
          --timeout 120 \
          config.wsgi:application

[Install]
WantedBy=multi-user.target
```
### Nginx 리버스 프록시 설정 (/etc/nginx/sites-available/myproject)
```
sudo nano /etc/nginx/sites-available/myproject
```
아래 설정을 입력 포트80(HTTP) 요청을 Gunicorn 소켁으로 흐르게 처리
```
server {
    listen 80;
    server_name <EC2 퍼블릭 IP 또는 도메인>;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        alias /home/ubuntu/myproject/static/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/ubuntu/myproject/gunicorn.sock;
        proxy_read_timeout 120s;
        proxy_connect_timeout 120s;
    }
}
```
### 반영 및 서비스 활성화
```
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo chmod 755 /home/ubuntu
sudo systemctl daemon-reload
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
sudo systemctl restart nginx
```
### CI/CD 구축 Gihub Actions 자동 배포
### git 초기화 및 최초 push
```
# EC2 터미널에서 실행
cd ~/myproject
git init
git remote add origin https://github.com/<본인_GitHub_ID>/django-chatbot-preview.git

# requirements.txt 파일 생성
pip freeze > requirements.txt

# 민감 정보 보호를 위해 .gitignore 작성
echo ".env" >> .gitignore
echo "venv/" >> .gitignore
echo "__pycache__/" >> .gitignore

# 커밋 및 최초 푸시
git add .
git commit -m "Initial skeleton setup"
git push -u origin main
```
### Github Secrets 등록
github-settings - Secrets and variables - actions - new repository secret 접속
```
EC2_HOST: EC2 퍼블릭 IP 주소
EC2_USER: ubuntu
EC2_SSH_KEY: EC2 접속 시 사용하는 .pem 키 내용 전체 복사 (메모장으로 열어 첫 줄부터 끝 줄까지 그대로 복사)
```
### 로컬pc로 프로젝트 복사(clone)

### 배포 워크플로우 파일 생성
로컬(원격) pc 프로젝트 루트 .github/workflows/deploy.yml 파일생성
```
name: Deploy Chatbot to EC2

on:
  push:
    branches: [ main ] # main 브랜치로 push 발생 시 배포 구동

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: SSH Connection and Auto Deploy
        uses: appleboy/ssh-action@v1.2.2
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd ~/myproject
            source venv/bin/activate
            git pull origin main
            pip install -r requirements.txt
            python manage.py migrate --noinput
            python manage.py collectstatic --noinput
            sudo systemctl restart gunicorn
```
첫번째 자동 배포 테스트 
```
git add .
git commit -m "add actions first"
git push origin main
```
github의 actions탭에서 초록색 체크마크 확인

