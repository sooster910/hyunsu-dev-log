---
hidden: true
---

# 이런 모달로 괜찮을까?

리덕스를 통해 사용하고 있던 어플리케이션 공통 모달이 있었는데 겉으로 동작을 잘하는것 같은데 과연 정말 아무런 문제가 없는걸까? 라는 생각이 들었다. 동작하는는 부분만 보이는 건 아닐까? 식으로 비동기처리를 하고 있었는데 리액트의 철학과 반한다 라고 하지만 구체적으로 왜 이게 문제가 되는 건지 성능의 문제 인지가 궁금으로 시작되었다. 가진 의문을 바탕으로 가설과 검증을 해보았다. &#x20;







코드는 다음과 같다.



처음 내가 이 코드를 보고 내린 가설은&#x20;

`await store.dispatch(setAlert(data))`가 완료 되었다고 해서, 상태가 업데이트되고 나서 바로 React 컴포넌트가 렌더링될 것이라고 확신할 수 없다 였다.



이 코드의 실행 흐름은 ,&#x20;



그래서 구독하고 있던 컴포넌트들이 영향을 먼저 받기 때문에 dom에선 이미 alert-btn1 아이디를 가진 element들이 항상 준비 되어 있다.



그럼 DOM을 직접 조작하더라도 동작의 흐름에는  문제가 없이 작성했으니 괜찮은 걸까?&#x20;



```tsx
// Some code
 <button id="alert-btn1" onClick={() => setCount(count + 1)}>
      {count}
    </button>
```



그러나 여전히 DOM을 조작하는게 리액트의 선언적 패턴과 맞지 않는건 사실이다. 왜냐하면 컴포넌트의 생명 주기와 무관하게 동작하기 때문이다.&#x20;





리덕스 상태가 업데이트되면 구독하고 있던 리액트 컴포넌트는 새상태를 반영하기 위한 리렌더링을 시작한다. 하지만 리렌더링이 완료되는 시점과&#x20;

**리렌더링이 완료되는 시점**과 **`store.dispatch(setAlert(data))`가 완료된 시점**이 **동시에 일어나지 않는 경우가 왜 중요하지?**&#x20;

**상태 업데이트 후 리렌더링까지 시간이 걸릴 수 있다.**

**store.dispatch는 액션을 디스패치 햇지만, 그 상태 업데이트가 완료되었 더라도 바로 리액트 컴포넌트의 렌더링이 끝나는 것이 아니기 때문이다.**

이 시점에 버튼 요소들이 아직 DOM에 렌더링 되지 않을 수도 있다.  상태가 업데이트 되었지만 리렌더링이 비동기적으로 일어나기 때문에 버튼 요소들을 찾기 위해 document.getElementId를 호출했을 때 null이 반환될 수 있다.&#x20;



하지만 현재 코드는 Promise를 감싸고 있잖아?&#x20;



### 새로운  요구사항과 복잡성 분석

* 모달 상태 관리를 확장해야 한다. 기존의 단순 열림, 닫힘 상태에서 로딩상태
* 동적 컨텐츠 변경
* 로딩 상태 관리
* 에러 메시지 표시
* 트랜잭션 상태에 따른 UI 변경

### 현재 있는 구조를 유지하면서 어떻게 확장 할 수 있을까?&#x20;

* 상환하기를 클릭해서 다시 리덕스 store에 상태 업데이트를 하고 나면 구독한 Alert가 다시 리렌더링이 되면서 업데이트된 데이터를 보여줄 수 있다. &#x20;
* 하지만 실제 보여줘야 하는건 상환하기로 api 통신이 되는동안 loading -> 상환실패 or 상환성공으로 컨텐츠만 변경되어야 한다. 어떻게 모달을 유지할 수 있을까?&#x20;

### 현재 코드 구조의 한계&#x20;

* Promise기반의 단일 상태 전환 만이 고려 되어 있다. 이 때문에 상태관리의 제한성이 생기는데

```tsx
// Some code
  async function handleClickModal() {
    customAlert({
      title: '상환 ',
      body: '상환하시겠습니가? 🎉',
      btn1: '네 지금 상환할게요',
      btn2: '아니요 나중에 상환할게요',
    })
      .then(async (result) => {
        await delay(1000) //비동기를 내부에서 처리하게 되면 모달이 닫힌다.
        store.dispatch(setAlert({ title: '로딩', body: 'loading...' }))
      })
      .catch((err) => {
        console.error('error ', err)
      })
  }
```

```typescript
// Some code
async function customAlert(data) {
  await store.dispatch(setAlert(data))
  
  return new Promise((resolve, reject) => {
    const btn1 = document.getElementById('alert-btn1')
    const btn2 = document.getElementById('alert-btn2')
    
    function onBtn1Click() {
      // 여기가 중요! 버튼 클릭시
      btn1.removeEventListener('click', onBtn1Click)
      btn2.removeEventListener('click', onBtn2Click)
      store.dispatch(setAlert(null))  // 모달을 닫음
      resolve()  // Promise 해결
    }
    // ...이벤트 리스너 등록
  })
}
```



* 모달이 닫히는 이유는 `customAlert` 함수의 구현이 "한 번의 모달 표시 후 닫기"를 전제로 설계되어 있기 때문이다.
* `customAlert()`는 다른 곳에서도 사용하는 공통 모달이라 수정 불가
* 모달이 유지된 상태로 컨텐츠만 변경되어야 함
* 상환 -> 로딩 -> 성공 순서로 모달 내용이 변경되어야 함

순서를 변경 한다.  먼저 로딩을 보여주고 delay후 완료 처리를 한다.

```tsx
// Some code
 async function handleClickModal() {
    customAlert({
      title: '상환 ',
      body: '상환하시겠습니가? 🎉',
      btn1: '네 지금 상환할게요',
      btn2: '아니요 나중에 상환할게요',
    })
      .then(async (result) => {
        store.dispatch(setAlert({ title: '로딩', body: 'loading...' }))
        await delay(1000) 
        store.dispatch(setAlert({ title: '완료', body: '상환완료' }))
      })
      .catch((err) => {
        console.error('error ', err)
      })
  }
```



이렇게 하면 의도한대로 동작하게 된다.&#x20;



여기서 사용자가 할수 있는 가능성에 대해 모든 시나리오를 살펴 보자.



1. 이탈 방지&#x20;

`beforeunload` 이벤트는:

1. 브라우저 탭 닫기
2. 브라우저 주소 직접 변경
3. 페이지 새로고침 같은 상황을 처리합니다.

ReactRouter 를 사용하면 Prompt를 사용할 수 있다. \


* 브라우저 뒤로가기/앞으로가기 클릭
* React Router의 Link 컴포넌트 클릭
* programmatic navigation (예: `history.push()` 호출)

2. 모바일에서 long press
3. 느린 네트워크에서 로딩 상태가 오래 지속될 때
4. 네트워크 끊김 상태에서 버튼 클릭
5. 백그라운드 탭에서 모달 조작 시도
6. 무한대기상태  방지 핸들링





#### 6. 무한 대기 상태 방지 핸들링

* 타임아웃이 없다면 네트워크 연결이 끊겼을 때, 서버가 다운되었을 때, 서버의 처리 지연 상황에서 사용자는 로딩상태에 갇힐 수 있다.&#x20;
* promise를 이용해서 타임아웃을 구현해 보자.&#x20;











#### Promise구조의 한계&#x20;

* resolve또는 reject될 때 모달을 열고 닫는 방식이다.&#x20;

#### DOM직접 조작 방식

* DOM 조작은 React의 Virtual DOM 개념과 충돌을 일으킨다.
* 충돌을 일으킨다는건 구체적으로 어떤 영향을 미친다는 것일까? 이게 정말 웹어플리케이션에 문제가 있는 걸까?&#x20;
* 만약 이 버튼이 여러번 클릭 되어 지는 경우? 사용자가 의도적으로 빠르게 클릭했거나 외부 환경으로 인해 두번 눌러지게된경우도 있지 않을까?&#x20;
* &#x20;



### 사용자 입장에서 발생할 수 있는 시나리오들&#x20;

1. 결제 페이지-> 모달 열기 -> 뒤로 가기-> 다시 결제 페이지&#x20;
2. 결제 페이지-> 모달열기 -> 메인으로 이동-> 다시 결제 페이지
3. 결제 페이지-> 모달열기-> 마이페이지로 이동-> 다시 결제 페이지&#x20;



### 해결 접근 방법&#x20;

* React의 선언적 상태 관리와 Virtual DOM을 활용하는 방식으로 전환
