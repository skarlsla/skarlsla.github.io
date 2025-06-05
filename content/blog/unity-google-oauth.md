---
title: 'Unity PC에서 OAuth2.0 Google 로그인 연동하기'
date: '2024-03-21'
category: 'Unity'
tags: ['Unity', 'OAuth2.0', 'Google Login', 'Authentication']
description: 'Unity PC 클라이언트에서 OAuth2.0을 사용한 Google 로그인 구현 방법을 상세히 설명합니다.'
---

Unity PC 클라이언트에서 SNS 로그인을 구현할 때 가장 일반적인 방식 중 하나는 OAuth2.0을 사용하는 것입니다. 특히, 서버가 이미 OAuth2 기반 로그인 시스템을 가지고 있다면, 클라이언트는 복잡한 인증 로직을 직접 구현할 필요 없이 서버와 안전하게 통신하는 방식으로 구성할 수 있습니다.

이 글에서는 xxx.com 도메인을 사용하는 OAuth2.0 인증 서버와 Unity 클라이언트를 연동하여 Google 로그인을 처리하는 전체 과정을 소개합니다.

## 전체 인증 흐름 요약

```arduino
Unity Client → 서버에 로그인 요청 → 구글 로그인 URL 오픈 (WebView)
      ↓
사용자 구글 로그인 완료 → 서버 상태 폴링 → accessToken 발급
```

### 1. 로그인 시작 요청

Unity 클라이언트는 먼저 서버에 로그인 요청을 보냅니다.

#### 요청
```http
POST https://dapi.xxx.com/auth/google/start
```

#### 응답 예시
```json
{
  "loginToken": "56cc9e13-c833-41e7-9d9f-9f9540f9d9d4",
  "loginUrl": "https://accounts.google.com/o/oauth2/auth?...&state=56cc9e13-c833-41e7-9d9f-9f9540f9d9d4"
}
```
- `loginUrl`: Unity WebView에서 열 URL
- `loginToken`: 이후 상태 확인 및 토큰 요청 시 사용할 고유 키

### 2. Unity WebView로 로그인 진행

loginUrl을 Unity의 WebView(예: WebViewObject 또는 UniWebView)에서 띄우고 사용자가 로그인하도록 유도합니다.

구글 인증이 성공하면 아래와 같은 메시지를 포함한 페이지가 출력됩니다:

```json
{
  "code": "0000",
  "msg": "Login successful. You can return to the app."
}
```

이 시점에 WebView를 닫고, 클라이언트는 다음 단계로 진행합니다.

### 3. 로그인 상태 확인 (Polling)

유저의 로그인 완료 여부를 주기적으로 서버에 확인합니다.

#### 요청
```http
GET http://dapi.xxx.com/auth/google/status/{loginToken}
```

#### 응답 예시
```json
{
  "code": "0000",
  "data": "pending",
  "msg": "success"
}
```
- `"data": "pending"` → 아직 로그인 대기 중
- `"data": "complete"` → 로그인 완료
- `"code": "1100"` → 잘못된 또는 만료된 토큰

Unity에서는 1~2초 간격으로 반복 요청하며 로그인 완료 여부를 체크합니다.

### 4. 로그인 토큰 발급

상태가 complete가 되면, 클라이언트는 최종적으로 액세스 토큰을 요청합니다.

#### 요청
```http
GET https://dapi.xxx.com/auth/google/token/{loginToken}
```

#### 성공 응답 예시
```json
{
  "code": "0000",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "atExpires": 1747726950,
    "rtExpires": 1747899750
  },
  "msg": "success"
}
```

#### 실패 시
```json
{
  "code": "1100",
  "msg": "Invalid or expired login_token"
}
```

### 5. Unity 클라이언트 내부 처리 흐름 요약

```csharp
// Step 1: 로그인 시작 요청
// POST https://dapi.xxx.com/auth/google/start

// Step 2: WebView로 loginUrl 띄우기
// 사용자가 로그인 완료하면 WebView 닫기

// Step 3: Polling으로 status 확인
// GET http://dapi.xxx.com/auth/google/status/{loginToken}

// Step 4: 토큰 발급 요청
// GET https://dapi.xxx.com/auth/google/token/{loginToken}
```

발급받은 accessToken은 이후 API 요청 시 `Authorization: Bearer {accessToken}` 형태로 사용됩니다.

## 보안 팁

- loginToken은 일정 시간이 지나면 만료됩니다. 유효 시간 내에 polling 및 token 요청을 완료해야 합니다.
- accessToken과 refreshToken은 보안 저장소(예: 암호화된 파일 또는 OS 키체인)에 저장하는 것이 좋습니다.
- 추후 refreshToken을 이용한 재인증 로직도 구현해두는 것이 바람직합니다.

## 마무리

OAuth2 방식의 SNS 로그인을 Unity PC 앱에 연동하는 것은 복잡하게 보일 수 있지만, 인증 서버에서 제공하는 login URL + 상태 polling + token 발급 API 조합을 잘 활용하면 쉽게 처리할 수 있습니다.

이제 여러분의 Unity 게임이나 앱에서도 Google 로그인을 편리하게 제공해보세요! 