---
published: true
title: Spring Security 로그인 처리 고민과 회고
date: 2025-04-28 18:28:00 +09:00
categories: [백엔드, 스프링부트]
tags:
  [스프링부트 로그인, filter를 사용한 로그인, 스프링 시큐리티]
---
#  Spring Security 로그인 처리 고민과 회고

## 1. 로그인 요청을 Controller에서 처리한 경험

Spring Boot로 API 서버를 개발할 때,  
**Spring Security의 formLogin 파라미터 바인딩**을 사용하지 않고,  
**JSON 형태로 로그인 요청을 받아** 직접 로그인 처리를 한 경험이 있다.

RESTful API를 적용하려고 `LoginController`를 별도로 작성해,  
로그인 요청을 받아 처리했었다.

하지만 Spring Security를 공부하면서,  
"**DispatcherServlet을 지나 Controller에서 로그인 요청을 직접 처리할 필요가 있을까?**"  
하는 의문이 들기 시작했다.

---

## 2. Controller에서 로그인 처리의 문제점

당시에는 사용자 관련 요청(UserController)에  
`@PostMapping("/login")`으로 로그인 API를 추가하고,  
LoginService 등을 통해 다음과 같이 처리했다.

```java
UsernamePasswordAuthenticationToken authenticationToken = 
    new UsernamePasswordAuthenticationToken(req.getUsername(), req.getPassword());

Authentication authentication = authenticationManager.authenticate(authenticationToken);

SecurityContextHolder.getContext().setAuthentication(authentication);
```

**AuthenticationManager**를 주입받아 인증하고,  
**SecurityContext**에 인증 객체를 저장하는 구조였다.

**하지만 이 방식에는 두 가지 문제가 있다고 생각한다.**

1. **SRP(Single Responsibility Principle) 위배**
   - UserController는 회원가입, 탈퇴 등 "사용자 관리"를 담당하는데,  
     여기에 "인증(Authentication)" 로직까지 섞이게 되어 책임이 명확하지 않다.
   - 인증/권한은 **Spring Security**가 책임져야 할 영역이다.

2. **Spring Security 흐름 무시**
   - Spring Security는 기본적으로  
     **Filter → DispatcherServlet → Controller** 순으로 요청을 흐르게 한다.
   - **Filter가 먼저 요청을 가로채 인증을 담당**하는데,  
     Controller에서 직접 로그인 처리를 하면 이 표준 흐름을 어기게 된다.
   - 특히 인증되지 않은 요청은 Controller까지 도달하기 전에 거부되어야 한다.

---

## 3. 적절한 로그인 처리 방법: Filter 기반 처리

결론적으로,  
**로그인은 Controller가 아니라 Filter**에서 처리해야 한다고 생각하게 됐다.

Spring Security에는 이미  
**UsernamePasswordAuthenticationFilter**라는 강력한 기본 필터가 존재한다.

```java
public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) {
    if (this.postOnly && !request.getMethod().equals("POST")) {
        throw new AuthenticationServiceException("Authentication method not supported");
    }
    String username = this.obtainUsername(request);
    username = (username != null) ? username.trim() : "";
    String password = this.obtainPassword(request);
    password = (password != null) ? password : "";

    UsernamePasswordAuthenticationToken authRequest = 
        new UsernamePasswordAuthenticationToken(username, password);

    this.setDetails(request, authRequest);
    return this.getAuthenticationManager().authenticate(authRequest);
}
```

→ 사용자의 아이디/비밀번호를 입력받아  
**자동으로 Authentication 객체를 생성하고, 인증을 시도하는 로직**이 기본으로 제공된다.

---

## 4. 커스텀 LoginAuthenticationFilter 작성

기본 Filter를 그대로 쓰는 대신,  
**UsernamePasswordAuthenticationFilter를 상속받아**  
JSON 기반 로그인 요청을 처리하는 **커스텀 필터**를 작성했다.

```java
@Slf4j
@RequiredArgsConstructor
public class LoginAuthenticationFilter extends UsernamePasswordAuthenticationFilter {

    private final JwtUtil jwtUtil;
    private final ResponseWrapper responseWrapper;

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws IOException {
        UserLoginDto userLoginDto = new ObjectMapper().readValue(request.getInputStream(), UserLoginDto.class);

        log.info("로그인 요청: {}", userLoginDto.toString());

        UsernamePasswordAuthenticationToken authRequest =
                new UsernamePasswordAuthenticationToken(
                        userLoginDto.getUserName(),
                        userLoginDto.getUserPassword()
                );

        return getAuthenticationManager().authenticate(authRequest);
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response,
                                             FilterChain chain, Authentication authentication) throws IOException {
        UserDetailsImpl userDetails = (UserDetailsImpl) authentication.getPrincipal();
        String userId = userDetails.getUsername();

        String accessToken = jwtUtil.generateAccessToken(userId);
        Users users = userDetails.getUsers();

        JwtToken jwtToken = JwtToken.builder()
                .accessToken(accessToken)
                .userResponseDto(UserResponseDto.of(users))
                .build();

        responseWrapper.convertObjectToResponse(response, jwtToken);
    }
}
```

---

## 5. 결론: Filter를 통한 인증이 SRP를 지키는 방법이다

누군가는  
"커스텀 필터를 작성해도 결국 SRP 위배 아닌가?"  
라고 질문할 수도 있다.

하지만 나는 그렇게 생각하지 않는다.

- Spring Security가 제공하는 **AuthenticationManager**와 인증 흐름을 그대로 사용했고,
- 로그인 책임을 Controller가 아닌 **Filter 레이어에 위임**했기 때문이다.

결과적으로,  
- **서비스 레이어(UserService)**는 인증 로직을 몰라도 되고
- **Security 필터 체인** 안에서 일관된 인증 절차를 유지할 수 있다.

---

## 6. 앞으로의 개발 방향

Spring Boot는 다양한 기능과 편리한 설정을 제공하지만,  
그렇기 때문에 **각 기술을 정확하게 이해하고 적절하게 사용하는 것**이 매우 중요하다고 생각한다.

앞으로 더 좋은 구조를 학습하기 전까지,  
나는 인증(Authentication)과 인가(Authorization)는  
**Filter 레이어에서 처리**하는 방식을 유지할 계획이다.

---

