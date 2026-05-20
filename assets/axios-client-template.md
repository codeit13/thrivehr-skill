# Axios Client Template

Copy this code to `src/api/client.ts` and customize as needed.

```typescript
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';
import { toast } from 'sonner';
import { v4 as uuidv4 } from 'uuid';

let accessToken: string | null = null;

export function setAccessToken(token: string | null) {
  accessToken = token;
}

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 30_000,
  withCredentials: true,
  headers: { 'Content-Type': 'application/json' },
});

apiClient.interceptors.request.use((config: InternalAxiosRequestConfig) => {
  if (accessToken) {
    config.headers.Authorization = `Bearer ${accessToken}`;
  }
  config.headers['X-Correlation-Id'] = uuidv4();
  return config;
});

apiClient.interceptors.response.use(
  (response) => {
    const envelope = response.data;
    if (envelope?.success) {
      response.data = envelope.data;
    }
    return response;
  },
  async (error: AxiosError<{ error?: { code: string; message: string } }>) => {
    const status = error.response?.status;
    const errorBody = error.response?.data?.error;

    if (status === 401 && error.config && !(error.config as any)._retry) {
      (error.config as any)._retry = true;
      try {
        const refreshRes = await axios.post(
          `${import.meta.env.VITE_API_BASE_URL}/auth/api/v1/token/refresh`,
          {},
          { withCredentials: true },
        );
        const newToken = refreshRes.data?.data?.accessToken;
        if (newToken) {
          setAccessToken(newToken);
          error.config!.headers.Authorization = `Bearer ${newToken}`;
          return apiClient(error.config!);
        }
      } catch {
        setAccessToken(null);
        window.location.href = '/login';
      }
    }

    const message = errorBody?.message ?? 'An unexpected error occurred';
    if (status !== 401) {
      toast.error(message);
    }

    return Promise.reject(error);
  },
);
```
