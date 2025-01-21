---
hidden: true
---

# as const



TypeScript의 const assertion으로, 객체나 배열을 읽기 전용(readonly)이면서 더 구체적인 타입으로 만들 수 있다.

```typescript
// Some code
// as const 없이
type ActionType = typeof ALLOWED_ACTIONS[GameStatus][number];
// 타입: string

// as const 사용
type ActionType = typeof ALLOWED_ACTIONS[GameStatus][number];
// 타입: "READY_TO_PRESS" | "START_PRESS"
```

* 문자열 리터럴 타입으로 추론
* 배열이 튜플 타입으로 추론
