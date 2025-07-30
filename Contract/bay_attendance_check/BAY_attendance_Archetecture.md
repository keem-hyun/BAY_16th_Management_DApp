⏺ BAY 출석체크 컨트랙트 구조 및 작동방식 분석 📊

🏗️ 전체 아키텍처 개요

프로그램 구조

bay_attendance_check/
├── 핵심 로직
(programs/bay_attendance_check/src/)
│   ├── lib.rs          → 메인 진입점
│   ├── state.rs        → 데이터 구조 정의
│   ├── errors.rs       → 에러 타입
│   └── instructions/   → 각 기능별 로직
│       ├── initialize_member.rs
│       ├── initialize_session.rs
│       ├── check_in.rs
│       ├── update_session.rs
│       └── get_stats.rs
└── 클라이언트 (scripts/)
    ├── admin.ts        → 운영진 관리
    ├── member.ts       → 학회원 기능
    └── stats.ts        → 통계 조회

📦 핵심 데이터 구조

1. Member (학회원 계정)

pub struct Member {
    pub wallet: Pubkey,          // 학회원 지갑 주소
    pub role: MemberRole,        // 역할 (Admin/Member)
    pub total_attendance: u32,   // 총 출석 횟수
    pub total_late: u32,         // 총 지각 횟수
    pub total_absence: u32,      // 총 결석 횟수
    pub total_points: u64,       // 총 획득 포인트
    pub is_active: bool,         // 활성화 상태
    pub bump: u8,                // PDA bump
}

2. Session (세션 계정)

pub struct Session {
    pub admin: Pubkey,           // 세션 생성 운영진
    pub session_date: i64,       // 세션 날짜
    pub start_time: i64,         // 시작 시간 (7:30)
    pub late_time: i64,          // 지각 기준 시간 (8:00)
    pub total_attendees: u32,    // 총 출석자 수
    pub total_late: u32,         // 총 지각자 수
    pub is_active: bool,         // 세션 활성 상태
    pub bump: u8,
}

3. AttendanceRecord (출석 기록)

pub struct AttendanceRecord {
    pub member: Pubkey,          // 학회원 주소
    pub session: Pubkey,         // 세션 주소
    pub check_in_time: i64,      // 체크인 시간
    pub status: AttendanceStatus,// 출석 상태
    pub points_earned: u8,       // 획득 포인트
    pub bump: u8,
}

🔑 PDA (Program Derived Address) 구조

PDA Seeds 패턴

1. Member PDA: ["member", 학회원_지갑_주소]
2. Session PDA: ["session", 세션_날짜_바이트]
3. AttendanceRecord PDA: ["attendance", 세션_주소, 학회원_주소]

⚙️ 주요 기능 및 작동 방식

1. 학회원 등록 (initialize_member)

[흐름도]
사용자 → 지갑 연결 → Member PDA 생성 → 역할 설정 → 초기화
- 운영진: Admin 권한 필요
- 일반 학회원: 누구나 등록 가능
- 각 지갑당 하나의 Member 계정만 생성 가능

2. 세션 생성 (initialize_session)

[흐름도]
운영진 → 날짜/시간 설정 → Session PDA 생성 → 활성화
- 권한: Admin 역할만 가능
- 시간 설정:
- start_time: 세션 시작 시간
- late_time: 지각 허용 시간
- 날짜별로 고유한 세션 생성

3. 출석 체크 (check_in)

[흐름도]
학회원 → 세션 확인 → 시간 검증 → 상태 결정 → 포인트 지급

시간별 처리 로직:
현재시간 ≤ start_time     → 출석 (10 포인트)
start_time < 현재시간 ≤ late_time → 지각 (5 포인트)
현재시간 > late_time      → 체크인 불가

검증 사항:
- 세션 활성화 상태
- 학회원 활성화 상태
- 중복 출석 방지 (PDA로 보장)

4. 세션 관리 (update_session_status)

- Admin만 세션 상태 변경 가능
- 활성/비활성 토글

5. 통계 조회 (get_stats)

- 학회원 통계: 출석/지각/결석 횟수, 총 포인트
- 세션 통계: 참석자 수, 지각자 수

🔐 보안 및 권한 구조

권한 계층

Admin (운영진)
├── 세션 생성/종료
├── 통계 전체 조회
└── 긴급 수정 권한

Member (일반 학회원)
├── 본인 출석 체크
├── 본인 기록 조회
└── 포인트 확인

보안 메커니즘

1. PDA 기반 계정 관리: 예측 가능한 주소로 중복 방지
2. Signer 검증: 트랜잭션 서명자 확인
3. 시간 검증: 블록체인 Clock 사용
4. 에러 처리: 명확한 에러 타입 정의

💰 포인트 시스템

포인트 지급 규칙

- 출석 (7:30 이전): 10 포인트
- 지각 (7:30-8:00): 5 포인트
- 결석: 0 포인트

포인트 누적

- Member 계정의 total_points에 자동 누적
- 향후 SPL 토큰 연동 가능

🚀 사용 시나리오

운영진 워크플로우

1. 관리자 지갑으로 Admin 등록
2. 세션 생성 (날짜, 시간 설정)
3. 학회원들이 출석 체크
4. 세션 종료
5. 통계 확인

학회원 워크플로우

1. 지갑 생성 및 SOL 충전
2. Member 등록
3. 세션 시간에 출석 체크
4. 포인트 및 출석 기록 확인

🔧 기술적 특징

1. Anchor Framework: 타입 안전성과 개발 편의성
2. PDA 패턴: 계정 주소 예측 가능성
3. 시간 기반 로직: Solana Clock 활용
4. 원자적 트랜잭션: 모든 작업이 성공하거나 전체 롤백

📈 확장 가능성

1. SPL 토큰 통합: 실제 토큰으로 포인트 지급
2. NFT 발행: 특별 출석 증명서
3. 웹 인터페이스: React 기반 프론트엔드
4. QR 코드: 모바일 출석 체크
5. 이벤트 로깅: 체인 외부 데이터 분석

이 구조는 투명하고 신뢰할 수 있는 출석 관리 시스템을 제공하며, 블록체인의 불변성을 활용해 출석 기록의 위변조를 방지합니다.