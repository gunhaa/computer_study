# N100 Scripts SOP

## Docker(+mysql, redis image) 설치

```shell
# 1. 시스템 패키지 목록 최신화 및 업데이트
sudo apt update && sudo apt upgrade -y

# 2. 도커 공식 자동 설치 스크립트 다운로드 및 실행 후 삭제
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
rm get-docker.sh

# 3. 최신 Docker Compose V2 플러그인 확실하게 추가 설치
sudo apt install -y docker-compose-plugin

# 4. 현재 사용자($USER)에게 도커 권한 부여
sudo usermod -aG docker $USER

# 5. 변경된 권한을 터미널 세션에 즉시 적용
newgrp docker

# 6. 설치 완료 테스트
docker --version
docker compose version

# 7. 필요 이미지 설치
docker pull mysql:8.0
docker pull redis:7.0-alpine

# 8. 확인 
docker images
```

