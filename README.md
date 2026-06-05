# 123dsada

## 로그인 흐름

```mermaid
sequenceDiagram
    autonumber
    actor User as 사용자
    participant Frontend as 프론트엔드
    participant AuthAPI as 인증 API
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
