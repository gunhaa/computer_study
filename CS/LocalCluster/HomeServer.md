# Home Server 외부 공개 SOP(N100)

1. N100에서 Application 실행
- ssh targetUser@192.168.0.x
- scp . targetUser@192.168.0.x
- bash start.sh
2. 내부망에서 접속해 작동 확인
3. 공유기 IP 고정
4. DDNS 등록
- `{id}.iptime.org`
5. 포트포워딩 설정
```plaintext
순위: 1
규칙 이름: lostark-mcp
규칙 종류: 포트포워드 사용자 정의
프로토콜: TCP
외부 포트: {$외부 사용 포트} ~ {$외부 사용 포트}
내부 IP주소: 192.168.0.20
내부 포트: {$내부 인스턴스 포트} ~ {$내부 인스턴스 포트}
```
6. 도메인의 `CNAME` 설정한 ddns로 변경


## ngrok 사용시

- `ngrok` 사용
```bash
# 공식 GPG 키 및 레포지토리 추가
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list

# 패키지 목록 업데이트 및 설치
sudo apt update && sudo apt install ngrok

ngrok config add-authtoken {$authToken}

ngrok http --url=${url}$ ${port}
```