# Frontend global 401 redirect trap

Many SPA frontends intercept 401 responses and redirect to login. This breaks public-access pages.

## The pattern
In `src/lib/api.ts`, a fetch wrapper intercepts 401:
```
window.localStorage.removeItem("portal-session");
window.location.href = "/login";
```

## The problem
Public pages that call API without a token trigger this redirect even when the user wasn't trying to log in.

## The fix (STG)
Only redirect if the user previously had a valid session:
```
const existing = loadSession();
if (existing) {
  // clean up stale session and redirect
  window.location.href = "/login";
}
throw new Error("请先登录");  // no session -> let component handle it
```

## Pitfalls
- Import `loadSession` from storage module
- Backend API routes for public pages must NOT require auth middleware
