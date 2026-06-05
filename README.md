# 123dsada

## 1. 로그인 흐름
```mermaid
sequenceDiagram
    autonumber
    actor User as 사용자
    participant Frontend as 프론트엔드
    participant AuthAPI as 인증 API<br/>/api/auth
    participant DB as DB

    User->>Frontend: 학번, 비밀번호 입력
    Frontend->>AuthAPI: POST /api/auth/login
    AuthAPI->>DB: 사용자 조회<br/>student_id 확인
    DB-->>AuthAPI: 사용자 정보 반환

    alt 인증 성공
        AuthAPI->>AuthAPI: 비밀번호 검증
        AuthAPI->>AuthAPI: accessToken, refreshToken 생성
        AuthAPI-->>Frontend: 200 OK<br/>accessToken + refreshToken
        Frontend-->>User: 로그인 완료
    else 인증 실패
        AuthAPI-->>Frontend: 401 Unauthorized
        Frontend-->>User: 로그인 실패 메시지 표시
    end
## 2. 예약 신청 흐름
```mermaid  
sequenceDiagram
    autonumber
    actor User as 사용자
    participant Frontend as 프론트엔드
    participant ReservationAPI as 예약 API<br/>/api/reservations
    participant ResourceAPI as 자원 API<br/>/api/resources
    participant DB as DB

    User->>Frontend: 자원 선택 후 예약 시간 입력
    Frontend->>ReservationAPI: POST /api/reservations<br/>Bearer accessToken

    ReservationAPI->>DB: 사용자 토큰 검증
    DB-->>ReservationAPI: 사용자 정보 반환

    ReservationAPI->>ResourceAPI: 자원 상태 확인
    ResourceAPI->>DB: resource_id 조회
    DB-->>ResourceAPI: 자원 정보 반환
    ResourceAPI-->>ReservationAPI: active / maintenance / retired 상태 반환

    alt 자원이 없거나 사용 불가
        ReservationAPI-->>Frontend: 400 또는 404 오류
        Frontend-->>User: 예약 불가 메시지 표시
    else 자원 사용 가능
        ReservationAPI->>ReservationAPI: 운영 요일/시간 검증
        ReservationAPI->>DB: 같은 자원, 같은 시간대 예약 조회
        DB-->>ReservationAPI: 기존 예약 목록 반환

        alt 중복 예약 존재
            ReservationAPI-->>Frontend: 409 Conflict
            Frontend-->>User: 이미 예약된 시간 안내
        else 예약 가능
            ReservationAPI->>DB: 예약 생성<br/>status = waiting
            DB-->>ReservationAPI: 생성된 예약 반환
            ReservationAPI-->>Frontend: 201 Created<br/>예약 신청 완료
            Frontend-->>User: 예약 신청 완료 표시
        end
    end
```
