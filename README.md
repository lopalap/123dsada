# 123dsada

## 1. 로그인 흐름
```mermaid
sequenceDiagram
    autonumber
    actor User as 사용자
    participant Frontend as 프론트엔드
    participant AuthAPI as 인증 API
    participant DB as DB

    User->>Frontend: 학번, 비밀번호 입력
    Frontend->>AuthAPI: POST /api/auth/login
    AuthAPI->>DB: 사용자 조회
    DB-->>AuthAPI: 사용자 정보 반환

    alt 인증 성공
        AuthAPI->>AuthAPI: 비밀번호 검증
        AuthAPI->>AuthAPI: 토큰 생성
        AuthAPI-->>Frontend: 200 OK
        Frontend-->>User: 로그인 완료
    else 인증 실패
        AuthAPI-->>Frontend: 401 Unauthorized
        Frontend-->>User: 로그인 실패 메시지 표시
    end
    sequenceDiagram
    autonumber
    actor User as 사용자
    participant Frontend as 프론트엔드
    participant ReservationAPI as 예약 API
    participant ResourceAPI as 자원 API
    participant DB as DB

    User->>Frontend: 자원 선택 후 시간 입력
    Frontend->>ReservationAPI: POST /api/reservations
    ReservationAPI->>DB: 사용자 토큰 검증
    DB-->>ReservationAPI: 사용자 정보 반환
    ReservationAPI->>ResourceAPI: 자원 상태 확인
    ResourceAPI->>DB: 조회
    DB-->>ResourceAPI: 정보 반환
    ResourceAPI-->>ReservationAPI: 상태 반환

    alt 자원 사용 불가
        ReservationAPI-->>Frontend: 400 또는 404 오류
        Frontend-->>User: 예약 불가 메시지
    else 자원 사용 가능
        ReservationAPI->>ReservationAPI: 시간 검증
        ReservationAPI->>DB: 기존 예약 조회
        DB-->>ReservationAPI: 목록 반환
        alt 중복 예약 존재
            ReservationAPI-->>Frontend: 409 Conflict
            Frontend-->>User: 중복 안내
        else 예약 가능
            ReservationAPI->>DB: 예약 생성
            DB-->>ReservationAPI: 생성 결과 반환
            ReservationAPI-->>Frontend: 201 Created
            Frontend-->>User: 신청 완료
        end
    end
    sequenceDiagram
    autonumber
    actor Admin as 관리자
    participant Frontend as 관리자 화면
    participant ReservationAPI as 예약 API
    participant DB as DB

    Admin->>Frontend: 목록 조회
    Frontend->>ReservationAPI: GET /api/reservations
    ReservationAPI->>DB: 권한 확인
    DB-->>ReservationAPI: 정보 반환
    ReservationAPI->>DB: 목록 조회
    DB-->>ReservationAPI: 목록 반환
    ReservationAPI-->>Frontend: 200 OK
    Frontend-->>Admin: 목록 표시

    alt 승인
        Admin->>Frontend: 승인 클릭
        Frontend->>ReservationAPI: PATCH /api/reservations/:id/approve
        ReservationAPI->>DB: 상태 업데이트
        DB-->>ReservationAPI: 반환
        ReservationAPI-->>Frontend: 완료
        Frontend-->>Admin: 상태 표시
    else 거절
        Admin->>Frontend: 거절 클릭
        Frontend->>ReservationAPI: PATCH /api/reservations/:id/reject
        ReservationAPI->>DB: 상태 업데이트
        DB-->>ReservationAPI: 반환
        ReservationAPI-->>Frontend: 완료
        Frontend-->>Admin: 상태 표시
    end
