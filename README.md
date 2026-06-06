# 시퀀스 다이어그램 모음

## 목차
1. [로그인 흐름](#1-로그인-흐름)
2. [예약 신청 흐름](#2-예약-신청-흐름)
3. [예약 승인 / 거절 흐름](#3-예약-승인--거절-흐름)

---

## 1. 로그인 흐름

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#f0f0f0', 'primaryTextColor': '#333', 'primaryBorderColor': '#999', 'lineColor': '#555', 'secondaryColor': '#f9f9f9', 'tertiaryColor': '#fff'}}}%%
sequenceDiagram
    autonumber
    actor User as 사용자
    participant Frontend as 프론트엔드
    participant AuthAPI as 인증 API
    participant DB as DB (User)

    User->>Frontend: 학번(student_id), 비밀번호 입력
    Frontend->>AuthAPI: POST /api/auth/login
    note right of AuthAPI: { student_id, password }

    AuthAPI->>DB: User.findOne({ student_id })
    DB-->>AuthAPI: User 도큐먼트 반환

    alt 사용자 없음
        AuthAPI-->>Frontend: 401 Unauthorized
        Frontend-->>User: 로그인 실패 메시지 표시
    else is_active = false (이용정지)
        AuthAPI-->>Frontend: 403 Forbidden
        Frontend-->>User: 이용정지 안내 메시지 표시
    else 사용자 존재
        AuthAPI->>AuthAPI: bcrypt로 password 검증
        alt 비밀번호 불일치
            AuthAPI-->>Frontend: 401 Unauthorized
            Frontend-->>User: 로그인 실패 메시지 표시
        else 비밀번호 일치
            AuthAPI->>AuthAPI: accessToken 생성 (JWT)
            AuthAPI->>AuthAPI: refreshToken 생성 (JWT)
            AuthAPI->>DB: User.updateOne({ refresh_token })
            DB-->>AuthAPI: 저장 완료
            AuthAPI-->>Frontend: 200 OK
            note right of AuthAPI: { accessToken, refreshToken }
            Frontend-->>User: 로그인 완료
        end
    end
```

[↑ 목차로](#목차)

---

## 2. 예약 신청 흐름

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#f0f0f0', 'primaryTextColor': '#333', 'primaryBorderColor': '#999', 'lineColor': '#555', 'secondaryColor': '#f9f9f9', 'tertiaryColor': '#fff'}}}%%
sequenceDiagram
    autonumber
    actor User as 사용자
    participant Frontend as 프론트엔드
    participant API as 예약 API
    participant DB_U as DB (User)
    participant DB_R as DB (Resource)
    participant DB_RV as DB (Reservation)

    User->>Frontend: 자원 선택 및 시간·목적 입력
    Frontend->>API: POST /api/reservations
    note right of API: Bearer accessToken<br/>{ resource_id, start_time,<br/>  end_time, purpose }

    API->>DB_U: User.findById (토큰 검증)
    DB_U-->>API: User 도큐먼트 반환

    note over API: [1단계] 리소스 상태 확인
    API->>DB_R: Resource.findById({ resource_id })
    DB_R-->>API: Resource 도큐먼트 반환

    alt 자원 없음
        API-->>Frontend: 404 Not Found
        Frontend-->>User: 존재하지 않는 자원 안내
    else status = maintenance
        API-->>Frontend: 400 Bad Request
        Frontend-->>User: 현재 점검 중인 시설 안내
    else status = retired
        API-->>Frontend: 400 Bad Request
        Frontend-->>User: 사용 불가능한 시설 안내
    else status = active
        note over API: [2단계] 운영 요일 / 시간 확인<br/>(operating_hours.days, start_time, end_time)
        alt 운영일 아님 (Mon~Fri 외)
            API-->>Frontend: 400 Bad Request
            Frontend-->>User: 운영일이 아닙니다 안내
        else 운영 시간 외 (09:00~22:00 벗어남)
            API-->>Frontend: 400 Bad Request
            Frontend-->>User: 운영 시간 외 안내
        else 운영 시간 내
            note over API: [3단계] 중복 예약 확인
            API->>DB_RV: Reservation.find<br/>({ resource_id, start_time, end_time,<br/>  status: waiting/reserved/using })
            DB_RV-->>API: 기존 예약 목록 반환
            alt 완전히 동일한 시간 예약 존재
                API-->>Frontend: 409 Conflict
                Frontend-->>User: 이미 동일한 예약 존재 안내
            else
                note over API: [4단계] 동시 예약 수 확인<br/>(max_concurrent, 기본값 1)
                alt 동시 예약 수 초과
                    API-->>Frontend: 409 Conflict
                    Frontend-->>User: 해당 시간대 예약 마감 안내
                else 모든 검증 통과
                    API->>DB_U: current_reservations + 1 업데이트
                    API->>DB_RV: Reservation.create<br/>({ user_id, resource_id, start_time,<br/>  end_time, purpose, status: waiting })
                    DB_RV-->>API: 생성된 Reservation 도큐먼트 반환
                    API-->>Frontend: 201 Created
                    note right of API: { message: 예약 신청 완료,<br/>  reservation: { ..., status: waiting } }
                    Frontend-->>User: 예약 신청 완료 (승인 대기 중)
                end
            end
        end
    end
```

[↑ 목차로](#목차)

---

## 3. 예약 승인 / 거절 흐름

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#f0f0f0', 'primaryTextColor': '#333', 'primaryBorderColor': '#999', 'lineColor': '#555', 'secondaryColor': '#f9f9f9', 'tertiaryColor': '#fff'}}}%%
sequenceDiagram
    autonumber
    actor Admin as 관리자
    participant Frontend as 관리자 화면
    participant API as 예약 API
    participant DB_U as DB (User)
    participant DB_RV as DB (Reservation)

    Admin->>Frontend: 전체 예약 목록 조회 요청
    Frontend->>API: GET /api/reservations
    note right of API: Bearer accessToken (admin)

    API->>DB_U: User.findById → role 확인
    DB_U-->>API: User 도큐먼트 반환 (role: admin)

    API->>DB_RV: Reservation.find().populate(user_id, resource_id)
    DB_RV-->>API: 전체 예약 목록 반환
    API-->>Frontend: 200 OK (예약 목록)
    Frontend-->>Admin: 예약 목록 표시

    alt 승인 처리
        Admin->>Frontend: 특정 예약 승인 클릭
        Frontend->>API: PATCH /api/reservations/:id/approve
        API->>DB_RV: Reservation.findById({ _id })
        DB_RV-->>API: Reservation 도큐먼트 반환
        alt status가 waiting이 아님
            API-->>Frontend: 400 Bad Request
            Frontend-->>Admin: 승인 불가 안내
        else status = waiting
            API->>DB_RV: Reservation.updateOne<br/>({ status: reserved, approved_by: admin._id })
            DB_RV-->>API: 수정된 도큐먼트 반환
            API-->>Frontend: 200 OK
            Frontend-->>Admin: 승인 완료 표시
        end
    else 거절 처리
        Admin->>Frontend: 특정 예약 거절 클릭 + 사유 입력
        Frontend->>API: PATCH /api/reservations/:id/reject
        note right of API: { cancel_reason }
        API->>DB_RV: Reservation.findById({ _id })
        DB_RV-->>API: Reservation 도큐먼트 반환
        alt status가 waiting이 아님
            API-->>Frontend: 400 Bad Request
            Frontend-->>Admin: 거절 불가 안내
        else status = waiting
            API->>DB_RV: Reservation.updateOne<br/>({ status: cancelled, cancel_reason })
            DB_RV-->>API: 수정된 도큐먼트 반환
            API->>DB_U: current_reservations - 1 업데이트
            DB_U-->>API: 완료
            API-->>Frontend: 200 OK
            Frontend-->>Admin: 거절 완료 표시
        end
    end
```

[↑ 목차로](#목차)
