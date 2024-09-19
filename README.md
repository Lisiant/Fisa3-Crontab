[우리FISA 3기 클라우드 엔지니어링] Crontab 활용 예시 프로젝트
# 💰 로또 번호 자동 수집기
### 개요
매주 토요일 저녁 8시 35분, 많은 사람들이 희망을 가지고 로또 번호를 확인한다.  
이에 로또 번호를 가져오는 스크립트를 자동으로 실행하기 위해 Crontab을 사용할 것이다.

### 1. 데이터베이스 테이블 설계

회차 번호, 번호 6개, 보너스 번호를 DB 테이블에 저장한다.  

```sql
CREATE TABLE lotto_results (
    id INT AUTO_INCREMENT PRIMARY KEY,
    draw_number INT UNIQUE NOT NULL,  -- 회차 번호
    number1 INT NOT NULL,
    number2 INT NOT NULL,
    number3 INT NOT NULL,
    number4 INT NOT NULL,
    number5 INT NOT NULL,
    number6 INT NOT NULL,
    bonus_number INT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 2. 로또 API를 통해 번호를 불러오는 스크립트 작성

> 로또 API 주소: `https://www.dhlottery.co.kr/common.do?method=getLottoNumber&drwNo=1`

`drwNo=` 항목 뒤에 회차 번호를 넣으면 해당 회차에 대한 로또 번호 정보가 출력된다.

```json
{
  "returnValue": "success",
  "drwNo": 1,
  "drwNoDate": "2002-12-07",
  "firstAccumamnt": 863604600,
  "firstPrzwnerCo": 0,
  "firstWinamnt": 0,
  "totSellamnt": 3681782000,
  "drwtNo1": 10,
  "drwtNo2": 23,
  "drwtNo3": 29,
  "drwtNo4": 33,
  "drwtNo5": 37,
  "drwtNo6": 40,
  "bnusNo": 16
}
```

**Python으로 데이터베이스 연결 및 저장**  

로또 번호를 API로 받아오고, 데이터베이스에 저장하는 Python 스크립트를 작성한다.
최신 회차가 `2024-09-19` 기준 1137회로, 해당 회차부터 저장한다.

### 3. 최신 회차 조회

DB에 접속한 뒤 SQL 구문을 통해 최신 회차의 로또 번호를 가져온다. 

```python
def get_latest_lotto():
    try:
        connection = connect_db()
        with connection.cursor() as cursor:
            # 최신 회차 번호 조회
            cursor.execute("SELECT * FROM lotto_results ORDER BY draw_number DESC LIMIT 1")
            latest_result = cursor.fetchone()

            if latest_result:
                print(f"최신 회차: {latest_result['draw_number']}")
                print(f"당첨 번호: {latest_result['number1']}, {latest_result['number2']}, {latest_result['number3']}, {latest_result['number4']}, {latest_result['number5']}, {latest_result['number6']}")
                print(f"보너스 번호: {latest_result['bonus_number']}")
            else:
                print("데이터가 없습니다.")
    except Exception as e:
        print(f"오류 발생: {e}")
    finally:
        connection.close()

if __name__ == "__main__":
    get_latest_lotto()
```

### 4. Crontab으로 자동화

터미널에서 `crontab -e` 명령어를 통해 crontab 편집 모드로 들어간다.

이후 로또 번호 저장 스크립트 스케줄을 설정한다.
로또 발표일인 매주 토요일 저녁 9시에 최신 로또 번호를 저장하도록 설정한다.
```cron
0 21 * * 6 /usr/bin/python3 /home/username/lotto_to_db.py >> /home/username/lotto_to_db.log 2>&1
```

DB에서 로또 번호 조회하는 스크립트 스케줄을 설정한다.
매일 오전 9시에 최신 회차의 로또 번호를 DB에서 불러오는 작업을 설정한다.
```cron
0 9 * * * /usr/bin/python3 /home/username/fetch_lotto_from_db.py >> /home/username/fetch_lotto_from_db.log 2>&1
```

