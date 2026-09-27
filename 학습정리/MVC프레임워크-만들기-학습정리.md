# MVC 프레임워크 만들기 학습 정리 (프론트 컨트롤러 v1 → v5)

> **이 문서의 목표**
> 앞 챕터에서 만든 MVC 패턴에는 **컨트롤러마다 중복되는 코드**라는 한계가 있었다.
> 이 문서는 그 중복을 **"프론트 컨트롤러(Front Controller)"** 라는 구조로 걷어내고, 그것을
> **v1 → v5까지 단계적으로 리팩터링**하며 **스프링 MVC와 똑같은 구조를 내 손으로 완성**하는 과정이다.
>
> **핵심 흐름 한눈에**
> - **v1**: 입구(프론트 컨트롤러)를 하나로 → 요청을 컨트롤러로 분배
> - **v2**: 뷰로 이동하는 중복 코드를 `MyView`로 분리
> - **v3**: 컨트롤러에서 **서블릿 기술(request/response)을 제거** → 순수 자바 컨트롤러 (`ModelView`, `ViewResolver`)
> - **v4**: `ModelView` 생성이 번거로워 → 컨트롤러가 **뷰 이름(String)만 반환**하도록 편의 개선
> - **v5**: 서로 다른 컨트롤러 방식(v3·v4)을 **동시에** 지원 → **어댑터 패턴(HandlerAdapter)**
>
> **이전 문서 연결**: `서블릿-JSP-MVC패턴-학습정리.md`의 "4. MVC 패턴의 한계"에서 예고한 바로 그
> 프론트 컨트롤러다. 거기서 지적한 3가지 중복(뷰 이동 코드·viewPath 접두/접미사·공통 처리)을 여기서 하나씩 해결한다.
> 그리고 이 여정의 종착지가 `서블릿-학습정리.md` 7번의 **`DispatcherServlet`** 이다.

---

## 0. 프론트 컨트롤러 패턴이란?

**입구를 하나로 만드는 것.** 지금까지는 요청 URL마다 서블릿(컨트롤러)이 따로 있었다. 프론트 컨트롤러는
**모든 요청을 먼저 받는 서블릿 하나**를 두고, 그 안에서 "이 요청은 어느 컨트롤러가 처리할지" 정해 넘긴다.

```
[기존]  요청A → 컨트롤러A
        요청B → 컨트롤러B          ← 공통 처리(로그·인증·뷰이동)를 컨트롤러마다 중복
        요청C → 컨트롤러C

[프론트 컨트롤러]
        요청A ┐
        요청B ┼→ [프론트 컨트롤러] → 알맞은 컨트롤러 호출 → 결과로 뷰 처리
        요청C ┘        (공통 처리는 여기서 딱 한 번)
```

### 왜 좋은가
- **공통 처리를 입구 한 곳에** 모을 수 있다 (뷰 이동, 로그, 인증 체크 등). 컨트롤러 중복 제거.
- 각 컨트롤러는 **자기 로직만** 신경 쓰면 된다.
- 프론트 컨트롤러를 제외한 나머지 컨트롤러는 **서블릿을 안 써도 되게** 만들 수 있다. (v3에서 실현)

> 🔗 **이것이 바로 스프링 MVC의 핵심.** 스프링의 `DispatcherServlet`이 정확히 이 프론트 컨트롤러다.
> 지금 v1~v5로 만드는 게 사실상 미니 스프링 MVC다.

---

## v1 — 프론트 컨트롤러 도입

**목표**: 일단 구조부터 만든다. 컨트롤러들을 **공통 인터페이스**로 묶고, 프론트 컨트롤러가 URL을 보고 골라 호출.

### ① 컨트롤러 인터페이스 — 규격 통일
```java
public interface ControllerV1 {
    void process(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException;
}
```
- 모든 컨트롤러가 이 인터페이스를 구현한다. → 프론트 컨트롤러는 **인터페이스 타입으로 다형성 있게** 호출 가능.
- 아직은 `process()`가 `request`, `response`를 그대로 받는다. (서블릿과 비슷)

### ② 개별 컨트롤러 — 기존 서블릿 로직을 그대로 이사
```java
public class MemberSaveControllerV1 implements ControllerV1 {
    private MemberRepository memberRepository = MemberRepository.getInstance();

    @Override
    public void process(HttpServletRequest request, HttpServletResponse response) ... {
        String username = request.getParameter("username");
        int age = Integer.parseInt(request.getParameter("age"));
        Member member = new Member(username, age);
        memberRepository.save(member);
        request.setAttribute("member", member);

        String viewPath = "/WEB-INF/views/save-result.jsp";     // ← 아직 forward를 직접 함
        request.getRequestDispatcher(viewPath).forward(request, response);
    }
}
```
- 앞 챕터의 `MvcMemberSaveServlet`과 내용이 거의 같다. **아직 forward 코드가 컨트롤러 안에 남아있다.** (v2에서 제거)

### ③ 프론트 컨트롤러 — 매핑 & 위임
```java
@WebServlet(name = "frontControllerServletV1", urlPatterns = "/front-controller/v1/*")
public class FrontControllerServletV1 extends HttpServlet {

    private Map<String, ControllerV1> controllerMap = new HashMap<>();  // URL → 컨트롤러

    public FrontControllerServletV1() {   // 생성 시 매핑 등록
        controllerMap.put("/front-controller/v1/members/new-form", new MemberFormControllerV1());
        controllerMap.put("/front-controller/v1/members/save", new MemberSaveControllerV1());
        controllerMap.put("/front-controller/v1/members", new MemberListControllerV1());
    }

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response) ... {
        String requestURI = request.getRequestURI();

        ControllerV1 controller = controllerMap.get(requestURI);   // URL로 컨트롤러 조회
        if (controller == null) {                                  // 없으면 404
            response.setStatus(HttpServletResponse.SC_NOT_FOUND);
            return;
        }
        controller.process(request, response);                     // 위임
    }
}
```

**두 가지 핵심 장치**
- `urlPatterns = "/front-controller/v1/*"` → 끝의 **`/*`가 "이 경로 하위 전부"** 를 뜻한다. `/front-controller/v1/members/save`,
  `/front-controller/v1/members` 등 **모든 요청을 이 서블릿 하나가 다 받는다.** (프론트 컨트롤러의 전제)
- `controllerMap` → **URL을 key, 컨트롤러 객체를 value** 로 저장한 지도. `requestURI`로 꺼내 위임한다.

> 💬 v1의 성과: 흩어져 있던 컨트롤러들이 **하나의 입구 + 하나의 인터페이스**로 통합됐다. 구조의 뼈대 완성.
> 남은 문제: 컨트롤러마다 **`viewPath` + forward 3줄이 여전히 중복**된다. → v2에서 해결.

---

## v2 — View를 분리 (MyView 도입)

**목표**: v1에서 모든 컨트롤러가 반복하던 **forward 코드**(`RequestDispatcher... forward`)를 없앤다.

### 아이디어
"뷰로 이동하는 일"을 담당하는 객체 `MyView`를 만들고, 컨트롤러는 **어디로 갈지(MyView)만 반환**한다.
실제 forward는 프론트 컨트롤러가 대신 호출한다.

### ① MyView — 뷰 이동 전담 객체
```java
public class MyView {
    private String viewPath;
    public MyView(String viewPath) { this.viewPath = viewPath; }

    public void render(HttpServletRequest request, HttpServletResponse response) ... {
        RequestDispatcher dispatcher = request.getRequestDispatcher(viewPath);
        dispatcher.forward(request, response);      // 중복되던 forward가 여기 한 곳으로
    }
}
```

### ② 컨트롤러 — 이제 MyView를 "반환"만
```java
public interface ControllerV2 {
    MyView process(HttpServletRequest request, HttpServletResponse response) ...;   // 반환 타입이 MyView!
}

public class MemberSaveControllerV2 implements ControllerV2 {
    public MyView process(...) {
        // ... 저장, setAttribute ...
        return new MyView("/WEB-INF/views/save-result.jsp");   // forward 안 함! 경로만 담아 반환
    }
}
```

### ③ 프론트 컨트롤러 — 반환받은 MyView로 render 호출
```java
MyView view = controller.process(request, response);
view.render(request, response);      // 여기서 딱 한 번 forward
```

> 💬 v2의 성과: 컨트롤러에서 `RequestDispatcher...forward` 반복이 **완전히 사라졌다.** 뷰 이동은 이제 프론트 컨트롤러 + MyView의 몫.
> 남은 문제: 컨트롤러가 여전히 **`HttpServletRequest`, `HttpServletResponse`에 의존**하고, `setAttribute`로 데이터를 담는다.
> → 컨트롤러를 서블릿 기술에서 **완전히 독립**시키고 싶다. 그리고 `/WEB-INF/views/... .jsp` 전체 경로도 계속 반복된다. → v3.

---

## v3 — 서블릿 종속성 제거 + View 이름 분리 (ModelView, ViewResolver)

**목표(두 마리 토끼)**
1. 컨트롤러에서 `request`/`response`를 **아예 없앤다** → 순수 자바 코드로 만들어 테스트·재사용 쉽게.
2. 컨트롤러는 뷰의 **논리 이름**(`"save-result"`)만 알고, 실제 경로(`/WEB-INF/views/save-result.jsp`)는 프론트 컨트롤러가 완성.

### ① ModelView — 모델(데이터) + 뷰 이름을 담는 그릇
```java
public class ModelView {
    private String viewName;                          // 뷰 논리 이름
    private Map<String, Object> model = new HashMap<>();   // 뷰에 넘길 데이터
    // 생성자, getter/setter ...
}
```
- 앞선 버전은 데이터를 `request.setAttribute()`로 담았다(서블릿 종속). 이제 **`ModelView`의 `model` 맵**에 담는다 → 서블릿과 무관.

### ② 컨트롤러 — request/response가 사라졌다!
```java
public interface ControllerV3 {
    ModelView process(Map<String, String> paramMap);   // 파라미터는 Map으로 받고, ModelView 반환
}

public class MemberSaveControllerV3 implements ControllerV3 {
    public ModelView process(Map<String, String> paramMap) {
        String username = paramMap.get("username");         // request 대신 Map에서 꺼냄
        int age = Integer.parseInt(paramMap.get("age"));
        Member member = new Member(username, age);
        memberRepository.save(member);

        ModelView mv = new ModelView("save-result");        // 논리 이름만!
        mv.getModel().put("member", member);                // 데이터는 model 맵에
        return mv;
    }
}
```
- **이 클래스에 `javax.servlet` import가 하나도 없다.** 순수 자바다. → 단위 테스트도 쉽고, 서블릿 없이도 동작한다.
- 파라미터는 `HttpServletRequest`가 아니라 **`Map<String,String> paramMap`** 으로 받는다. (프론트 컨트롤러가 만들어 넘겨줌)

### ③ 프론트 컨트롤러 — paramMap 생성 + ViewResolver
```java
protected void service(...) {
    // ... 컨트롤러 조회 (v1과 동일) ...
    Map<String, String> paramMap = createParamMap(request);   // request 파라미터 → Map 변환
    ModelView mv = controller.process(paramMap);              // 순수 컨트롤러 호출

    String viewName = mv.getViewName();                       // "save-result"
    MyView view = viewResolver(viewName);                     // → 실제 경로 완성
    view.render(mv.getModel(), request, response);            // model 넘겨 렌더
}

private MyView viewResolver(String viewName) {
    return new MyView("/WEB-INF/views/" + viewName + ".jsp"); // 논리이름 → 물리경로
}

private Map<String, String> createParamMap(HttpServletRequest request) {
    Map<String, String> paramMap = new HashMap<>();
    request.getParameterNames().asIterator()
            .forEachRemaining(name -> paramMap.put(name, request.getParameter(name)));
    return paramMap;
}
```

**핵심 두 가지**
- **`viewResolver`** = 뷰 논리 이름(`save-result`)을 물리 경로(`/WEB-INF/views/save-result.jsp`)로 바꿔주는 것.
  → `/WEB-INF/views/` 접두사와 `.jsp` 접미사 중복이 **여기 한 곳**으로 모였다. (앞 챕터 한계 ②번 해결!)
- **`MyView.render(model, request, response)`** = 오버로딩된 버전. model 맵을 받아 내부에서 `request.setAttribute()`로 옮겨준 뒤 forward.
  ```java
  public void render(Map<String,Object> model, HttpServletRequest req, HttpServletResponse res) {
      model.forEach((key, value) -> req.setAttribute(key, value));   // model → request attribute
      req.getRequestDispatcher(viewPath).forward(req, res);
  }
  ```

> 💬 v3의 성과: 컨트롤러가 **서블릿에서 완전히 독립**했다(순수 자바). viewResolver로 경로 중복도 제거.
> 남은 불편: 컨트롤러가 매번 `new ModelView(...)` 만들고 `getModel().put(...)` 하는 게 **번거롭다.** → v4에서 편의 개선.

---

## v4 — 단순하고 실용적인 컨트롤러

**목표**: v3는 구조가 좋지만 개발자가 **항상 `ModelView`를 생성·반환**해야 해서 손이 많이 간다.
→ 컨트롤러가 **뷰 이름(String)만 반환**하게 하고, 모델은 **파라미터로 받아 채우기만** 하도록 바꾼다.

> 중요한 관점: **기본 구조(v3)는 그대로 두고 "인터페이스만" 개발자에게 편하게** 바꾼 것. 프레임워크가 개발자 편의를 위해
> 하는 전형적인 일이다.

### ① 인터페이스 — model을 인자로 받고, viewName(String) 반환
```java
public interface ControllerV4 {
    String process(Map<String, String> paramMap, Map<String, Object> model);  // model을 받아옴!
}
```

### ② 컨트롤러 — 훨씬 간결해졌다
```java
public class MemberSaveControllerV4 implements ControllerV4 {
    public String process(Map<String, String> paramMap, Map<String, Object> model) {
        String username = paramMap.get("username");
        int age = Integer.parseInt(paramMap.get("age"));
        Member member = new Member(username, age);
        memberRepository.save(member);

        model.put("member", member);   // 넘겨받은 model에 담기만
        return "save-result";          // 뷰 이름만 반환 (ModelView 생성 X)
    }
}
```
- v3의 `ModelView mv = new ModelView("save-result"); mv.getModel().put(...); return mv;` (3줄)
  → v4는 `model.put(...); return "save-result";` (2줄). **더 직관적이고 간결.**

### ③ 프론트 컨트롤러 — model을 만들어 넘겨준다
```java
Map<String, String> paramMap = createParamMap(request);
Map<String, Object> model = new HashMap<>();      // 프론트 컨트롤러가 model을 생성해서

String viewName = controller.process(paramMap, model);   // 넘겨주고, 컨트롤러가 채움
MyView view = viewResolver(viewName);
view.render(model, request, response);
```

> 💬 v4의 성과: 컨트롤러 코드가 눈에 띄게 단순해졌다. 개발자 입장에선 v4가 제일 쓰기 편하다.
> 남은 문제: 이제 컨트롤러 방식이 **v3(ModelView 반환)** 과 **v4(String 반환)** 두 가지가 됐다.
> 한 프론트 컨트롤러가 **둘 다 지원**하려면? 인터페이스가 달라 그냥은 안 된다. → v5 어댑터.

---

## v5 — 유연한 컨트롤러 (어댑터 패턴, HandlerAdapter)

**목표**: v3 컨트롤러와 v4 컨트롤러는 **인터페이스가 다르다**(`ModelView process(Map)` vs `String process(Map,Map)`).
프론트 컨트롤러 하나로 **서로 다른 방식의 컨트롤러를 모두** 처리하고 싶다. → **어댑터 패턴**으로 해결.

### 어댑터 패턴이란
서로 안 맞는 것 사이에 **변환기(어댑터)** 를 끼워 호환시키는 것. (110V 기기를 220V 콘센트에 쓰는 돼지코 어댑터를 떠올리면 된다.)
여기선 "프론트 컨트롤러 ↔ 제각각인 컨트롤러들" 사이에 **핸들러 어댑터**를 둔다.

### ① 어댑터 인터페이스 — MyHandlerAdapter
```java
public interface MyHandlerAdapter {
    boolean supports(Object handler);   // 이 어댑터가 저 컨트롤러(handler)를 처리할 수 있나?
    ModelView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws ServletException, IOException;   // 실제로 실행하고 결과를 ModelView로 통일해 반환
}
```
- 컨트롤러를 **`Object handler`** 라는 가장 넓은 타입으로 받는다. → v3든 v4든 무엇이든 담을 수 있음.

### ② 구현 어댑터 — 버전별로 하나씩
```java
public class ControllerV3HandlerAdapter implements MyHandlerAdapter {
    public boolean supports(Object handler) {
        return (handler instanceof ControllerV3);        // V3 컨트롤러면 내가 처리
    }
    public ModelView handle(..., Object handler) ... {
        ControllerV3 controller = (ControllerV3) handler; // 캐스팅
        Map<String, String> paramMap = createParamMap(request);
        return controller.process(paramMap);              // V3 방식으로 호출 (이미 ModelView 반환)
    }
}

public class ControllerV4HandlerAdapter implements MyHandlerAdapter {
    public boolean supports(Object handler) {
        return (handler instanceof ControllerV4);
    }
    public ModelView handle(..., Object handler) ... {
        ControllerV4 controller = (ControllerV4) handler;
        Map<String, String> paramMap = createParamMap(request);
        HashMap<String, Object> model = new HashMap<>();
        String viewName = controller.process(paramMap, model);  // V4 방식 호출 (String 반환)

        ModelView mv = new ModelView(viewName);      // ★ V4의 결과(String+model)를
        mv.setModel(model);                          //   ModelView로 "변환"해서
        return mv;                                   //   반환 형식을 통일! (어댑터의 핵심)
    }
}
```
- **V4 어댑터가 하는 일이 어댑터의 정수다.** V4는 `String`을 반환하는데, 어댑터가 그것을 `ModelView`로 **변환**해준다.
  → 덕분에 프론트 컨트롤러는 v3든 v4든 **언제나 `ModelView`를 돌려받는다.** (형식 통일)

### ③ 프론트 컨트롤러 V5 — 매핑 + 어댑터 목록
```java
@WebServlet(name = "frontControllerServletV5", urlPatterns = "/front-controller/v5/*")
public class FrontControllerServletV5 extends HttpServlet {

    private final Map<String, Object> handlerMappingMap = new HashMap<>();  // URL → 핸들러(Object!)
    private final List<MyHandlerAdapter> handlerAdapters = new ArrayList<>();

    public FrontControllerServletV5() {
        initHandlerMappingMap();   // v3, v4 컨트롤러를 모두 등록 (URL로 구분)
        initHandlerAdapters();     // 어댑터 2개 등록
    }

    protected void service(...) {
        Object handler = getHandler(request);        // ① URL로 핸들러 조회
        if (handler == null) { /* 404 */ return; }

        MyHandlerAdapter adapter = getHandlerAdapter(handler);   // ② 이 핸들러를 처리할 어댑터 찾기
        ModelView mv = adapter.handle(request, response, handler); // ③ 어댑터가 실행 → ModelView 통일

        String viewName = mv.getViewName();          // ④ 뷰 이름 → 물리 경로 → 렌더
        MyView view = viewResolver(viewName);
        view.render(mv.getModel(), request, response);
    }

    private MyHandlerAdapter getHandlerAdapter(Object handler) {
        for (MyHandlerAdapter adapter : handlerAdapters) {   // 등록된 어댑터들에게
            if (adapter.supports(handler)) return adapter;   // "너 이거 처리 가능?" 물어봄
        }
        throw new IllegalArgumentException("handler adapter를 찾을 수 없습니다. handler=" + handler);
    }
    // getHandler(), viewResolver() 는 이전과 유사
}
```

**동작 흐름 정리**
```
요청 → 프론트컨트롤러V5
   ① 핸들러 조회      handlerMappingMap.get(URI)   → V3 or V4 컨트롤러 (Object)
   ② 어댑터 선택      supports()로 맞는 어댑터 찾기  → V3Adapter or V4Adapter
   ③ 실행+변환        adapter.handle(...)          → 어떤 컨트롤러든 ModelView로 통일 반환
   ④ 뷰 처리          viewResolver → MyView.render
```

> 💬 v5의 성과: **v3, v4 컨트롤러를 한 프론트 컨트롤러가 동시에 지원.** 게다가 나중에 v6, v7 같은 새 방식이 나와도
> **어댑터 하나만 추가**하면 프론트 컨트롤러 본체는 안 건드려도 된다. → **확장에 열려 있고(OCP) 변경에 닫혀 있는** 좋은 구조.

---

## 정리 — 각 단계가 무엇을 해결했나

| 버전 | 도입한 것 | 해결한 문제 |
|------|-----------|-------------|
| **v1** | 프론트 컨트롤러 + 공통 인터페이스(`ControllerV1`) | 입구 통합, 컨트롤러 구조화 |
| **v2** | `MyView` (뷰 반환) | forward 코드 중복 제거 |
| **v3** | `ModelView` + `ViewResolver` | 컨트롤러의 **서블릿 종속 제거**, viewPath 중복 제거 |
| **v4** | 뷰 이름(String) 반환 + model 파라미터 | 컨트롤러 작성 **편의성** 향상 |
| **v5** | `MyHandlerAdapter` (어댑터 패턴) | 서로 다른 컨트롤러 방식 **동시 지원 + 확장성** |

**리팩터링의 사고법**: 한 버전이 문제를 해결하면 다음 불편이 드러나고, 그것이 다음 버전의 목표가 된다.
절대 한 번에 완성하지 않는다. **작동하는 상태를 유지하며 조금씩 개선**하는 이 흐름 자체가 실무 리팩터링의 축소판이다.

---

## 우리가 만든 것 = 스프링 MVC (구조 대응표)

이 v5 구조는 **이름만 빼면 스프링 MVC와 똑같다.** 우리가 손으로 만든 것이 스프링에선 다음에 대응한다.

| 우리가 만든 것 | 스프링 MVC | 역할 |
|----------------|-----------|------|
| `FrontControllerServletV5` | **`DispatcherServlet`** | 모든 요청을 받는 프론트 컨트롤러 |
| `handlerMappingMap` | **`HandlerMapping`** | URL → 핸들러(컨트롤러) 찾기 |
| `MyHandlerAdapter` | **`HandlerAdapter`** | 다양한 방식의 핸들러를 실행 |
| `viewResolver()` | **`ViewResolver`** | 뷰 논리 이름 → 실제 뷰 |
| `MyView` | **`View`** | 뷰 렌더링 |
| `ModelView` | **`ModelAndView`** | 모델 + 뷰 정보 |

> 🔗 그래서 `서블릿-학습정리.md` 7번에서 "`DispatcherServlet`이 모든 요청을 받아 알맞은 컨트롤러로 분배하는 프론트 컨트롤러"라고
> 했던 그 말의 **실체를 이제 완전히 이해**하게 된다. 스프링 MVC를 쓸 때 `HandlerMapping`, `HandlerAdapter`, `ViewResolver`라는
> 용어가 나오면, 지금 v5에서 내 손으로 만든 그것들을 떠올리면 된다.
>
> **다음 챕터부터는 진짜 스프링 MVC**를 쓴다. 지금까지 밑바닥을 직접 만들어봤기 때문에, 스프링이 어떤 수고를 대신 해주는지
> 훤히 보이는 상태로 출발할 수 있다.
