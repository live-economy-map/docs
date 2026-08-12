## Project: Shadow Economy Map — Frontend Specification
**Feature:** Admin Access
**Conventions:** see `0-frontend-conventions.md` — `/admin/login` uses `PublicRoute`+`AuthLayout`; nothing else in this feature is a route.
**API reference:** `04-admin-access-api.md`

---

### 4.1 Route
```
/admin/login → PublicRoute → AuthLayout → AdminLoginPage
```

### 4.2 Store (src/store/adminAuth.store.ts)
Zustand, persisted under `admin-auth-storage` (per conventions 0.3):
```typescript
interface AdminAuthState {
  token: string | null;
  admin: { id: string; email: string } | null;
  setAuth: (token: string, admin: { id: string; email: string }) => void;
  logout: () => void;
}
```
`logout()` clears both fields and, per API spec 4.2's note on statelessness, does not itself call the API — the logout API call and the store clear happen together in the `useAdminLogout` hook below, not split across two places.

### 4.3 Types
```typescript
export interface AdminLoginResponse {
  token: string;
  admin: { id: string; email: string };
}
```

### 4.4 Hooks (src/hooks/useAdminAuth.ts)
```typescript
export function useAdminLogin() {
  const setAuth = useAdminAuthStore((s) => s.setAuth);
  return useMutation({
    mutationFn: (creds: { email: string; password: string }) =>
      api.post<AdminLoginResponse>('/admin/auth/login', creds).then((r) => r.data),
    onSuccess: (data) => setAuth(data.token, data.admin),
  });
}

export function useAdminLogout() {
  const logout = useAdminAuthStore((s) => s.logout);
  return useMutation({
    mutationFn: () => api.post('/admin/auth/logout'),
    onSettled: () => logout(), // clear local state even if the API call itself fails
  });
}

export function useAdminMe() {
  const token = useAdminAuthStore((s) => s.token);
  return useQuery({
    queryKey: ['admin-me'],
    queryFn: () => api.get('/admin/auth/me').then((r) => r.data),
    enabled: !!token,
    retry: false, // a failed /me means an invalid token — don't retry, let the 401 interceptor handle it
  });
}
```

### 4.5 Form (src/pages/admin/AdminLoginPage.tsx)
React Hook Form + Zod, per template convention. Zod schema mirrors the backend's `loginSchema` (email required/valid, password required min 8) so invalid input is caught client-side before the request — but the 401 "Invalid email or password" message from a *valid-shaped but wrong* login must still be displayed as a form-level error, not silently swallowed, since it's a legitimate case the client-side schema can't catch.

---

**Next:** proceed to → [05. Data Pipeline Management Frontend]
