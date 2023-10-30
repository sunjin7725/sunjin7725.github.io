---
title: Go RESTFul API(식품영양성분 API)
categories: [Go]
tags: [Go, Language]
comments : true
---

# Go를 활용한 식품영양성분DB 조회

## 개요

Git: [https://github.com/sunjin7725/api-call-test](https://github.com/sunjin7725/api-call-test)

- Go를 공부하고 공공 데이터를 불러오는 Restful API 개발  
- 활용데이터는 아래와 같음  
  [식품의약품안전처 식품영양성분DB](https://www.foodsafetykorea.go.kr/api/newDatasetDetail.do)
- 개발 현재 진행 중(2023-09-25 ~ 2023-10-27)
- 추가 예정부분은 천천히 추가할수도 있음    

## 개발환경

```cmd
OS: Windows
Language: Go 1.21.1
```

## 기술스택

### 기본

- Go
- Gin
- Docker

### 기능 추가가 된다면

- Postgresql

## 개발 기능

### 필수

- [x] [viper](https://github.com/spf13/viper)를 활용한 개발/운영환경 간의 config 관리  
- [x] 식품영양성분 Open API 호출
- [x] 호출 결과 Model을 Structure로 구성하여 객체화 하기
- [x] [Gin](https://github.com/gin-gonic/gin) Web Framework를 활용하여 Restful API 화(사용하기 편하도록)
- [x] Swagger를 통해 API 문서 정리
- [x] Rasberry pi 내, 도커에 배포하기
  - deploy/startup.sh

### 추가예정(?)

- [ ] 별도 배치 프로그램을 작성하여, 데이터를 Postgresql에 저장하고 DB에서 결과를 끌어오도록 변경
