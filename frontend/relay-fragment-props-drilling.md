---
hidden: true
---

# 처음 Relay fragment 사용에서 props drilling 감소라는 막연한 생각

데이터 의존성을 컴포넌트와 함께 선언적으로 정의한다라는 colocation

여기서 말하는 데이터 의존성이라는게 뭘까?&#x20;

화면을 그릴 때 ㅇ

**컴포넌트가 화면을 렌더링하는 데 필요한 데이터**\
컴포넌트는 어떤 데이터를 필요로 하는지 명확히 정의해야 합니다. 이 데이터는 서버에서 가져와야 하며, 컴포넌트는 데이터의 구조와 내용을 알아야 렌더링할 수 있습니다.

**쿼리로 선언적으로 표현한 데이터 요구사항**\
컴포넌트는 자신이 의존하는 데이터를 **GraphQL 프래그먼트**로 명시합니다. 이 프래그먼트는 Relay가 컴포넌트의 데이터 요구사항을 자동으로 처리하고 쿼리를 최적화할 수 있도록 돕습니다.



props drilling이라는 막연한 생각에 어떻게 하면 prop drilling을 하지 않으려고 억지(?) 를 쓰는 듯한 고민을 했다.&#x20;

사실 리액트의 데이터 흐름은 단일 방향 철학으로써 양방향에 대한 문제점을 해결하기위한 메커니즘으로 위에서 아래로 내려주는건 당연하다.



1. date는 실제로 \*\*쿼리 변수(variable)\*\*입니다:

* 데이터의 일부가 아닌, 데이터를 가져오기 위한 조건
* UI 상태에 가까움
* 사용자의 선택에 따라 변경되는 값



```tsx
// BookingQuery
export const BookingQuery = graphql`
  query bookingQuery($storeId: String!) {
    stores(where: { _id: $storeId }) {
      ...RegularClassesStore_fragment
    }
  }
`

// Store Fragment
export const RegularClassesStoreFragment = graphql`
  fragment RegularClassesStore_fragment on Store
  @refetchable(queryName: "RegularClassesStoreRefetchQuery")
  @argumentDefinitions(
    date: { type: "Date!", defaultValue: null }
  ) {
    id
    name
    regularClasses(date: $date) {
      edges {
        node {
          id
          name
        }
      }
    }
  }
`

export const RegularClassesStore = ({ store }) => {
  const [data, refetch] = useRefetchableFragment(
    RegularClassesStoreFragment,
    store
  );

  const handleOnDayClick = (day) => {
    const date = DateTime.fromJSDate(day).setZone('Asia/Seoul').toISODate()
    refetch({ date }); // fragment variables 업데이트
  }

  return (
    <div>
      <SelectDate
        onChange={handleOnDayClick}
        value={selectedDate}
      />
      <RegularClasses data={data} />
    </div>
  );
};
```





그러니 Relay에서 관리하는 데이터는 서버 데이터이고 위에서 이야기한 데이터 의존성은 쿼리 데이터를 의미하는 것으로써 상위-하위 데이터 흐름을 유지하는게 예츠 가능한 UI동작을 보장한다.&#x20;



***



## variable은 어떻게 넘겨줄까?



```tsx
// Some code


export const BookingClassFormQuery = graphql`
  query BookingClassFormQuery($storeId: String!, $date: Date!) {
    stores(where: { _id: $storeId }) {
      _id
      id
      description
      ...RegularClassesFragment @arguments(date: $date)
    }
  }



export const BookingClassForm = () => {
  console.time('BookingClassForm')

  const router = useRouter()

  const data = useLazyLoadQuery<BookingQueryType>(BookingClassFormQuery, {
    storeId: '1',
    date: DateTime.now().setZone('Asia/Seoul').toISODate(),
  })
  const formik = useFormik<FormValues>({
    initialValues: {
      date: new Date(),
      store: '1',
    },
    onSubmit: (values) => {
      alert(JSON.stringify(values, null, 2))
    },
  })

  const handleOnStoreClick = (storeId) => {
    formik.setFieldValue('store', storeId, true)
  }
  function handleOnDayClick(selected: Date) {
    formik.setFieldValue('date', selected, true)
  }
  return (
    <form
      onSubmit={formik.handleSubmit}
      className={'flex flex-col items-center justify-center w-full'}
    >
      {/* <StoreList queryRef={data} onChange={handleOnStoreClick} value={formik.values.store} /> */}
      <SelectDate onChange={handleOnDayClick} value={formik.values.date} />
      <RegularClasses regularClasses={data.stores[0]} />
      {/*<StoreDetail store={data.stores[0]} />*/}
    </form>
  )
}

```



````tsx
// Some code
```typescriptreact
const regularClassesFragment = graphql`
  fragment RegularClassesFragment on Store
  @refetchable(queryName: "RegularClassesPaginationQuery")
  @argumentDefinitions(
    after: { type: "String" }
    date: { type: "Date!" }
    first: { type: "Int", defaultValue: 1 }
  ) {
    regularClasses(date: $date, after: $after, first: $first)
      @connection(key: "RegularClasses_regularClasses") {
      edges {
        cursor
        node {
          ...RegularClassFragment @arguments(date: $date)
        }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
`
type RegularClassesProps = {
  regularClasses: RegularClassesFragment$key
}
export const RegularClasses = ({ regularClasses }: RegularClassesProps) => {
  const { data, loadNext, hasNext, isLoadingNext } = usePaginationFragment(
    regularClassesFragment,
    regularClasses
  )
  console.log('regularClassesPagination', data)
  const handleLoadMore = () => {
    if (hasNext && !isLoadingNext) {
      loadNext(1)
    }
  }
  return (
    <LocationDetailLayout>
      <div className={'flex items-center justify-start'}>
        <Clock size={24} />
        <Text>{'클래스/시간 선택'}</Text>
      </div>
      <div className={' mx-auto '}>
        {data.regularClasses.edges.map((edge, i) => (
          <RegularClass key={i} regularClass={edge.node} />
        ))}
      </div>
      {hasNext && (
        <button
          onClick={handleLoadMore}
          disabled={isLoadingNext}
          className="mt-4 bg-blue-500 text-white px-4 py-2 rounded"
        >
          {isLoadingNext ? '로딩 중...' : '더 보기'}
        </button>
      )}
    </LocationDetailLayout>
  )
}

```
````



