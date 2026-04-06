# Python 5주차 정규 과제 

📌Python 정규과제는 매주 정해진 분량의 『*파이썬 라이브러리를 활용한 데이터 분석*』 을 읽고 학습하는 것입니다. 이번주는 아래의 **Python_5th_TIL**에 나열된 분량을 읽고 공부하시면 됩니다.

아래의 문제를 풀어보며 학습 내용을 점검하세요. 문제를 해결하는 과정에서 개념을 스스로 정리하고, 필요한 경우 참고 자료를 통해 보완하는 것이 좋습니다.

**교재 실습 예제 파일은 07_Python_Template 레포지토리의 notebooks 폴더에 업로드되어 있습니다.**

**아나콘다 환경에서는 많은 패키지가 기본적으로 포함되어 있어 별도의 설치 없이 사용할 수 있지만, 환경에 따라 conda install이나 pip install이 필요할 수 있습니다.**

**👀(수행 인증샷은 필수입니다.)** 

## Python_5th_TIL

### 6장 데이터 로딩과 저장, 파일 형식
#### 1. 텍스트 파일에서 데이터를 읽고 쓰는 법
#### 2. 이진 데이터 형식
#### 3. 웹 API와 함께 사용하기
#### 4. 데이터베이스와 함께 사용하기
#### 5. 마치며
### 7장 데이터 정제 및 준비
#### 1. 누락된 데이터 처리하기
#### 2. 데이터 변형 


## Study Schedule

| 주차  | 공부 범위     | 완료 여부 |
| ----- | ------------- | --------- |
| 1주차 | p.25~82    | ✅         |
| 2주차 | p.83~129   | ✅         |
| 3주차 | p.131~179  | ✅         |
| 4주차 | p.181~246 | ✅         |
| 5주차 | p.247~309 | ✅         |
| 6주차 | p.310~379 | 🍽️         |
| 7주차 | p.381~465 | 🍽️         |


<br>

<!-- 여기까진 그대로 둬 주세요-->

---

# 1️⃣ 학습 내용 정리

## 1. 텍스트 파일에서 데이터를 읽고 쓰는 법

### 개념정리

Pandas는 표 형식의 데이터를 DataFrame으로 읽어오는 다양한 함수(read_csv, read_table 등)를 제공합니다. 현업에서는 데이터의 형태가 불규칙한 경우가 많아 구분자 지정(sep), 헤더 지정(header), 특정 열을 인덱스로 설정(index_col), 그리고 결측치 처리(na_values)와 같은 다양한 매개변수를 능숙하게 활용하는 것이 중요합니다. 처리한 데이터는 to_csv 메서드를 통해 다시 텍스트 파일로 내보낼 수 있습니다.

### 실습 인증

<!-- 예제 실습을 진행한 후, 실행 화면을 2-3장 캡쳐하여 제출해주세요. -->
![alt text](image-3.png)
C:\Users\tkvkm\OneDrive\바탕 화면\Dart_B\Python_Homework\image-3.png

![alt text](image-4.png)
C:\Users\tkvkm\OneDrive\바탕 화면\Dart_B\Python_Homework\image-4.png
## 2. 이진 데이터 형식

### 개념정리

CSV 같은 텍스트 파일은 읽고 쓰는 속도가 느립니다. Pandas는 파이썬 내장 pickle 직렬화를 사용하여 데이터를 이진 형식으로 저장(to_pickle)하고 불러올(read_pickle) 수 있어, 중간 작업 내역을 빠르고 효율적으로 저장할 때 유용합니다. 또한 대용량 데이터를 처리할 때는 HDF5 형식을, 엑셀 파일 처리 시에는 read_excel과 to_excel을 사용하여 업무 효율을 높일 수 있습니다.

### 실습 인증

<!-- 예제 실습을 진행한 후, 실행 화면을 2-3장 캡쳐하여 제출해주세요. -->

![alt text](image-5.png)
C:\Users\tkvkm\OneDrive\바탕 화면\Dart_B\Python_Homework\image-5.png

![alt text](image-6.png)
C:\Users\tkvkm\OneDrive\바탕 화면\Dart_B\Python_Homework\image-6.png

## 3. 웹 API와 함께 사용하기

### 개념정리

많은 웹사이트와 공공기관이 JSON 형식으로 데이터를 제공하는 공개 API를 가지고 있습니다. 파이썬의 requests 패키지를 사용하면 이러한 웹 API에 HTTP 요청을 보내고 데이터를 받아올 수 있습니다. 받아온 JSON 응답 데이터는 파이썬의 기본 딕셔너리 형태로 파싱된 후, 손쉽게 Pandas DataFrame으로 변환하여 분석에 활용할 수 있습니다.

### 실습 인증

<!-- 예제 실습을 진행한 후, 실행 화면을 2-3장 캡쳐하여 제출해주세요. -->

![alt text](image-7.png)
C:\Users\tkvkm\OneDrive\바탕 화면\Dart_B\Python_Homework\image-7.png

![alt text](image-8.png)
C:\Users\tkvkm\OneDrive\바탕 화면\Dart_B\Python_Homework\image-8.png
## 4. 데이터베이스와 함께 사용하기

### 개념정리

대부분의 기업 데이터는 텍스트 파일이 아닌 SQL 기반의 관계형 데이터베이스(MySQL, PostgreSQL 등)에 저장됩니다. 파이썬의 sqlite3 모듈이나 SQLAlchemy를 활용하면 데이터베이스에 쿼리를 보내 데이터를 읽어올 수 있습니다. Pandas의 read_sql 함수는 SQL 쿼리 결과를 곧바로 DataFrame으로 변환해주어, DB 추출부터 데이터 분석까지의 파이프라인을 매끄럽게 연결해 줍니다.

### 실습 인증

<!-- 예제 실습을 진행한 후, 실행 화면을 2-3장 캡쳐하여 제출해주세요. -->

![alt text](image-9.png)
C:\Users\tkvkm\OneDrive\바탕 화면\Dart_B\Python_Homework\image-9.png

![alt text](image-10.png)
C:\Users\tkvkm\OneDrive\바탕 화면\Dart_B\Python_Homework\image-10.png

## 5. 누락된 데이터 처리하기

### 개념정리

실제 데이터는 비어있는 값(결측치, NaN)을 포함하는 경우가 대부분입니다. Pandas에서는 isna()를 통해 결측치를 찾고, dropna()를 이용해 누락된 데이터가 있는 행이나 열을 제거하거나, fillna()를 사용해 특정 값(예: 평균값, 중앙값, 0 등)으로 결측치를 대체할 수 있습니다. 결측치 처리는 머신러닝 모델의 성능이나 분석 결과에 직접적인 영향을 미치므로 매우 신중하게 접근해야 합니다.

### 실습 인증

<!-- 예제 실습을 진행한 후, 실행 화면을 2-3장 캡쳐하여 제출해주세요. -->

![alt text](image-11.png)
C:\Users\tkvkm\OneDrive\바탕 화면\Dart_B\Python_Homework\image-11.png

![alt text](image-12.png)
C:\Users\tkvkm\OneDrive\바탕 화면\Dart_B\Python_Homework\image-12.png
## 6. 데이터 변형 

### 개념정리
데이터를 분석하기 좋은 형태로 가공하는 과정입니다. duplicated()와 drop_duplicates()를 통해 중복된 행을 제거하고, map()을 사용하여 딕셔너리 기반으로 값을 매핑하여 새로운 열을 추가할 수 있습니다. 또한 replace()로 특정 값을 다른 값으로 치환하거나, rename()으로 축의 색인을 변경하는 등 원본 데이터를 목적에 맞게 재구조화하는 핵심 기술들을 포함합니다.

### 실습 인증

<!-- 예제 실습을 진행한 후, 실행 화면을 2-3장 캡쳐하여 제출해주세요. -->

![alt text](image-13.png)
C:\Users\tkvkm\OneDrive\바탕 화면\Dart_B\Python_Homework\image-13.png

![alt text](image-14.png)
C:\Users\tkvkm\OneDrive\바탕 화면\Dart_B\Python_Homework\image-14.png

# 2️⃣ 실습 과제

각 문제에 대한 실행 결과가 확인되도록 코드를 작성하고 실행한 뒤, **모든 문제의 실행 화면을 캡처하여 제출하시기 바랍니다.**

**1. 아래 코드를 실행하여 텍스트와 데이터를 선언합니다.**
```python
import pandas as pd
import json
import requests

# 1. 테스트용 CSV 내용 (메모리 내 시뮬레이션용)
csv_data = "id,name,score,note\n1,Kim,85,NA\n2,Lee,NULL,Good\n3,Park,90,None"
with open("test_data.csv", "w") as f:
    f.write(csv_data)

# 2. 테스트용 JSON 문자열
json_obj = """
{
    "company": "DataService",
    "employees": [
        {"name": "Alice", "age": 30, "dept": "IT"},
        {"name": "Bob", "age": 25, "dept": "HR"}
    ]
}
"""
```

**2. 문제**
```
1. CSV 파일 읽기 및 결측치 지정
  - 문제 설명: 제공받은 test_data.csv 파일을 읽어오기(단, 데이터의 특성에 맞게 옵션 설정)
  - read_csv()를 사용하여 파일을 읽으세요.
  - 이때 NA, NULL, None이라는 문자열을 모두 **결측치(NaN)**로 인식하도록 na_values 옵션을 설정하세요.
  - print()를 이용해 읽어온 DataFrame을 출력하세요.

2. JSON 데이터 변환 및 특정 데이터 추출
  - 문제 설명: 문자열 형태의 JSON 데이터를 파싱하여 직원 명단만 추출
  - json.loads()를 사용하여 json_obj 문자열을 파이썬 객체로 변환하세요.
  - 변환된 객체에서 employees 리스트만 추출하여 DataFrame으로 만드세요.
  - print()를 이용해 생성된 직원 명단 DataFrame을 출력하세요.

3. 웹 API 데이터 가져오기
  - 문제 설명: 판다스 깃허브 저장소의 이슈(Issues) 데이터를 가져와 상위 항목 확인
  - requests.get()을 사용하여 https://api.github.com/repos/pandas-dev/pandas/issues URL의 데이터를 가져오세요.
  - 응답받은 JSON 데이터를 DataFrame으로 변환하세요.
  - print()를 이용해 데이터의 상단 5행(head)을 출력하세요.
```

<!![alt text](image.png)
C:\Users\tkvkm\OneDrive\바탕 화면\Dart_B\Python_Homework\image.png

![alt text](image-1.png)
C:\Users\tkvkm\OneDrive\바탕 화면\Dart_B\Python_Homework\image-1.png

![alt text](image-2.png)
C:\Users\tkvkm\OneDrive\바탕 화면\Dart_B\Python_Homework\image-2.png

### 🎉 수고하셨습니다.






