# elk-security-monitoring
# 🔐 ELK Stack 기반 보안 모니터링 시스템

SSH 로그인 이벤트를 실시간으로 수집·분석하고, 이상 징후 발생 시 이메일로 자동 알림을 보내는 보안 모니터링 파이프라인입니다.

> 단국대학교 네트워크 시스템 및 보안 연구실 (BoanLab) 에서 진행한 개인 프로젝트입니다.

---

## 📌 주요 기능

- **로그 수집**: Filebeat로 Linux 시스템의 `/var/log/auth.log` 실시간 수집
- **로그 파싱**: Logstash Grok 필터로 SSH 로그인 성공/실패 이벤트 구조화
- **GeoIP 시각화**: 접속 IP의 국가/도시 정보를 Kibana 지도로 시각화
- **자동 알림**: 로그인 실패 감지 시 네이버 SMTP를 통해 이메일 자동 발송

---

## 🛠 기술 스택

| 구분 | 기술 |
|------|------|
| 로그 수집 | Filebeat 8.17.3 |
| 로그 처리 | Logstash 8.17.3 |
| 검색/저장 | Elasticsearch 8.17.3 |
| 시각화 | Kibana 8.17.3 |
| 알림 | Logstash Email Output (SMTP) |
| OS | Ubuntu (Linux) |

---

## 🏗 아키텍처

```
Linux 시스템 로그
(/var/log/auth.log)
        │
        ▼
   [ Filebeat ]         ← 로그 파일 실시간 모니터링
        │ (5044 포트)
        ▼
   [ Logstash ]         ← Grok 파싱 + GeoIP 변환 + 알림 트리거
        │
        ├──────────────────────────────────┐
        ▼                                  ▼
[ Elasticsearch ]               [ Email Alert ]
  (로그 저장/인덱싱)              (로그인 실패 시 발송)
        │
        ▼
   [ Kibana ]           ← 대시보드 시각화 (localhost:5601)
```

---

## ⚙️ 환경 설정

### 1. Elasticsearch 실행
```bash
cd ~/Desktop/elasticsearch/elasticsearch-8.17.3/bin
./elasticsearch
# 확인: http://localhost:9200
```

### 2. Kibana 실행
```bash
cd ~/Desktop/elasticsearch/kibana-8.17.3/bin
./kibana
# 확인: http://localhost:5601
```

### 3. Logstash 실행
```bash
cd ~/Desktop/elasticsearch/logstash-8.17.3/bin
./logstash -f config/logstash.conf
```

### 4. Filebeat 실행 (Linux)
```bash
sudo systemctl enable filebeat
sudo systemctl start filebeat
sudo systemctl status filebeat   # 상태 확인
```

---

## 📄 주요 설정 파일

### filebeat.yml
```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/auth.log
      - /var/log/syslog
    fields:
      log_type: authentication

output.logstash:
  hosts: ["localhost:5044"]
```

### logstash.conf
```ruby
input {
  # Filebeat에서 수신한 로그 (5044포트)
}

filter {
  grok {
    match => {
      "auth_message" => [
        "Accepted password for %{DATA:logname} from %{IP:src_ip} port %{NUMBER:port} %{GREEDYDATA}",
        "Failed password for %{DATA:logname} from %{IP:src_ip} port %{NUMBER:port} %{GREEDYDATA}"
      ]
    }
    overwrite => ["logname", "src_ip", "port"]
  }

  geoip {
    source => "src_ip"
    target => "geoip"
    database => "/usr/share/GeoIP/GeoLite2-City.mmdb"
  }
}

output {
  elasticsearch {
    pipelines:
      - pipeline: "auth_pipeline"
  }

  # 로그인 실패 시 이메일 알림
  if "Failed password" in [message] {
    email {
      to       => "your@email.com"
      from     => "your@email.com"
      subject  => "Login Failure Alert on %{host}"
      body     => "A failed login attempt detected.\nHost: %{hostname}\nMessage: %{auth_message}"
      address  => "smtp.naver.com"
      port     => 587
      use_tls  => true
    }
  }
}
```

---

## 🧪 로그인 실패 이벤트 테스트

SSH 로그인 실패를 10회 발생시켜 알림 동작을 테스트할 수 있습니다.

```bash
sudo apt install sshpass -y

for i in {1..10}; do
  sshpass -p "wrongpassword" ssh -o StrictHostKeyChecking=no testuser@<target_ip> exit
done
```

---

## 📊 Kibana 활용

Kibana의 **Stack Management > Data Views** 에서 인덱스 패턴을 등록하면 다음을 확인할 수 있습니다.

- **Table**: IP별 로그인 시도 횟수 및 실패율
- **Geo Map**: 접속 국가/도시 지도 시각화
- **Timeline**: 시간대별 로그인 시도 추이

인덱스 목록 예시:
| 인덱스 | 설명 |
|--------|------|
| `auth-logs` | SSH 인증 로그 |
| `utm-logs-test` | UTM 로그 테스트 |
| `filebeat` | Filebeat 수집 로그 |

---

## 🔧 트러블슈팅

**SMTP 인증 오류 발생 시**

네이버 메일 사용 시 일반 비밀번호로는 인증이 거부됩니다.

해결 방법:
1. 네이버 메일 환경설정 → POP3/SMTP → **사용함** 으로 변경
2. 네이버 보안설정 → **2단계 인증** 활성화
3. **애플리케이션 전용 비밀번호** 발급 후 logstash.conf의 `password` 항목에 적용

---

## 📚 참고 자료

- [Elastic 공식 문서](https://www.elastic.co/docs)
- [Filebeat 설치 가이드](https://www.elastic.co/beats/filebeat)
- [GeoLite2 데이터베이스](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data)
