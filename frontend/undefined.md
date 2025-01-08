---
description: >-
  회사 코드를 보면서  메모리 누수가 나는 코드인지 의심만 하고 제대로 보지 못했던 적이 있다. 이번기회에 다시 메모리 누수에 대해 알아보면서
  준비된 대응을 해보려고 한다.  이 과정에서는 아무것도 모르는 상태에서 가설과 리서치 그리고 증명 방식으로 이루어 지는데 백그라운드 지식이
  없어 잘못된 부분이 분명 있을 수 있다.
---

# 메모리 누수

웹 페이지가 멈춘 경우, 원인이 무엇이라고 생각하는가?&#x20;



* 네트워크 요청이 많이 데이터 전송이 느려지는지 확인한다. -> 캐싱으로 최적화
* 특정 리소스의 번들이 너무 클 수 있으므로 해당 번들을 분할할 수 있다.
* 많은 루프로 인해 메인스레드를 오래 점유하고 있진 않은지 자바스크립트 코드로 검토한다.
* 페이지 렌더링 프로세스 중에서 너무 많은 리플로우와 리페인트가 일어난다.
* 잘모르겟다.

메모리 누수가 페이지 로딩 속도의 문제의 원인





### 컴포넌트의 리렌더링 사이클과 독립적으로 유지되는 참조



### 모든 변수나 상태가 즉시 해제되지 않는다고 해서 메모리 누수인가?

메모리 누수의 핵심은

1. 더 이상 필요하지 않은 데이터가&#x20;
2. 가비지 컬렉터에 의해 회수되지 못하고
3. 계속 메모리를 점유하는 상태로&#x20;

```tsx
const Memoryleak=(props)=> {
  const res = []  
  
  function handleClickButton() {
    res.push(new Array(10000))  
  }
}
```

* 버튼을 클릭할 때마다 큰 크기의 배열이 생성된다.
* 이 배열들이 res에 계속 추가 됨&#x20;
* 컴포넌트가 마운트 되어 있는 동안 버튼을 클릭할 때마다 메모리가 계속 증가
* 컴포넌트가 다시 렌더링되거나 함수가 재호출 할 때 초기화 됨&#x20;
* 해당 페이지를 벗어나기 전까진 메모리가 해제되지 않음
* 메모리 사용량이 증가하는 것은 맞으나,  res는 현재 Memoryleak 함수 컴포넌트 범위 내에서만 존재하며, React의 컴포넌트 생명주기에 따라 메모리에서 제거된다.&#x20;

#### 가설 :

{% hint style="info" %}
`res`가 컴포넌트 함수 범위 내에만 존재한다면, 컴포넌트가 언마운트될 때 `res`에 저장된 데이터는 더 이상 참조되지 않으므로 Garbage Collector(GC) 가 이를 메모리에서 제거할 것이다.
{% endhint %}

* **단계 1**: React 컴포넌트를 생성하고, 내부에서 `res` 배열에 데이터를 추가.
* **단계 2**: 데이터의 메모리 사용량을 기록.
* **단계 3**: 페이지 이동으로 컴포넌트를 언마운트하고, 데이터가 메모리에서 해제되었는지 확인.
* **도구**: 브라우저의 DevTools(Memory 탭)에서 힙 스냅샷을 찍어 메모리 상태 확인.

기본 크롬탭의 max 메모리는 얼마일까?&#x20;

리서치를 해보았는데 정확히 말하기에는알 수 없다고 하나  range 인지만 해두고 넘어간다.\
![](<../.gitbook/assets/image (25).png>)



해당 자료:

[https://cloudzy.com/blog/which-browsers-use-the-least-memory/](https://cloudzy.com/blog/which-browsers-use-the-least-memory/)

[https://stackoverflow.com/questions/17491022/max-memory-usage-of-a-chrome-process-tab-how-do-i-increase-it#:\~:text=By%20default%20v8%20has%20a,4GB)%20on%2064%2Dbit.](https://stackoverflow.com/questions/17491022/max-memory-usage-of-a-chrome-process-tab-how-do-i-increase-it)

{% embed url="https://stackoverflow.com/questions/29620041/is-there-any-memory-limit-for-google-chrome-browser/34667584#34667584" %}
ㅊ
{% endembed %}

어플리케이션 실행 후 idle 상태&#x20;

대부분의 현대 브라우저(예: Chrome, Firefox)는 각 탭마다 **1GB에서 2GB** 정도의 메모리 한계를 가지므로, **4.5MB**는 전체 메모리 용량의 약 0.45% 정도를 차지 한다고 볼 수 있을까?&#x20;



```tsx
// Some code
function ChildMemoryleakTest() {
  const res = []

  const fn1 = () => {
    const a = new Array(10000)
    return a
  }

  function hadnleClickButton() {
    res.push(fn1())
  }

  return (
    <>
      <button onClick={hadnleClickButton}>trigger memory leak</button>
    </>
  )
}

function Memoryleak(props) {
  return (
    <div>
      <ChildMemoryleakTest />
    </div>
  )
}

export default Memoryleak

```



<figure><img src="../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

메모리를 해제 했는데 초기 기본 메모리 곡선보다 더 높은 상태에서 해제되었다.&#x20;





상태로 변경해 보면 어떨까?  메모리를 수동 해지 하고 나니 처음&#x20;

![](<../.gitbook/assets/image (28).png>)





setState를 연속 3번 클릭했을 때이다.

<figure><img src="../.gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>



3번 클릭 후 해지했을 때&#x20;

<figure><img src="../.gitbook/assets/image (30).png" alt=""><figcaption></figcaption></figure>



이 두 그래프들로 미루어 보았을 때 리액트도 직접 수동으로 해지해 주지 않으면 계속 남는건가? 라는 생각이 들었다. &#x20;

메모리 패널을 사용해 다시 한번더 보기로 했다.

useState로  state에 새로운 array가 업데이트 되면서 마운트와 언마운트를 반복했을 때 그래프 이다. 클릭을 할 때마다 히스토그램이 생성되고, 회색은 이전에 생성된 메모리 공간이 해지 되어\
![](<../.gitbook/assets/image (31).png>)
