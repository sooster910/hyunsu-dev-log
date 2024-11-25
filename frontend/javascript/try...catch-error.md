---
hidden: true
---

# try...catch 와 비동기 error 동작원리





동기적인 코드&#x20;

* JS는 에러 전파라는 메커니즘에 의해  콜스택을 거슬러 올라가 가장 가까운 try..catch 문을 찾고 catch문에서 캡처된다.
* forEach 메서드의 콜백함수 내에서 에러가 발생하더라도 이 에러는 try블록 내부에서 발생한 것으로 간주된다. 그 이유는 try..catch문과 forEach의 콜백함수가 여전히 같은 메인스레드 콜스택 내에 존재하기 때문에, 가장 가까운 catch 블록을 찾을 수 있다. 이건 try..catch가 동기적 동작한다는 원리와 아래의 forEach 콜백 함수 또한 현재 동기적으로 실행 되므로 실행 컨텍스트내 순차적으로 처리가 가능하다는 점에 의의가 있다.  try..catch 블록 내 비동기처리가 있었더라면 catch문에서 캡처가 되지는 않는다.&#x20;

```javascript
//동기 코드 내에서 에러가 잘 잡힌다.
export function main(){
    try{
        [1,2,3].forEach((index)=>{
            throw new Error(`err: ${index}`)
        })
    }catch(err){
        console.log(err)
    }
}
```

{% hint style="info" %}
참고로 try..catch는 새로운 실행 컨텍스트를 생성하진 않는다.  현재 실행 중인 컨텍스트 내에서 처리되는데 에러가 발생하면 JS엔진이 내부적으로 처리해 catch블록으로 제어를 이동시킨다.  모든 블록마다 실행 컨텍스트를 생성하면 메모리 사용량이 크게 증가하고 성능 저하에 대한 JS엔진의 최적화된 동작 방식이다. &#x20;
{% endhint %}



### 비동기 코드

```
// Some code
```

<figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption><p>Webstorm IDE</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (1) (1).png" alt=""><figcaption><p>chrome browser</p></figcaption></figure>



여기서 주목할 점은 Uncaught Error 이다.

왜 이렇게 나타날까?&#x20;



먼저 단순한 코드 부터 파악 해보자.

```javascript

    try{
        (async()=> {
            throw new Error("에러 ")
        })()
            
    }catch(err){
        console.log(err)
    }
```

<figure><img src="../../.gitbook/assets/image (2) (1).png" alt=""><figcaption></figcaption></figure>

주목 해서 봐야 할 부분은 PromiseState: "rejected" 와 에러를 내뿜어 주긴 하기만 Uncaught Error 라는 부분을 인지하고 이런 상황이 왜 일어났는지를 이해해야 한다.





<figure><img src="../../.gitbook/assets/image (3).png" alt=""><figcaption><p>MDN Promise</p></figcaption></figure>





위의 코드에 대한 실행 흐름이다.

1. global execution context가  callstack 에 올라가고 , try 문을 실행&#x20;
2. async() 함수를 담은 anonymous execution context 실행&#x20;
3. async() 문은 await를 만나기 전까지 동기적으로 실행해 첫 줄인 throw new Error("에러") 구문 실행&#x20;
4. throw 키워드로 인해 rejected Promise가 생성되어 반환됨. (하지만 이 시점에서 제어권이 이벤트 루프로 넘어가지 않음(후속 처리 X)  Promise 객체는 생성되자마자 rejected 상태로 생성됨
5. &#x20;외부 try-catch는 동기적으로 실행되어 이미 콜스택에서 pop이 됨.
6. 하지만 해당 에러를 처리할 .catch() 나 await가 없으므로 이 상태로 방치된다.
7.  처리되지 않은 rejection으로 판단되어 'Uncaught (in promise) Error'를 출력한다.







다소 과정이 좀 길었지만,  글의 제일 처음 부터 주장해 온 try..catch의 동기식 처리를 미루어 보았을 땐  결국  마이크로큐에 있던 callback이 콜스택으로 넘어와 실행될 때는 try...catch 구문이 종료된 상태로 같은 콜스택에 존재하지 않기 때문이다.&#x20;



그럼 async 함수가 await를 만나기 전까지는 동기식으로 동작하니 try...catch의 특성상 error를 만나면 catch로 넘어가게끔 동기식 try...catch로 작성해 같은  콜스택에 있게 해주면 되지 않을까?&#x20;



```javascript
// Some code
    try{
        (async()=> {
            try{
                throw new Error("에러 ")
            }catch(err){
                console.log("내부 catch", err)
            }
                
        })()
            
    }catch(err){
        console.log(err)
    }
```

<figure><img src="../../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

결과는 예상한대로 try...catch 문 내에서 에러를 잡을  수 있었다.&#x20;

다시 흐름을 복기해보자면, &#x20;

async 함수를 호출 하면 await가 만나기 전까진 동기식으로 코드를 실행한다.

첫번째 줄의 throw new Error("에러") 를 만났을 때  try...catch의 동기식 방식으로 바로 catch로 넘어간다. catch로 넘어가서 console.log를 실행하고  콜스택에서&#x20;



그럼 Promise상태는 어떻게 될까?&#x20;





에러처리가 비동기적으로 수행되면 마이크로 태스크 큐에 관여하게 된다.&#x20;

&#x20;





더이상 uncaught error가 되지 않는다.  그러나 이 방법이 비동기 에러 핸들링에 좋은 방법이 아니라는 사실을 알게 되었다.&#x20;





8번 부분에서 HostPromiseRejectionTracker 라는 부분을 내가 다시 조사했을 땐, 단순히 비동기 콜백함수와 외부 try..catch 문이  같은 콜스택에 존재하지 않았기 때문이라기보다는 조금 더 근본적인 이유 라고 생각했다.



그 근본적인 이유는  HostPromiseRejectionTracker 가 하는 일인 rejection을 처리하는 핸들러가 없기 때문이다.&#x20;





`Promise`가 `fulfilled` 상태로 변경되는 과정과 이 과정에서 **마이크로태스크 큐**가 사용되는가?&#x20;



함수 내부의 코드 실행 중에 반환값이 정해지는 순간이 바로 `Promise`의 상태가 변경되는 시점

`try-catch` 블록에 의해 즉시 처리되므로 함수 실행이 멈추지 않습니다

함수 실행은 계속 진행되고, `try-catch` 이후의 작업이 없으므로 함수는 끝납니다





그러면 어떻게 대응해야 할까?&#x20;

