---
title: Vue 3 + Laravel Inertia에서 다른 페이지 데이터 가져오기 모범사례 가이드
thumb: img/ui-thumb.png
og: img/ui-1.png
date: 2025-02-06
credits: yjh
---

# Vue 3 + Laravel Inertia 데이터 가져오기 모범 사례 가이드

## 데이터 가져오기 전략 및 유스케이스

### 1. 실시간 데이터 업데이트가 필요한 경우
```typescript
// types/PageData.ts
interface PageData {
  id: number;
  title: string;
  content: string;
  lastUpdated: string;
}

// composables/useRealtimeData.ts
import { ref, onMounted, onUnmounted } from 'vue'
import { router } from '@inertiajs/vue3'
import type { PageData } from '@/types/PageData'

export function useRealtimeData(url: string, interval: number = 5000) {
  const data = ref<PageData | null>(null)
  const loading = ref<boolean>(false)
  const error = ref<string | null>(null)
  let timer: number | null = null

  const fetchData = async () => {
    loading.value = true
    try {
      const response = await router.visit(url, {
        method: 'get',
        preserveState: true,
        preserveScroll: true,
        only: ['pageData']
      })
      data.value = response.props.pageData
    } catch (e) {
      error.value = '데이터 업데이트 실패'
      console.error('실시간 데이터 업데이트 오류:', e)
    } finally {
      loading.value = false
    }
  }

  onMounted(() => {
    fetchData()
    timer = window.setInterval(fetchData, interval)
  })

  onUnmounted(() => {
    if (timer) clearInterval(timer)
  })

  return { data, loading, error }
}

// 사용 예시
const { data, loading, error } = useRealtimeData('/api/live-data')
```

### 2. 대량 데이터 처리가 필요한 경우
```typescript
// stores/largeDataStore.ts
import { defineStore } from 'pinia'
import { router } from '@inertiajs/vue3'

interface DataChunk {
  page: number
  items: any[]
  total: number
}

export const useLargeDataStore = defineStore('largeData', {
  state: () => ({
    chunks: new Map<number, any[]>(),
    loading: false,
    error: null as string | null,
    totalPages: 0
  }),

  actions: {
    async fetchChunk(page: number, pageSize: number = 100) {
      if (this.chunks.has(page)) return

      this.loading = true
      try {
        const response = await router.visit(`/api/data?page=${page}&size=${pageSize}`, {
          method: 'get',
          preserveState: true,
          only: ['chunk']
        })

        const chunk: DataChunk = response.props.chunk
        this.chunks.set(page, chunk.items)
        this.totalPages = Math.ceil(chunk.total / pageSize)
      } catch (error) {
        this.error = '데이터 청크 로딩 실패'
        console.error('청크 로딩 오류:', error)
      } finally {
        this.loading = false
      }
    }
  }
})
```

### 3. 오프라인 지원이 필요한 경우
```typescript
// services/offlineStorage.ts
import { ref } from 'vue'
import localforage from 'localforage'

export async function setupOfflineStorage() {
  await localforage.config({
    name: 'myApp',
    storeName: 'offlineData'
  })
}

export async function saveForOffline(key: string, data: any) {
  try {
    await localforage.setItem(key, {
      data,
      timestamp: new Date().toISOString()
    })
  } catch (e) {
    console.error('오프라인 저장 실패:', e)
    throw e
  }
}

export async function loadOfflineData(key: string) {
  try {
    return await localforage.getItem(key)
  } catch (e) {
    console.error('오프라인 데이터 로드 실패:', e)
    throw e
  }
}
```

## 보안 고려사항

### 1. CSRF 보호
```php
// app/Http/Middleware/VerifyCsrfToken.php
class VerifyCsrfToken extends Middleware
{
    protected $except = [
        // CSRF 토큰 검증이 필요없는 경로
        'api/public/*'
    ];
}

// JavaScript에서 CSRF 토큰 설정
axios.defaults.headers.common['X-CSRF-TOKEN'] = document
    .querySelector('meta[name="csrf-token"]')
    .getAttribute('content');
```

### 2. 데이터 유효성 검증
```typescript
// types/validation.ts
import { z } from 'zod'

export const PageDataSchema = z.object({
  id: z.number(),
  title: z.string().min(1).max(200),
  content: z.string(),
  lastUpdated: z.string().datetime()
})

export type PageData = z.infer<typeof PageDataSchema>

// 데이터 검증 함수
export function validatePageData(data: unknown): PageData {
  return PageDataSchema.parse(data)
}
```

## 성능 최적화

### 1. 데이터 캐싱
```typescript
// utils/cache.ts
export class Cache<T> {
  private cache = new Map<string, { data: T; timestamp: number }>()
  private maxAge: number

  constructor(maxAgeMs: number = 5 * 60 * 1000) {
    this.maxAge = maxAgeMs
  }

  set(key: string, data: T): void {
    this.cache.set(key, {
      data,
      timestamp: Date.now()
    })
  }

  get(key: string): T | null {
    const item = this.cache.get(key)
    if (!item) return null

    if (Date.now() - item.timestamp > this.maxAge) {
      this.cache.delete(key)
      return null
    }

    return item.data
  }
}

// 사용 예시
const pageDataCache = new Cache<PageData>(5 * 60 * 1000) // 5분 캐시
```

### 2. 요청 최적화
```typescript
// utils/requestOptimizer.ts
import { ref } from 'vue'
import type { Ref } from 'vue'

export function useDebounce<T>(value: Ref<T>, delay: number = 300) {
  const debouncedValue = ref(value.value) as Ref<T>
  let timeout: number

  watch(value, (newValue) => {
    clearTimeout(timeout)
    timeout = setTimeout(() => {
      debouncedValue.value = newValue
    }, delay)
  })

  return debouncedValue
}

// 사용 예시
const searchTerm = ref('')
const debouncedSearch = useDebounce(searchTerm)

watch(debouncedSearch, (value) => {
  // API 요청 수행
})
```

## 테스트 전략

### 1. 컴포넌트 테스트
```typescript
// tests/components/DataFetcher.test.ts
import { mount } from '@vue/test-utils'
import { describe, it, expect, vi } from 'vitest'
import DataFetcher from '@/components/DataFetcher.vue'

describe('DataFetcher', () => {
  it('데이터를 성공적으로 가져옵니다', async () => {
    const wrapper = mount(DataFetcher, {
      global: {
        plugins: [createTestingPinia({
          createSpy: vi.fn
        })]
      }
    })

    await wrapper.vm.$nextTick()
    expect(wrapper.text()).toContain('데이터 로딩 완료')
  })

  it('에러를 적절히 처리합니다', async () => {
    // 에러 상황 테스트
  })
})
```

### 2. API 통합 테스트
```php
// tests/Feature/PageDataTest.php
class PageDataTest extends TestCase
{
    public function test_can_fetch_page_data()
    {
        $response = $this->get('/api/page-data');

        $response
            ->assertStatus(200)
            ->assertJsonStructure([
                'title',
                'content',
                'lastUpdated'
            ]);
    }
}
```

## 실제 사용 사례

### 1. 대시보드 데이터 관리
```typescript
// composables/useDashboardData.ts
export function useDashboardData() {
  const store = usePageDataStore()
  const dataCache = new Cache<PageData>()

  const fetchDashboardData = async () => {
    const cachedData = dataCache.get('dashboard')
    if (cachedData) return cachedData

    const data = await store.fetchPageData('/api/dashboard')
    dataCache.set('dashboard', data)
    return data
  }

  return {
    fetchDashboardData
  }
}
```

### 2. 실시간 채팅 구현
```typescript
// composables/useChat.ts
export function useChat(roomId: string) {
  const messages = ref<Message[]>([])
  const echo = new Echo({
    broadcaster: 'pusher',
    // ... Pusher 설정
  })

  echo.private(`chat.${roomId}`)
    .listen('MessageSent', (e: MessageEvent) => {
      messages.value.push(e.message)
    })

  return {
    messages
  }
}
```

## 에러 처리 패턴

### 1. 전역 에러 처리
```typescript
// plugins/errorHandler.ts
export function setupErrorHandling(app: App) {
  app.config.errorHandler = (err, vm, info) => {
    console.error('전역 에러:', err)
    // 에러 로깅 서비스로 전송
    errorLoggingService.log(err, info)
  }
}

// 사용자 정의 에러 클래스
export class ApiError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public originalError?: unknown
  ) {
    super(message)
    this.name = 'ApiError'
  }
}
```

### 2. 재시도 로직
```typescript
// utils/retry.ts
export async function withRetry<T>(
  fn: () => Promise<T>,
  maxAttempts: number = 3,
  delay: number = 1000
): Promise<T> {
  let lastError: Error

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn()
    } catch (error) {
      lastError = error as Error
      if (attempt === maxAttempts) break
      await new Promise(resolve => setTimeout(resolve, delay * attempt))
    }
  }

  throw lastError
}

// 사용 예시
const data = await withRetry(() => fetchPageData('/api/data'))
```

## 권장 사항

**1. 데이터 로딩 상태**

- 항상 로딩 상태를 표시하여 사용자 경험 개선
- 스켈레톤 로딩 사용 고려

**2. 에러 처리**

- 모든 API 호출에 적절한 에러 처리 구현
- 사용자 친화적인 에러 메시지 제공

**3. 성능 최적화**

- 필요한 데이터만 요청
- 적절한 캐싱 전략 사용
- 페이지네이션 구현

**4. 타입 안전성**

- TypeScript 사용 적극 권장
- API 응답에 대한 타입 정의

**5. 테스트**

- 주요 데이터 흐름에 대한 테스트 작성
- 에러 케이스 테스트 포함
