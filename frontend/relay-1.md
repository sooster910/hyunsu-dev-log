---
hidden: true
---

# Relay

### useQueryLoader

* **Relay**에서 데이터를 **지연 로딩**할 때 사용하는 훅인데 초기 렌더링시 데이터를 바로 로드하지 않고 실제로 필요할 때만 데이터를 로드 한다.&#x20;



## Relay의 핵심철학 Fragment&#x20;

* 프래그먼트를 알기전까진 한번의 호출로 원하는 데이터를 모두 불러 올 수 있으니, 필요한 쿼리를 모두 작성하고 한곳에 모든 컴포넌트를 넣었다. 이렇게 하고 나니 딱히 장점이 느껴지지 않았다. 다시 Relay에 대한 공식문서와 튜토리얼을 살펴본 결과 Fragment가 핵심인걸 알게 되었다.&#x20;
* Relay에서는 **Fragment**를 사용해 쿼리를 잘게 나누고, 컴포넌트마다 필요한 데이터만 요청할 수 있다.
* 예를 들어, `StoreList`와 `StoreDetails`를 나누고, 각각의 프래그먼트를 정의할 수 있다.

```tsx
fragment StoreList_query on Store {
  id
  name
  description
}

fragment RegularClasses_query on Store {
  regularClasses(date: $date, first: 2) {
    edges {
      node {
        _id
        name
        imageURL {
          url
        }
        timeSlots(where: { day: $date }) {
          _id
          startDateTime
          status
        }
      }
    }
  }
}

```



***

## Relay Store Data 확인

#### 📌 **1. Relay DevTools 사용하기**

Relay의 데이터를 시각적으로 볼 수 있는 가장 쉬운 방법은 **Relay DevTools**를 사용하는 것입니다.

**설치 방법 (크롬 확장 프로그램)**

1. **Chrome 웹 스토어**에서 Relay DevTools 설치
2. **개발자 도구 (F12) → Relay 탭** 확인

**확인할 수 있는 정보**

* **Store 데이터**: 현재 캐시에 저장된 데이터 (ID 단위로 정규화됨)
* **Fragment Data**: 각 Fragment의 데이터가 어떻게 Store에 매핑되는지 볼 수 있음
* **Pending Requests**: 현재 대기 중인 네트워크 요청 확인

***

#### 📌 **2. `environment.getStore().getSource().toJSON()` 직접 확인하기**

Relay Store의 내부 데이터를 코드로 확인할 수 있습니다.

**코드 예시**

```javascript
javascriptCopy codeimport { useRelayEnvironment } from 'react-relay';

const logRelayStore = () => {
  const environment = useRelayEnvironment();
  const storeData = environment.getStore().getSource().toJSON();
  console.log('Relay Store Data:', storeData);
};
```

<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption><p>store data</p></figcaption></figure>



## Relay정규화

*



***

## Store, Date 업데이트



### router를 통해 url로 상태관리를 할 경우

한번 버튼을 클릭했을 때  총 9번의 렌더가 있었고, 그 중 한번은 전체 페이지가 렌더링 되는 것을 발견했다. &#x20;

```tsx
// Some code

function StoreList({queryRef}:{queryRef:StoreList_query$key}) {
    const router = useRouter();
    const storeList = useFragment(StoreListFragment, queryRef);
    const avaialableStores =  storeList.stores.filter((store)=>config.CURRENT_AVAILABLE_STORE.includes(store.name))

    const handleClick = (storeId) => () => {
        router.push({
            pathname:"/booking",
            query:{...router.query, storeId}
        })
    };

    return (
        <><h1 className={"text-amber-700 text-3xl font-bold underline"}>StoreList</h1>
            <div>
                {avaialableStores.map(store => (
                    <div key={store._id}>
                        <Button variant={"bordered"} onPress={handleClick(store._id)}>{ store.name}</Button>
                    </div>
                ))}
            </div>
s        </>

    );
}
```

<figure><img src="../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>





### shallow 적용 후 :&#x20;

<figure><img src="../.gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>



### shallow 적용 전

<figure><img src="../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>
