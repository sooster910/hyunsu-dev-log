---
hidden: true
---

# CORS

```typescript
// Next.js API Route에서 CORS 처리 경험
export default function handler(req: NextApiRequest, res: NextApiResponse) {
  res.setHeader('Access-Control-Allow-Origin', '*');
  // CORS 관련 설정과 그 이유를 설명할 수 있음
}
```



* 브라우저의 보안 정책으로 인해, 다른 도메인(origin)의 리소스에 접근하는 것을 기본적으로 막는다.
* Access-Control-Allow-Origin 헤더는 어떤 도메인의 접근에서 허용할지 서버가 브라우저에게 알려주는 역할을 한다.
