# 서블릿 · JSP · MVC 패턴 학습 정리

> **이 문서의 목표**
> 똑같은 "회원 관리 웹앱"을 **서블릿 → JSP → MVC 패턴** 세 가지 방식으로 차례로 만들어 보면서,
> **각 방식이 어떤 고통을 겪고, 그 고통을 다음 방식이 어떻게 해결하는지**의 흐름을 이해한다.
>
> **큰 줄거리 한 줄 요약**
> - **서블릿**: 자바 코드 안에 HTML을 문자열로 찍음 → *뷰(화면)가 지옥*
> - **JSP**: HTML 안에 자바 코드를 넣음 → *화면은 편해졌지만, 비즈니스 로직과 뷰가 한 파일에 뒤섞임*
> - **MVC**: 로직은 서블릿(Controller), 화면은 JSP(View)로 **역할을 분리** → 지금까지의 문제 해결
> - 그런데 MVC도 컨트롤러마다 **중복 코드**가 생김 → 다음 챕터(프론트 컨트롤러)로 이어짐
>
> **이전 문서 연결**: `서블릿-학습정리.md`의 18번에서 "자바로 HTML 찍는 건 너무 불편해서 템플릿 엔진(JSP)이 나왔다"고
> 예고했는데, **그 JSP가 바로 이 문서의 주인공**이다. 그때 느낀 불편함을 여기서 실제로 겪고 해결한다.

---

# 0. 공통 준비물 — 회원 도메인과 저장소

세 가지 방식 모두 **같은 회원 도메인**을 쓴다. 방식만 바뀔 뿐 다루는 데이터는 동일하다.

## 회원 객체 — Member
```java
@Getter @Setter
public class Member {
    private Long id;          // 저장소가 부여하는 식별자
    private String username;
    private int age;

    public Member() {}
    public Member(String username, int age) {   // id 없이 생성 (저장 시점에 id 부여)
        this.username = username;
        this.age = age;
    }
}
```
- `id`는 회원이 만들 때 정하는 게 아니라 **저장소에 저장되는 순간 자동으로 매겨진다.** 그래서 생성자엔 `username`, `age`만 받는다.
- 기본 생성자(`Member()`)도 같이 둔 건, 프레임워크·라이브러리가 객체를 만들 때 기본 생성자를 필요로 하는 경우가 많기 때문.

## 회원 저장소 — MemberRepository (싱글톤)
```java
public class MemberRepository {

    private static Map<Long, Member> store = new HashMap<>();   // 실제 저장 공간 (임시, 메모리)
    private static long sequence = 0L;                          // id 자동 증가용

    private static final MemberRepository instance = new MemberRepository();  // 유일한 인스턴스
    public static MemberRepository getInstance() { return instance; }         // 이걸로만 접근
    private MemberRepository() {}                               // 외부에서 new 금지

    public Member save(Member member) {
        member.setId(++sequence);      // id 부여
        store.put(member.getId(), member);
        return member;
    }
    public Member findById(Long id) { return store.get(id); }
    public List<Member> findAll() { return new ArrayList<>(store.values()); }
    public void clearStore() { store.clear(); }
}
```

### 왜 싱글톤(딱 1개)으로 만들었나 — 이 패턴을 뜯어보자
저장소는 **애플리케이션 전체에서 하나만 존재**해야 한다. 저장소가 여러 개면 데이터가 여기저기 흩어지니까.
그래서 **싱글톤 패턴**을 직접 구현했다. 세 가지 장치가 한 세트다.

| 장치 | 코드 | 역할 |
|------|------|------|
| ① private 생성자 | `private MemberRepository() {}` | **외부에서 `new` 못 하게** 막음 |
| ② static 인스턴스 | `private static final ... instance = new ...()` | 클래스 로딩 시 **딱 1개** 미리 생성 |
| ③ 공개 접근 메서드 | `public static getInstance()` | 그 1개를 꺼내 쓰는 **유일한 통로** |

→ 그래서 어디서든 `MemberRepository.getInstance()`를 부르면 **항상 같은 저장소**를 얻는다.

> 🔗 **`서블릿-학습정리.md` 4번(서블릿 싱글톤)과 연결.** 서블릿도, 스프링 빈도 기본이 싱글톤이었다.
> 여기 저장소도 싱글톤. "하나만 만들어 공유한다"는 같은 사상이다.
>
> 💡 실무에서는 이렇게 손으로 싱글톤을 짜지 않는다. **스프링 컨테이너가 빈을 싱글톤으로 관리**해주기 때문
> (`@Repository` 붙이면 끝). 지금 직접 만드는 건 스프링이 없을 때 무슨 일이 벌어지는지 체험하는 학습용이다.

> ⚠️ 코드 주석에도 있듯 **이 저장소는 동시성 문제를 고려하지 않았다.** `HashMap`, `long sequence`는
> 여러 스레드가 동시에 건드리면 값이 깨질 수 있다. 실무라면 `ConcurrentHashMap`, `AtomicLong`을 쓴다.
> (학습 단계라 단순하게 둔 것)

---

# 1단계. 서블릿으로 만들기 — "자바 안에 HTML" 의 고통

`web.servlet` 패키지. 서블릿 3개로 회원 등록 폼 → 저장 → 목록을 구현한다.

| 서블릿 | URL | 하는 일 |
|--------|-----|---------|
| `MemberFormServlet` | `/servlet/members/new-form` | 회원 등록 **폼 HTML** 출력 |
| `MemberSaveServlet` | `/servlet/members/save` | 폼 데이터 받아 **저장** 후 결과 HTML 출력 |
| `MemberListServlet` | `/servlet/members` | 저장된 회원 **목록 HTML** 출력 |

## MemberFormServlet — 폼 화면 출력
```java
@WebServlet(name = "memberFormServlet", urlPatterns = "/servlet/members/new-form")
public class MemberFormServlet extends HttpServlet {

    private MemberRepository memberRepository = MemberRepository.getInstance();

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {

        response.setContentType("text/html");
        response.setCharacterEncoding("utf-8");

        PrintWriter w = response.getWriter();
        w.write("<!DOCTYPE html>\n" +
                "<html>\n" +
                "<head>\n" +
                "    <meta charset=\"UTF-8\">\n" +
                "    <title>Title</title>\n" +
                "</head>\n" +
                "<body>\n" +
                "<form action=\"/servlet/members/save\" method=\"post\">\n" +   // 저장 서블릿으로 POST
                "    username: <input type=\"text\" name=\"username\" />\n" +
                "    age:      <input type=\"text\" name=\"age\" />\n" +
                "    <button type=\"submit\">전송</button>\n" +
                "</form>\n" +
                "</body>\n" +
                "</html>\n");
    }
}
```
- 하는 일은 단순하다. **HTML 폼을 문자열로 만들어 `response`에 써서 내보낸다.** (`서블릿-학습정리.md` 18번의 그 방식)
- 폼의 `action="/servlet/members/save"` → 전송 버튼을 누르면 `MemberSaveServlet`으로 데이터가 간다.

## MemberSaveServlet — 저장 + 결과 화면
```java
@WebServlet(name = "memberSaveServlet", urlPatterns = "/servlet/members/save")
public class MemberSaveServlet extends HttpServlet {

    private MemberRepository memberRepository = MemberRepository.getInstance();

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {

        // ① 데이터 꺼내기 (서블릿 정리 14번: 파라미터 조회)
        String username = request.getParameter("username");
        int age = Integer.parseInt(request.getParameter("age"));   // 파라미터는 String → int 변환 필요

        // ② 비즈니스 로직 (저장)
        Member member = new Member(username, age);
        memberRepository.save(member);

        // ③ 결과 화면 그리기 (동적 데이터를 HTML에 끼워 넣음)
        response.setContentType("text/html");
        response.setCharacterEncoding("utf-8");
        PrintWriter w = response.getWriter();
        w.write("<html>\n" +
                "<head><meta charset=\"UTF-8\"></head>\n" +
                "<body>\n성공\n" +
                "<ul>\n" +
                "    <li>id="       + member.getId()       + "</li>\n" +   // ← 자바 변수를 문자열에 이어 붙임
                "    <li>username=" + member.getUsername() + "</li>\n" +
                "    <li>age="      + member.getAge()       + "</li>\n" +
                "</ul>\n" +
                "<a href=\"/index.html\">메인</a>\n" +
                "</body>\n</html>");
    }
}
```
- `request.getParameter()`로 값 꺼내는 건 이전 문서에서 배운 그대로다.
- 주목할 점: **`Integer.parseInt()`.** 파라미터는 항상 문자열(`"20"`)로 오므로 숫자로 쓰려면 직접 변환해야 한다.
- ③에서 `"id=" + member.getId()`처럼 **자바 변수를 HTML 문자열 사이에 끼워** 동적 화면을 만든다.

## MemberListServlet — 목록 (반복 출력)
```java
@WebServlet(name = "memberListServlet", urlPatterns = "/servlet/members")
public class MemberListServlet extends HttpServlet {
    // ...
    protected void service(...) {
        List<Member> members = memberRepository.findAll();
        // ... <table> 헤더 출력 ...
        for (Member member : members) {              // 회원 수만큼 <tr> 반복 출력
            w.write("    <tr>");
            w.write("        <td>" + member.getId() + "</td>");
            w.write("        <td>" + member.getUsername() + "</td>");
            w.write("        <td>" + member.getAge() + "</td>");
            w.write("    </tr>");
        }
        // ...
    }
}
```
- `for`문으로 회원 수만큼 `<tr>`을 찍는다. 반복되는 화면을 자바 반복문으로 처리.

## 🔥 1단계의 결론 — "이건 못 해먹겠다"
코드를 보면 딱 느껴진다. **자바 코드가 주인이고 HTML이 문자열 손님**이다. 문제가 한둘이 아니다.

- `w.write("...\n")` 의 **따옴표·역슬래시·줄바꿈 지옥**. HTML 태그 하나 고치려면 자바 문자열을 고쳐야 함.
- 화면이 어떻게 생겼는지 **눈에 안 들어온다.** (디자이너에게 넘길 수도 없음)
- HTML이 조금만 복잡해져도 코드가 폭발한다.

> 💬 핵심 통찰: **"화면(뷰)을 만드는 데 자바가 주인공인 게 문제"** 다. 우리가 원하는 건 반대다 —
> **HTML이 주인공이고, 바뀌는 데이터만 자바로 살짝 끼워 넣는 것.** 그 발상의 전환이 바로 다음 단계, JSP다.

---

# 2단계. JSP로 만들기 — "HTML 안에 자바" 로 뒤집기

**JSP(Java Server Pages)** = HTML 파일을 기본으로 두고, 필요한 곳에만 자바 코드를 넣는 **템플릿 엔진**.
1단계와 **주객이 완전히 뒤바뀐다.** HTML이 주인이고 자바가 손님.

> 📦 이 프로젝트는 JSP를 쓰려고 `build.gradle`에 `tomcat-embed-jasper`(JSP 엔진)를 추가했고, 배포 형태도 `war`다.
> (`서블릿-학습정리.md` 6번 WAR/JAR 참고 — JSP를 쓰는 전통적 구성)

### JSP 기본 문법 3종 (먼저 알고 가자)
| 문법 | 이름 | 하는 일 |
|------|------|---------|
| `<%@ page ... %>` | 지시자(directive) | 페이지 설정 (인코딩, import 등) |
| `<% 자바코드 %>` | 스크립틀릿(scriptlet) | 자바 코드를 그대로 실행 |
| `<%= 값 %>` | 표현식(expression) | 자바 값을 화면에 출력 (`out.print`와 같음) |

## new-form.jsp — 폼 화면
```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head><title>Title</title></head>
<body>
<form action="/jsp/members/save.jsp" method="post">
    username: <input type="text" name="username" />
    age:      <input type="text" name="age" />
    <button type="submit">전송</button>
</form>
</body>
</html>
```
- 첫 줄 `<%@ page ... %>`만 빼면 **그냥 순수 HTML**이다. 1단계의 `w.write("...")` 지옥과 비교하면 천지 차이.
- **화면이 눈에 그대로 보인다.** 이게 JSP의 가장 큰 장점.

## save.jsp — 저장 (여기서 문제가 드러난다)
```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ page import="hello.servlet.domain.member.Member" %>
<%@ page import="hello.servlet.domain.member.MemberRepository" %>
<%
    // ↓↓↓ 여기가 전부 "자바 코드" — 비즈니스 로직 ↓↓↓
    MemberRepository memberRepository = MemberRepository.getInstance();

    String username = request.getParameter("username");   // request/response 그냥 사용 가능
    int age = Integer.parseInt(request.getParameter("age"));

    Member member = new Member(username, age);
    memberRepository.save(member);
%>
<html>
<head><title>Title</title></head>
<body>
성공
<ul>
    <li>id=<%=member.getId()%></li>            <%-- 자바 값을 화면에 출력 --%>
    <li>username=<%=member.getUsername()%></li>
    <li>age=<%=member.getAge()%></li>
</ul>
<a href="/index.html">메인</a>
</body>
</html>
```
- `<% ... %>` 안은 **서블릿의 service() 안에 있던 자바 코드가 그대로** 들어와 있다.
- `request`, `response`를 별도 선언 없이 바로 쓸 수 있다. (JSP가 내부적으로 서블릿으로 변환되기 때문 — 아래 박스)
- `<%= member.getId() %>` → 자바 값을 화면에 출력.

> 🧩 **JSP의 정체: 사실은 JSP도 서블릿이다.**
> JSP 파일은 서버에서 최초 호출 시 **자바 서블릿 코드로 변환·컴파일**된다. 우리가 1단계에서 손으로 짠 그
> `w.write("<html>...")` 서블릿을, **JSP 엔진이 대신 자동 생성**해주는 것뿐이다. 그래서 원리는 똑같고 편하기만 하다.

## members.jsp — 목록
```jsp
<%@ page ... %>
<% List<Member> members = memberRepository.findAll(); %>
...
<tbody>
    <%
        for (Member member : members) {          // JSP 안에서 자바 for문
            out.write("    <tr>");
            out.write("        <td>" + member.getId() + "</td>");
            // ...
        }
    %>
</tbody>
```
- 목록은 결국 `for`문으로 `out.write(...)` — **1단계 서블릿과 거의 똑같은 모양**이 JSP 안에 다시 등장했다.

## 🔥 2단계의 결론 — 뷰는 해결됐지만 새 문제가 생겼다
JSP 덕분에 "화면 만들기"는 편해졌다. 그런데 `save.jsp`를 다시 보자.

- **위쪽 절반**: 회원을 저장하는 **비즈니스 로직** (자바)
- **아래쪽 절반**: 결과를 보여주는 **화면** (HTML)

즉 **하나의 JSP 파일에 "일 처리"와 "화면"이 뒤섞여 있다.** 이게 왜 문제냐면:
- 코드가 커지면 자바 로직과 HTML이 엉켜 **읽기도, 고치기도 어렵다.**
- 화면만 바꾸고 싶은데 비즈니스 로직 코드까지 봐야 한다. (역할이 안 나뉨)
- 비즈니스 로직 테스트도 어렵다. (JSP에 묶여 있으니)

> 💬 핵심 통찰: 1단계는 "자바가 화면을 침범"했고, 2단계는 "화면이 비즈니스 로직을 떠안았다."
> **결국 문제의 본질은 "일 처리와 화면이 한 곳에 섞여 있다"는 것.** 그럼 답은 명확하다 — **둘을 분리하자.**
> 그게 바로 **MVC 패턴**이다.

---

# 3단계. MVC 패턴 — 역할을 나누자

## MVC란? — 관심사의 분리(separation of concerns)
| 구성요소 | 역할 | 이 프로젝트에서 |
|----------|------|----------------|
| **Model** (모델) | 뷰에 전달할 **데이터** | `request`의 attribute에 담은 `member`, `members` |
| **View** (뷰) | **화면**을 그리는 것만 담당 | JSP (`WEB-INF/views/*.jsp`) |
| **Controller** (컨트롤러) | 파라미터 검증·**비즈니스 로직**·모델에 데이터 담기 | 서블릿 (`MvcMember...Servlet`) |

**한 문장으로:** 컨트롤러(서블릿)가 **일을 처리하고 데이터를 모델에 담아** 뷰(JSP)에 넘기면,
뷰는 받은 데이터로 **화면만** 그린다. 서로 남의 일에 간섭하지 않는다.

```
[클라이언트] → HTTP 요청 → [Controller(서블릿)]
                              │  ① 파라미터 조회, 비즈니스 로직 실행
                              │  ② 결과 데이터를 Model(request)에 담기
                              │  ③ View(JSP)로 forward
                              ▼
                          [View(JSP)]  ── Model 데이터로 화면 생성 → HTML 응답 → [클라이언트]
```

> 💬 왜 컨트롤러를 서블릿으로, 뷰를 JSP로? 각자 **제일 잘하는 일**을 시키는 것이다.
> 서블릿은 자바 로직에 강하고(→ 컨트롤러), JSP는 HTML 그리기에 강하다(→ 뷰). 억지로 한쪽에 다 시키던 게 1·2단계였다.

## 컨트롤러 → 뷰로 넘기는 2가지 핵심 도구

### ① forward — 서버 내부에서 다른 자원(JSP)에게 넘기기
```java
String viewPath = "/WEB-INF/views/new-form.jsp";
RequestDispatcher dispatcher = request.getRequestDispatcher(viewPath);
dispatcher.forward(request, response);
```
- `forward` = **서버 내부에서** 컨트롤러 → JSP로 제어를 넘기는 것. 같은 요청(`request`/`response`)을 그대로 전달한다.

**⚠️ forward vs redirect — 자주 나오는 시험 포인트**

| 구분 | forward | redirect |
|------|---------|----------|
| 위치 | **서버 내부**에서 호출 | 클라이언트를 거쳐 **다시 요청** |
| 요청 횟수 | 1번 (client 모름) | 2번 (client가 새 URL로 재요청) |
| URL 변화 | **안 바뀜** | 바뀜 (새 주소로 이동) |
| request 데이터 | **그대로 유지** | 사라짐 (새 요청이라) |

→ MVC에서 컨트롤러가 모델(request)을 담아 뷰로 넘길 땐 **request가 유지돼야 하므로 반드시 forward**.

### ② Model — request의 attribute에 데이터 담기
```java
request.setAttribute("member", member);      // 컨트롤러: 데이터를 request에 저장
// ... 뷰(JSP)에서 꺼내 씀 ...
```
- `request` 객체를 **데이터 보관함(Model)** 으로 쓴다. `setAttribute(이름, 값)`으로 담고, 뷰에서 그 이름으로 꺼낸다.
- forward는 같은 request를 넘기므로, 컨트롤러가 담은 데이터를 JSP가 그대로 받는다.

## MVC 적용 코드

### MvcMemberFormServlet — 폼 (로직 없이 뷰로만)
```java
@WebServlet(name = "mvcMemberFormServlet", urlPatterns = "/servlet-mvc/members/new-form")
public class MvcMemberFormServlet extends HttpServlet {
    protected void service(HttpServletRequest request, HttpServletResponse response) ... {
        String viewPath = "/WEB-INF/views/new-form.jsp";
        RequestDispatcher dispatcher = request.getRequestDispatcher(viewPath);
        dispatcher.forward(request, response);        // 그냥 폼 JSP로 넘김
    }
}
```
- 폼 화면은 보여줄 데이터가 없으니 **비즈니스 로직 없이 JSP로 forward만** 한다.

### MvcMemberSaveServlet — 저장 (로직 + 모델 + 뷰)
```java
@WebServlet(name = "mvcMemberSaveServlet", urlPatterns = "/servlet-mvc/members/save")
public class MvcMemberSaveServlet extends HttpServlet {
    private MemberRepository memberRepository = MemberRepository.getInstance();

    protected void service(HttpServletRequest request, HttpServletResponse response) ... {
        // ① 비즈니스 로직 (컨트롤러의 일)
        String username = request.getParameter("username");
        int age = Integer.parseInt(request.getParameter("age"));
        Member member = new Member(username, age);
        memberRepository.save(member);

        // ② Model에 데이터 담기
        request.setAttribute("member", member);

        // ③ View로 forward
        String viewPath = "/WEB-INF/views/save-result.jsp";
        request.getRequestDispatcher(viewPath).forward(request, response);
    }
}
```
- 2단계 `save.jsp`에 뒤섞여 있던 자바 로직이 **전부 컨트롤러(서블릿)로 이사**했다. 그리고 결과 데이터만 모델에 담아 뷰로 넘긴다.

### save-result.jsp — 뷰 (화면만! 자바 코드 없음)
```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<body>
성공
<ul>
    <li>id=${member.id}</li>              <%-- EL로 모델 데이터 출력 --%>
    <li>username=${member.username}</li>
    <li>age=${member.age}</li>
</ul>
<a href="/index.html">메인</a>
</body>
</html>
```
- **`<% %>` 자바 코드가 하나도 없다!** 2단계 `save.jsp`와 비교하면 이게 MVC의 성과다. 뷰는 순수하게 화면만.
- `${member.id}` = **EL(Expression Language)**. 아래에서 설명.

## 뷰(JSP)를 깔끔하게 해주는 두 도구 — EL & JSTL

### EL(표현식 언어) — `${...}`
```jsp
<li>id=${member.id}</li>
```
- 2단계의 `<%= member.getId() %>`(자바 코드)를 **`${member.id}`** 로 짧고 깔끔하게 대체.
- `${member.id}`는 내부적으로 모델에서 `member`를 찾아 **`getId()`를 호출**한다. (그래서 `Member`에 getter가 필요했던 것)

### JSTL `<c:forEach>` — 반복을 태그로
```jsp
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
...
<c:forEach var="item" items="${members}">
    <tr>
        <td>${item.id}</td>
        <td>${item.username}</td>
        <td>${item.age}</td>
    </tr>
</c:forEach>
```
- 2단계 `members.jsp`의 `<% for(...) { out.write(...) } %>` **자바 반복문**을, JSTL `<c:forEach>` **태그**로 대체.
- `items="${members}"`(모델의 리스트)를 돌면서 `var="item"`으로 하나씩 꺼내 화면을 그린다. **자바 코드가 사라져 훨씬 읽기 쉽다.**

## 🗂️ 왜 JSP를 `WEB-INF/` 안에 뒀나? — 직접 접근 차단
뷰 JSP들은 `src/main/webapp/WEB-INF/views/` 안에 있다. 여기엔 중요한 이유가 있다.

- **`WEB-INF` 폴더 안의 파일은 브라우저가 URL로 직접 접근할 수 없다.** (서블릿 규약)
- 즉 사용자가 `.../WEB-INF/views/save-result.jsp`를 주소창에 쳐도 **안 열린다.**
- → **반드시 컨트롤러(서블릿)를 거쳐 forward로만** 접근하게 강제하는 것.
- 만약 밖에 두면 컨트롤러의 비즈니스 로직(저장 등)을 건너뛰고 뷰에 바로 접근할 수 있어 위험하다.

> 💬 즉 `WEB-INF`는 "이 화면은 **컨트롤러를 통해서만** 보여줄 거야"라는 강제 장치다. MVC의 흐름을 깨지 못하게 막는 울타리.

## 참고: 상대경로 폼 action
```jsp
<!-- new-form.jsp -->
<form action="save" method="post">   <!-- 절대경로 아님! -->
```
- `action="save"`는 **상대경로**다. 현재 URL(`/servlet-mvc/members/new-form`)이 속한 계층(`/servlet-mvc/members/`)
  기준으로 `save`가 붙어 → 최종 `/servlet-mvc/members/save`로 전송된다.
- (다만 상대경로는 현재 경로에 의존해서 깨지기 쉬워, 실무에선 상황에 맞게 절대경로도 많이 쓴다.)

---

# 4. MVC 패턴의 한계 — 그래도 아직 불편하다

MVC로 역할은 잘 나눴다. 그런데 컨트롤러 3개(`MvcMember...Servlet`)를 나란히 보면 **똑같은 코드가 계속 반복**된다.

### 반복 1) 뷰로 이동하는 코드가 매번 똑같다
```java
String viewPath = "/WEB-INF/views/xxx.jsp";
RequestDispatcher dispatcher = request.getRequestDispatcher(viewPath);
dispatcher.forward(request, response);
```
- 모든 컨트롤러가 이 **3줄짜리 forward 코드**를 복붙하고 있다. viewPath만 다를 뿐.

### 반복 2) viewPath의 접두사·접미사가 중복
- `/WEB-INF/views/` (접두사)와 `.jsp` (접미사)가 **모든 컨트롤러에 하드코딩**돼 있다.
- 나중에 뷰 폴더 위치를 바꾸면? **모든 컨트롤러를 다 고쳐야** 한다.

### 반복 3) 공통 처리가 어렵다
- 로그를 남기거나, 로그인 체크를 하거나 하는 **공통 작업을 넣으려면 모든 컨트롤러에 똑같이** 추가해야 한다.
- 컨트롤러가 많아질수록 이 중복은 재앙이 된다.

### 그리고 매번 `HttpServletRequest`, `HttpServletResponse`를 다 받아야 한다
- 어떤 컨트롤러는 사실 request/response가 필요 없는데도 서블릿이라 무조건 받아야 한다.

## 💡 그래서 다음은? — 공통 부분을 한 곳으로 모으자
이 "공통 코드 중복" 문제의 해결책이 **프론트 컨트롤러(Front Controller) 패턴**이다.
- **입구를 하나로** 만들어서(컨트롤러 하나가 모든 요청을 먼저 받음), 공통 처리(forward, 로그 등)를 **거기서 한 번만** 하고,
- 실제 로직만 각 컨트롤러에 위임하는 구조.
- → 이게 바로 **스프링 MVC의 `DispatcherServlet`** 이 하는 일이다. (`서블릿-학습정리.md` 7번에서 예고한 그 프론트 컨트롤러!)

**다음 챕터(MVC 프레임워크 만들기)에서 이 프론트 컨트롤러를 v1 → v5로 단계별로 직접 만들며, 스프링 MVC의 구조를 완성해간다.**

---

# 정리 — 세 방식의 진화 한눈에 보기

| | 1단계 서블릿 | 2단계 JSP | 3단계 MVC |
|---|---|---|---|
| 주인공 | **자바** 안에 HTML | **HTML** 안에 자바 | 역할 **분리** |
| 화면 만들기 | `w.write("...")` 지옥 | 순수 HTML처럼 편함 | JSP + EL/JSTL로 깔끔 |
| 비즈니스 로직 위치 | 서블릿 | JSP 안에 뒤섞임 😱 | 컨트롤러(서블릿)로 분리 ✅ |
| 남은 문제 | 뷰가 지옥 | 로직·뷰 뒤섞임 | 컨트롤러 **중복 코드** |
| 해결 방향 | → JSP | → MVC | → **프론트 컨트롤러**(다음 챕터) |

**흐름의 본질:** 문제를 하나 해결하면 다음 문제가 드러나고, 그 해결이 다음 단계가 된다.
이 사슬의 끝에 **스프링 MVC**가 있다. 지금 우리는 스프링이 왜 그런 구조가 되었는지를 **바닥부터 따라 올라가는 중**이다.
