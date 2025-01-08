---
description: >-
  예약 어플리케이션 에서 Relay를 사용하며 컴포넌트 설계 철학에 따라 고민했던 점과 그 해결 과정을 정리해 보았다.  REST API와
  GraphQL의 설계 차이를 느낄 수 있었고, Relay 의 철학을 최대한 잘 반영하기 위해 어떤 선택을 했는지 리팩토링을 통해 얻은
  인사이트를 다룰려고 한다.
hidden: true
---

# Relay 철학에 따른 컴포넌트 설계 고민과 적용기



### 기획 요구 사항과 설계 고민&#x20;

기존에는 RegularClass 내에&#x20;

Relay는 데이터 fetching 로직과 UI 렌더링을 명확히 분리하고, 각 컴포넌트가 자신의 데이터만을 요청하도록 설계하는 것을 지향합니다. 하지만, 프로젝트를 진행하면서 아래와 같은 고민이 있었습니다:

```
RegularClasses (상위)
  └─ RegularClass (카드)
      └─ Modal (클래스 정보 + 타임슬롯)
```

```
RegularClasses
  └─ RegularClass
      └─ TimeSlots
          └─ Modal (클래스 정보 + 선택된 타임슬롯)
```



1. **데이터 fetching 책임 분리**: `TimeSlots` 컴포넌트에서 `reservationQuery`를 호출해 데이터를 가져오도록 설계했지만, 이는 Relay 철학에 어긋나는 구조였습니다. `TimeSlots`는 단순히 타임 슬롯을 렌더링하는 책임만 가져야 함에도 불구하고 추가적인 데이터를 fetch하는 책임까지 맡게 되었습니다.
2. **모달 위치와 로직의 모호성**: 클릭 이벤트에 따라 모달을 열고 데이터를 fetching하는 로직이 여러 컴포넌트에 분산되어 관리가 어렵고, 각 컴포넌트의 역할이 불분명해 보였습니다.
3. **GraphQL과 REST API의 설계 차이로 인한 혼란**: REST API에서는 여러 엔드포인트를 호출하거나 서버 로직을 변경해야 가능했던 작업들이 GraphQL에서는 더 유연하게 처리될 수 있다는 점이 명확히 체감되지 않았습니다.



### Relay &#x20;

#### 기존 설계

아래는 기존 설계입니다:

```
type TimeSlotsProps = {
  timeSlots: ReadonlyArray<TimeSlotFragment$key>
};

export const reservationQuery = graphql`
  query RegularClassReservationQuery($id: Int!, $date: Date!) {
    regularClass(where: { regularClassId: $id }) {
      ...RegularClassFragment @arguments(date: $date)
    }
  }
`;

export const TimeSlots = ({ timeSlots }: TimeSlotsProps) => {
  const [queryReference, loadQuery] = useQueryLoader(reservationQuery);
  const [isModalOpen, setModalOpen] = useState(false);

  const handleCloseModal = () => {
    setModalOpen(false);
  };

  const handleTimeSlotClicked = (id: number) => {
    setModalOpen(true);
    loadQuery({ id, date: new Date() });
  };

  return (
    <>
      {timeSlots.map((timeSlot, idx) => (
        <TimeSlot key={idx} timeSlot={timeSlot} onClick={() => handleTimeSlotClicked(timeSlot.id)} />
      ))}
      {isModalOpen && queryReference && (
        <ReservationModal isOpen={isModalOpen} onClose={handleCloseModal} selectedClass={queryReference} />
      )}
    </>
  );
};
```

위 코드에서는 `TimeSlots` 컴포넌트가 `regularClass` 데이터를 fetching하는 역할까지 맡아 역할이 중첩되었습니다.

#### 리팩토링된 설계

리팩토링 후에는 데이터 fetching 로직과 UI 렌더링 책임을 분리했습니다:

```
const ReservationModal = ({ isOpen, onClose, classId, date }) => {
  const [queryReference, loadQuery] = useQueryLoader(reservationQuery);

  useEffect(() => {
    if (isOpen && classId && date) {
      loadQuery({ id: classId, date });
    }
  }, [isOpen, classId, date]);

  if (!queryReference) {
    return null;
  }

  return (
    <Modal isOpen={isOpen} onClose={onClose}>
      <Suspense fallback={<Loading />}>
        <ClassDetails queryReference={queryReference} />
      </Suspense>
    </Modal>
  );
};

const TimeSlots = ({ timeSlots, onTimeSlotClick }) => (
  <>
    {timeSlots.map((timeSlot, idx) => (
      <TimeSlot key={idx} timeSlot={timeSlot} onClick={() => onTimeSlotClick(timeSlot.id)} />
    ))}
  </>
);
```

`TimeSlots`는 단순히 `timeSlots`를 렌더링하며, 데이터 fetching은 `ReservationModal`에서 처리합니다. 이렇게 하면 각 컴포넌트의 역할이 명확해지고, 재사용성이 높아집니다.

### 얻은 인사이트

긍정적 측면&#x20;

1. **컴포넌트의 관심사 분리**: 데이터 fetching과 UI 렌더링을 분리하니 각 컴포넌트의 책임이 명확해졌습니다. 이는 코드 유지보수성과 가독성을 크게 향상시켰습니다.
2. **Relay의 강력함 체감**: GraphQL과 Relay를 통해 데이터를 선언적으로 요청하고, 이를 컴포넌트에 전달하는 과정을 학습하며 REST API에서 경험했던 제한점들을 극복할 수 있었습니다.
3. **리팩토링의 중요성**: 초기 설계에서 느꼈던 모호성을 해결하기 위해 리팩토링을 진행하며, 더 나은 설계를 발견할 수 있었습니다.
4. **선언적 데이터 요청**

```typescript
typescriptCopy// REST API - 여러 단계의 명령형 코드
const getModalData = async () => {
  const timeSlot = await fetchTimeSlot();
  const classInfo = await fetchClass(timeSlot.classId);
  const instructor = await fetchInstructor(classInfo.instructorId);
  // ... 데이터를 조합하는 추가 로직 필요
}

// GraphQL - 필요한 데이터 구조를 선언적으로 표현
const query = graphql`
  query TimeSlotQuery {
    timeSlot {
      time
      regularClass {
        name
        instructor {
          name
        }
      }
    }
  }
`
```

2. **컴포넌트와 데이터의 자연스러운 결합**

```typescript
typescriptCopy// TimeSlot 컴포넌트가 필요로 하는 데이터를 
// 컴포넌트 옆에서 바로 정의할 수 있음
export const TimeSlotFragment = graphql`
  fragment TimeSlotFragment on TimeSlot {
    id
    time
    availableSeats
  }
`

export const TimeSlot = ({timeSlot}) => {
  const data = useFragment(TimeSlotFragment, timeSlot);
  // ...
}
```

3. **타입 안전성**

```typescript
typescriptCopy// GraphQL 스키마로부터 자동 생성된 타입
type TimeSlotFragment$data = {
  readonly id: string;
  readonly time: string;
  readonly availableSeats: number;
}

// 컴파일 타임에 타입 오류 감지 가능
```

이런 특징들이 개발자 경험을 크게 향상시키는 것 같습니다:

* 생산성 향상
* 오류 가능성 감소
* 코드 유지보수 용이성
* 직관적인 데이터 모델링

특히 Relay와 함께 사용할 때 이러한 장점들이 더욱 빛나는 것 같습니다.

&#x20;CopyRetry[Claude can make mistakes.\
Please double-check responses.](https://support.anthropic.com/en/articles/8525154-claude-is-providing-incorrect-or-misleading-responses-what-s-going-on)

GraphQL과 Relay를 사용하면서 느낀 어려움이나 한계를 분석해보겠습니다:

1. **러닝 커브**

```typescript
typescriptCopy// Relay의 독특한 패턴들을 이해해야 함
const Fragment = graphql`
  fragment MyComponent_data on Type @argumentDefinitions(
    count: {type: "Int", defaultValue: 10}
  ) {
    items @arguments(count: $count) {
      id
    }
  }
`;
```

* Fragment, Query, Mutation의 관계 이해
* 복잡한 컨셉들 (Preloading, Refetching, Pagination 등)

2. **보일러플레이트**

```typescript
typescriptCopy// 하나의 컴포넌트에 필요한 여러 파일들
- MyComponent.tsx
- MyComponent.graphql
- __generated__/MyComponent_data.graphql
- __generated__/MyComponentQuery.graphql
```

3. **컴파일 프로세스**

```bash
bashCopy# 코드 수정할 때마다 필요한 과정
1. GraphQL 스키마 업데이트
2. Relay 컴파일러 실행
3. 타입 생성
4. 코드 실행
```

4. **디버깅의 어려움**

```typescript
typescriptCopy// 여러 레이어를 거치는 데이터 흐름
Fragment → Query → Network → Cache → Component
// 문제가 어디서 발생했는지 파악하기 어려울 수 있음
```

5. **유연성 제한**

```typescript
typescriptCopy// 때로는 간단한 데이터 요청도 복잡해질 수 있음
const simpleDataFetch = () => {
  // REST였다면 단순 fetch로 해결될 일도
  // GraphQL/Relay에서는 정식 쿼리/프래그먼트 정의 필요
}
```

하지만 이러한 어려움들은 대부분:

1. 초기 학습 단계에서의 어려움
2. 더 나은 구조와 타입 안전성을 위한 트레이드오프
3. 장기적으로는 이점이 더 큰 제약사항들

### 캐싱처리 해줘야 한다.

서버에 특정 로직이 업데이트 되고 난 후, 캐싱되어있지 않은 클래스의 타임슬롯을 클릭하면 컨텐츠가 업데이트된 컨텐츠로 되어있지만,  이미 한번 본 타임슬롯 컨텐츠에선 업데이트 되지 않고 Store에 이전 데이터가 그대로 남아 있다. &#x20;

ReactQuery나 SWR을 사용할 때와 마찬가지로 캐싱 업데이트를 해주고, 다시 리페치 해야 함을 이야기하는것 같다.



<figure><img src="../.gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

***

### REST API와 GraphQL의 설계 차이

#### REST API 설계

REST API에서는 각 엔드포인트가 리소스를 기준으로 설계됩니다. 예를 들어, `/regular-class/{id}/timeslots` 같은 구조에서는 `TimeSlots` 컴포넌트가 `regularClass` 정보를 가져오려면 별도의 API 호출이 추가로 필요했을 것입니다. 이는 데이터를 병합하거나 백엔드 엔드포인트를 수정해야 하는 번거로움을 초래합니다.

```
각각 별도의 엔드포인트 호출 필요
GET /timeslots/{timeSlotId}
GET /regular-classes/{classId}


```

```

const TimeSlots = ({ timeSlots }) => {
  const [modalData, setModalData] = useState(null);
  const [isLoading, setIsLoading] = useState(false);

  const handleTimeSlotClick = async (timeSlotId) => {
    setIsLoading(true);
    try {
      // 두 번의 API 호출이 필요
      const timeSlotData = await axios.get(`/timeslots/${timeSlotId}`);
      const classData = await axios.get(`/regular-classes/${timeSlotData.classId}`);
      
      setModalData({
        timeSlot: timeSlotData,
        classInfo: classData
      });
    } catch (error) {
      console.error('Failed to fetch data');
    }
    setIsLoading(false);
  };
}
```



#### GraphQL 설계

GraphQL은 필요한 데이터를 선언적으로 요청하고, 서버가 이를 조합해 반환합니다. `TimeSlots` 컴포넌트는 `regularClass`와 `timeSlots` 데이터를 한 번의 쿼리로 요청할 수 있어 더 유연하게 설계할 수 있었습니다. 이를 통해 클라이언트와 서버 간의 데이터 통신이 단순화되고, 프론트엔드 로직이 깔끔해졌습니다.



#### 의문점 : REST API를 사용할 경우 `regularClass`와 그 안의 `timeSlots`를 함께 가져오면 되지 않을까?

#### REST API에서의 설계적 제한

"REST API를 사용할 경우 `regularClass`와 그 안의 `timeSlots`를 함께 가져오면 되지 않느냐"라는 의문이 있을 수 있습니다. 물론 REST API에서 특정 엔드포인트(`/regular-class/{id}/timeslots`와 같은)를 통해 두 데이터를 함께 가져오는 것도 가능합니다. 하지만 이 방식에는 몇 가지 한계가 있습니다:

1. **데이터 유연성 부족**: REST API는 고정된 데이터 구조를 반환하기 때문에, 만약 다른 컴포넌트에서 `regularClass` 데이터의 특정 필드만 필요하거나 다른 시점에 데이터를 요청해야 하는 경우, 별도의 엔드포인트를 추가하거나 서버 로직을 수정해야 합니다.
2. **과도한 데이터 전송**: REST API는 종종 클라이언트가 필요로 하지 않는 데이터를 함께 반환합니다. 이는 네트워크 비용 증가와 클라이언트에서 불필요한 데이터 처리를 초래합니다.
3. **프론트엔드 의존성 증가**: REST API에서 데이터를 병합하거나 변형하는 작업이 필요하면, 프론트엔드에서 이러한 로직을 구현해야 하며 이는 유지보수의 복잡성을 증가시킵니다.

#### GraphQL 은 어떨까?

반면 GraphQL은 클라이언트가 필요한 데이터만 명확히 선언적으로 요청할 수 있는 구조를 제공합니다. 예를 들어, `TimeSlots` 컴포넌트에서 `regularClass` 정보와 함께 `timeSlots`를 요청하려면 다음과 같은 쿼리를 작성할 수 있습니다:

```graphql
graphqlCopy codequery GetRegularClassAndTimeSlots($id: Int!, $date: Date!) {
  regularClass(where: { regularClassId: $id }) {
    id
    name
    timeSlots(date: $date) {
      id
      startTime
      endTime
    }
  }
}
```

이 쿼리는 필요한 데이터만 가져오며, 데이터 병합 및 필터링 작업을 서버에서 처리하기 때문에 프론트엔드 로직을 단순화할 수 있습니다. 또한, GraphQL의 타입 시스템은 요청된 데이터의 구조를 명확히 정의하므로 디버깅과 유지보수가 용이합니다.

***





### 마무리

Relay 철학에 맞춘 설계는 초기에는 복잡하게 느껴질 수 있지만, 프로젝트를 진행하면서 그 가치를 점점 더 체감할 수 있었습니다. 컴포넌트의 관심사를 명확히 하고, 데이터 fetching과 렌더링을 분리하는 방식은 더욱 유지보수 가능한 코드를 작성하게 해줍니다. 앞으로도 이러한 철학을 바탕으로 더 나은 설계를 고민해 나갈 것입니다.
