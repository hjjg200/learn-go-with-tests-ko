# Context

**[이 챕터의 모든 코드를 이 곳에서 볼 수 있다.](https://github.com/quii/learn-go-with-tests/tree/main/context)**

소프트웨어(software)는 종종 오래 가동하고(long-running) 리소스를 많이 점유하는(resource-intensive) 프로세스(혹은 고루틴)를 시작한다. 만약 프로세스를 시작한 action이 어떠한 이유로 인하여 취소되거나 오류를 일으킬 경우(fails), 프로그램에서(your application) 실행된 프로세스들을 일관된 방법으로 멈춰줘야한다.

이것을 고려하지 않는다면(don't manage), 곧 당신이 자랑스러워 하는 그 멋진(snappy) 고 어플리케이션의 성능 문제를 디버그하는 데에 어려움을 겪게 될 것이다.

이 챕터에서는 우리는 `context` 패키지의 도움을 받아 오래 가동하는 프로세스를 관리해 볼 것이다.

첫 예제는 응답으로 보낼 어떤 데이터를 가져오기 위해 잠재적으로 오래 가동할 수도 있는 프로세스를 시작하는 상투적인(classic) 웹 서버이다.

데이터를 가져오는 도중에 유저가 요청을 취소하는 경우를 상정해 볼 것이며, 우리는 이 때 프로세스가 중단될 수 있도록 할 것이다.

에러나 예외가 발생하지 않을 happy path 코드를 준비했다. 아래는 우리의 서버 코드이다.

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, store.Fetch())
	}
}
```

`Server` 함수는 `Store` 인수를 받은 뒤 `http.HandlerFunc` 를 리턴한다. Store 는 다음과 같이 정의되어 있다:

```go
type Store interface {
	Fetch() string
}
```

리턴된 함수는 `store`의 `Fetch` 메소드를 통해 데이터를 얻은 뒤 응답에 해당 데이터를 작성한다.

아래는 이 테스트에 쓰인 `Store`의 나머지 부분(stub, or 구현 부분)이다.

```go
type StubStore struct {
	response string
}

func (s *StubStore) Fetch() string {
	return s.response
}

func TestServer(t *testing.T) {
	data := "hello, world"
	svr := Server(&StubStore{data})

	request := httptest.NewRequest(http.MethodGet, "/", nil)
	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if response.Body.String() != data {
		t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
	}
}
```

Happy path한 코드를 준비했으니 이제는 약간 더 현실성이 있는 경우를 상정해 볼 차례이다. 그 말인즉슨 유저가 요청을 취소하기 전까지 `Store`가 `Fetch`를 끝내지 못할 경우이다.

## 먼저 테스트를 작성하자

우리의 핸들러(http.HandlerFunc)에서 `Store`를 중단시킬 방법이 필요하므로 인터페이스를 그에 맞게 바꿔주자.

```go
type Store interface {
	Fetch() string
	Cancel()
}
```

우리는 우리의 spy를 수정하여 `data`를 가져오는 데에 시간이 걸리도록 하고, 취소 여부를 알 수 있도록 하자. 또한 우리는 해당 구조체(it)가 어떻게 호출되는지 지켜볼 것이므로 이름을 `SpyStore`로 바꾸자. 그리고 `Store` 인터페이스와 상응하도록 `Cancel` 메소드를 추가해주자.

```go
type SpyStore struct {
	response string
	cancelled bool
}

func (s *SpyStore) Fetch() string {
	time.Sleep(100 * time.Millisecond)
	return s.response
}

func (s *SpyStore) Cancel() {
	s.cancelled = true
}
```

100 밀리초가 지나기 전에 요청을 취소하는 테스트를 추가해보고, store가 취소되는지 확인해보자.

```go
t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
      data := "hello, world"
      store := &SpyStore{response: data}
      svr := Server(store)

      request := httptest.NewRequest(http.MethodGet, "/", nil)

      cancellingCtx, cancel := context.WithCancel(request.Context())
      time.AfterFunc(5 * time.Millisecond, cancel)
      request = request.WithContext(cancellingCtx)

      response := httptest.NewRecorder()

      svr.ServeHTTP(response, request)

      if !store.cancelled {
          t.Errorf("store was not told to cancel")
      }
  })
```

[Go Blog: Context](https://blog.golang.org/context)에 따르면

> Context 패키지는 존재하는 Context들로 부터 새 Context의 값들을 파생시키는(derive) 함수를 제공하고, 이 때 이 값들은 트리를 형성한다: 어떤 Context가 취소될 경우, 그것으로 부터 파생된 모든 Context들이 취소된다.

이 점을 주의하여, 요청이 주어질 경우, 해당 요청의 호출 스택을 따라 모든 Context가 취소될 수 있도록 Context들을 파생시키는 것이 중요하다.

우리는 `request` 에서 새로운 `cancellingCtx`를 파생시킴과 동시에 `cancel` 함수를 얻게 된다. 그리고나서 `time.AfterFunc`를 통해 해당 함수가 5 밀리초 후에 호출되도록 설정한다. 마지막으로 우리는 `request.WithContext`를 통하여 이 새 Context를 사용할 것이다.

## 테스트를 돌려보자

우리가 예상하는 데로 테스트는 실패할 것이다.

```go
--- FAIL: TestServer (0.00s)
    --- FAIL: TestServer/tells_store_to_cancel_work_if_request_is_cancelled (0.00s)
    	context_test.go:62: store was not told to cancel
```

## 테스트가 통과하기에 충분한 만큼만 코드를 작성하자

***COMMENT: 새 문체 시작***
TDD와 익숙해지도록(disciplined) 해봅시다. _최소한의_ 코드를 추가하여 테스트가 통과하도록 해봅시다.

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		store.Cancel()
		fmt.Fprint(w, store.Fetch())
	}
}
```

이렇게 함으로써 테스트를 통과하기는 하지만 기분이 그리 좋지만은 않습니다.  당연한 얘기이지만 *모든 요청*에 대하여 데이터를 가져오기도 전에 `Store`를 취소하여서는 안됩니다.

TDD에 익숙해짐으로써 테스트의 결점이 보이기 시작했습니다. 좋습니다!

Happy path한 테스트를 수정하여 `Store`(it)가 취소되지 않았음을 확인(assert)하도록 합니다.

```go
t.Run("returns data from store", func(t *testing.T) {
    data := "hello, world"
    store := &SpyStore{response: data}
    svr := Server(store)

    request := httptest.NewRequest(http.MethodGet, "/", nil)
    response := httptest.NewRecorder()

    svr.ServeHTTP(response, request)

    if response.Body.String() != data {
        t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
    }

    if store.cancelled {
        t.Error("it should not have cancelled the store")
    }
})
```

두개의 테스트를 실행해보면 happy path 테스트는 이제 실패할 것입니다. 조금 더 실용적인 구현이 필요한 시점입니다.

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()

		data := make(chan string, 1)

		go func() {
			data <- store.Fetch()
		}()

		select {
		case d := <-data:
			fmt.Fprint(w, d)
		case <-ctx.Done():
			store.Cancel()
		}
	}
}
```

위 코드를 살펴봅시다.

`context` 에게는 `Done()`이라는 메소드가 있으며 이는 context가 "완료"되거나 "취소"될 경우 신호(signal)를 받는 채널을 리턴합니다. 해당 신호를 listen하여 해당 신호가 올 경우, `store.Cancel`을 호출하고 싶지만, `Store`가 그 전에 `Fetch`를 완료할 경우, 그 신호를 무시해줘야 합니다.

그러기 위해서 고루틴에서 `Fetch`를 호출한 뒤 새로운 채널인 `data`에 결과를 보내줍니다. 그리고 `select`문을 사용하여 두 비동기 프로세스를 경합(race)시킨 뒤 응답을 작성하거나 `Cancel`을 수행합니다.

## 리팩토링

Spy에 assertion 메소드들을 추가하여 테스트 코드를 리팩토링 해봅시다.

```go
type SpyStore struct {
	response  string
	cancelled bool
	t         *testing.T
}

func (s *SpyStore) assertWasCancelled() {
	s.t.Helper()
	if !s.cancelled {
		s.t.Errorf("store was not told to cancel")
	}
}

func (s *SpyStore) assertWasNotCancelled() {
	s.t.Helper()
	if s.cancelled {
		s.t.Errorf("store was told to cancel")
	}
}
```

Spy를 생성할 때 `*testing.T`를 잊지 말도록 합시다.

```go
func TestServer(t *testing.T) {
	data := "hello, world"

	t.Run("returns data from store", func(t *testing.T) {
		store := &SpyStore{response: data, t: t}
		svr := Server(store)

		request := httptest.NewRequest(http.MethodGet, "/", nil)
		response := httptest.NewRecorder()

		svr.ServeHTTP(response, request)

		if response.Body.String() != data {
			t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
		}

		store.assertWasNotCancelled()
	})

	t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
		store := &SpyStore{response: data, t: t}
		svr := Server(store)

		request := httptest.NewRequest(http.MethodGet, "/", nil)

		cancellingCtx, cancel := context.WithCancel(request.Context())
		time.AfterFunc(5*time.Millisecond, cancel)
		request = request.WithContext(cancellingCtx)

		response := httptest.NewRecorder()

		svr.ServeHTTP(response, request)

		store.assertWasCancelled()
	})
}
```

위 접근 방식은 작동하기는 하지만 자연스럽지는 않습니다.

웹 서버에서 `Store`를 직접 취소하는데에 관여하는 것은 부자연스럽습니다. `Store`가 또 다른 slow-running 프로세스들에 의존하는 경우를 생각해 봐야 합니다. `Store.Cancel`이 올바르게 파생 context들에게 취소를 전파(propagate)하도록 해야합니다.

`context`를 사용하는 주요 이유 중의 하나는 일관된 취소를 수행하기 위함입니다.

[공식 고 문서에 의하면](https://golang.org/pkg/context/)

> 들어오는 요청들은 context를 생성하는 것이 좋습니다. 그리고 나가는 call은 context를 인수로 받는 것이 좋습니다. 두 과정의 사이의 함수들을 호출할 때 해당 context를 반드시 전파하여야 하며, 원한다면 해당 context를 WithCancel, WithDeadline, WithTimeout, 혹은 WithValue를 이용해 파생시킨 context를 상ㅇ할 수도 있습니다. Context가 취소될 때 해당 context를 상속(derived from)한 모든 context들 또한 취소됩니다.

다시 [Go Blog: Context](https://blog.golang.org/context)를 살펴보면:

> 구글에서는 고 프로그래머들로 하여금 모든 들어오는 요청과 나가는 요청 함수들의 모든 첫번째 인수를 context로 하도록 규정합니다. 이는 여러 팀에서 개발된 고 코드들이 서로 잘 작동하도록 합니다. Context는 간단한 방법을 통해 시간초과(timeout)와 취소를 관리할 수 있도록 하며, 보안 증명과 같은 중요한 값들이 고 프로그램에서 올바르게 이동되도록 합니다.

(잠시 시간을 내어 모든 함수가 context를 보낼 경우의 영향과 인간공학(ergonomics)적인 관점에서 생각해 봅시다.)

약간 불편하게 느껴진다면 좋습니다. though 해당 접근 방식을 따라하여 `context`를 `Store`에 넘겨 responsible하게 합니다. 이를 통해 해당 `context`를 그것에 의존하는 것들에 넘겨줄 수 있게 되고, 그 context들 또한 그것들을 멈추는 데에 responsible하게 됩니다.

## 테스트를 먼저 작성해 봅시다

각 구성요소(their)가 담당하는 부분(responsibility)이 바뀌었으므로 테스트 또한 수정해줍니다. 핸들러가 담당하는 부분은 이제 단지 context를 `Store`를 통해 전파시키는 일입니다. 또한 `Store`가 취소될 경우 보내지는 오류를 handle합니다.

`Store` 인터페이스를 수정하여 새 responsibilities를 수행할 수 있도록 합니다.

```go
type Store interface {
	Fetch(ctx context.Context) (string, error)
}
```

우선 핸들러 내부의 코드를 지워줍시다.

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
	}
}
```

`SpyStore`를 수정해줍시다.

```go
type SpyStore struct {
	response string
	t        *testing.T
}

func (s *SpyStore) Fetch(ctx context.Context) (string, error) {
	data := make(chan string, 1)

	go func() {
		var result string
		for _, c := range s.response {
			select {
			case <-ctx.Done():
				s.t.Log("spy store got cancelled")
				return
			default:
				time.Sleep(10 * time.Millisecond)
				result += string(c)
			}
		}
		data <- result
	}()

	select {
	case <-ctx.Done():
		return "", ctx.Err()
	case res := <-data:
		return res, nil
	}
}
```

Spy를 수정하여 `context`를 실제로 사용하도록 해봅시다.

결과값의 문자열을 한글자씩 append하는 느린 모의 프로세스를 고루틴을 이용해 만듭니다. 고루틴이 완료됨과 동시에 `data` 채널에 결과 값을 보내줍니다. 그리고 고루틴에서 `ctx.Done`을 listen하며 값이 들어올 경우 작업을 중단하게 해봅시다.

마지막으로 `select` 문을 사용하여 해당 고루틴이 완료되거나 취소되는 것을 확인하도록 합니다.

이전의 접근방식과 비슷하지만, 이번에는 고의 동시성 primitive를 사용하여 두개의 비동기 프로세스를 경합하게 하여 리턴할 값을 정해줍니다.

`context`를 사용하는 함수들과 메소드들을 만들 경우 아래와 비슷한 접근 방식을 사용하게 되므로 작동 방식을 이해해보도록 합시다.

이제 테스트들을 수정해줄 차례입니다. 이전에 진행한 취소 테스트를 지워줌으로써(comment out) happy path한 테스트를 수정해 봅시다.

```go
t.Run("returns data from store", func(t *testing.T) {
    data := "hello, world"
    store := &SpyStore{response: data, t: t}
    svr := Server(store)

    request := httptest.NewRequest(http.MethodGet, "/", nil)
    response := httptest.NewRecorder()

    svr.ServeHTTP(response, request)

    if response.Body.String() != data {
        t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
    }
})
```

## 테스트를 시도해봅시다

```
=== RUN   TestServer/returns_data_from_store
--- FAIL: TestServer (0.00s)
    --- FAIL: TestServer/returns_data_from_store (0.00s)
    	context_test.go:22: got "", want "hello, world"
```

## 테스트가 통과하기에 충분한 만큼만 코드를 작성해봅시다

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		data, _ := store.Fetch(r.Context())
		fmt.Fprint(w, data)
	}
}
```

Happy path는 이제... happy합니다. 이제 다른 테스트를 수정해 줄 차례입니다.

## 테스트를 먼저 작성해봅시다

오류가 발생할 경우 응답으로 어떤 것도 보내지지 않는 것을 테스트해야 합니다. 안타깝게도 `httptest.ResponseRecorder`는 이러한 기능을 제공하지 않으므로 spy를 추가하여 이를 테스트 해봅시다.

```go
type SpyResponseWriter struct {
	written bool
}

func (s *SpyResponseWriter) Header() http.Header {
	s.written = true
	return nil
}

func (s *SpyResponseWriter) Write([]byte) (int, error) {
	s.written = true
	return 0, errors.New("not implemented")
}

func (s *SpyResponseWriter) WriteHeader(statusCode int) {
	s.written = true
}
```

위의 `SpyResponseWriter`는 `http.ResponseWriter` 인터페이스와 상응하기에 테스트에 사용할 수 있습니다.

```go
t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
    store := &SpyStore{response: data, t: t}
    svr := Server(store)

    request := httptest.NewRequest(http.MethodGet, "/", nil)

    cancellingCtx, cancel := context.WithCancel(request.Context())
    time.AfterFunc(5*time.Millisecond, cancel)
    request = request.WithContext(cancellingCtx)

    response := &SpyResponseWriter{}

    svr.ServeHTTP(response, request)

    if response.written {
        t.Error("a response should not have been written")
    }
})
```

## 테스트를 시도해봅시다

```
=== RUN   TestServer
=== RUN   TestServer/tells_store_to_cancel_work_if_request_is_cancelled
--- FAIL: TestServer (0.01s)
    --- FAIL: TestServer/tells_store_to_cancel_work_if_request_is_cancelled (0.01s)
    	context_test.go:47: a response should not have been written
```

## 테스트가 통과하기에 충분한 만큼만 코드를 작성해봅시다

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		data, err := store.Fetch(r.Context())

		if err != nil {
			return // todo: 원하는 방식으로 로그를 남겨봅시다
		}

		fmt.Fprint(w, data)
	}
}
```

이제 서버 코드를 확인해보면 매우 간단해진 것을 확인할 수 있습니다. 직접적으로 취소하는 데에 관여하지 않고 단순히 `context`를 전파하고 이후로(downstream) 호출된 함수들에게 의존하기 때문입니다.

## 마치며

### 이 챕터에서 다뤄본 것들

- 클라이언트가 요청을 취소할 때의 HTTP 핸들러를 테스트 하는 방법
- Context를 사용하여 취소를 관리하는 법
- `context`를 인수로 받는 함수를 작성하고 고루틴, `select` 문, 채널을 이용하여 해당 context를 취소하는 방법
- Google의 가이드라인에 나와있는 데로 요청에 대한 호출 스택에 scoped context를 전파하여 취소를 관리하는 법
- `http.ResponseWriter`의 스파이를 작성하는 법

### context.Value는 무엇인가요?

[Michal Štrba](https://faiface.github.io/post/context-should-go-away-go2/) and I have a similar opinion.

> 만약 ctx.Value를 내 회사(존재하지는 않지만)에서 사용한다면, 해고당할 각오를 해야할 것입니다

몇몇의 엔지니어들은 `context`를 통해 값들을 전해주는 것이 *편리하다는* 이유로 옹호합니다.

편의성은 보통 좋지 않은 코드를 만들어냅니다.

`context.Values`의 문제점은 그것은 단순히 타입이 지정되지 않은 맵이기 때문에 타입 safety가 보장되지 않고 실제로는 가지고 있지 않은 값을 handle해줘야 할 수 있습니다. 한 모듈에서 다른 모듈로 보낼 경우 맵 키들의 대응 테이블(coupling)을 만들어야 하고, 코드에서 몇 부분이 수정될 경우 걷잡을 수 없게 되어버립니다.

요약하자면, **함수에 줄 값들이 필요하다면, `context.Value`를 사용하지 말고 타입이 지정된 인수로 넘겨줘야 합니다**. 이는 정적으로 해당 값들을 체크하는 것과 여러 사람이 쉽게 볼 수 있는 문서 작성에 있어서 반드시 필요한 부분입니다.

#### 하지만...

On other hand, it can be helpful to include information that is orthogonal to a request in a context, such as a trace id. Potentially this information would not be needed by every function in your call-stack and would make your functional signatures very messy.

[Jack Lindamood says **Context.Value should inform, not control**](https://medium.com/@cep21/how-to-correctly-use-context-context-in-go-1-7-8f2c0fafdf39)

> The content of context.Value is for maintainers not users. It should never be required input for documented or expected results.

### Additional material

- I really enjoyed reading [Context should go away for Go 2 by Michal Štrba](https://faiface.github.io/post/context-should-go-away-go2/). His argument is that having to pass `context` everywhere is a smell, that it's pointing to a deficiency in the language in respect to cancellation. He says it would better if this was somehow solved at the language level, rather than at a library level. Until that happens, you will need `context` if you want to manage long running processes.
- The [Go blog further describes the motivation for working with `context` and has some examples](https://blog.golang.org/context)
