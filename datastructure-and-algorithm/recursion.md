---
description: 재귀연습
---

# Recursion

## 재귀 기본 함수

### 1단계: 가장 단순한 재귀 함수 만들기

```javascript
// Some code// 목표: 1부터 N까지의 합 구하기

function sum(n) {
  // 1. 종료 조건 (Base case)
  if (n === 1) return 1;
  
  // 2. 재귀 호출
  return n + sum(n-1);
}

// 실행: sum(3)
// → 3 + sum(2)
// → 3 + (2 + sum(1))
// → 3 + (2 + 1)
// → 6

```

### 2단계: 최소값/최대값을 찾는 재귀 함수

```javascript

// Some code// 목표: 배열에서 최소값 찾기
function findMin(arr, index) {
  // 1. 종료 조건
  if (index === arr.length - 1) {
    return arr[index];
  }
  
  // 2. 현재값과 나머지 값들 중 최소값 비교
  const minOfRest = findMin(arr, index + 1);
  return Math.min(arr[index], minOfRest);
}

// 실행: findMin([3,1,4], 0)
// → Math.min(3, findMin([3,1,4], 1))
// → Math.min(3, Math.min(1, findMin([3,1,4], 2)))
// → Math.min(3, Math.min(1, 4))
// → Math.min(3, 1)
// → 1



findMinBetter([3,1,4], 0) 실행과정:
  
  
1단계: index = 0 (값: 3)
└─ findMinBetter([3,1,4], 1) 호출
   2단계: index = 1 (값: 1)
    └─ findMinBetter([3,1,4], 2) 호출
       3단계: index = 2 (값: 4)
       └─ 마지막 원소니까 4 반환
   ← 1과 4 비교: 1 반환
← 3과 1 비교: 1 반환



```

