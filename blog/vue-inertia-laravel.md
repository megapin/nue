---
title: Vue 3 + Laravel Inertia에서 다른 페이지 데이터 가져오기 가이드
thumb: img/ui-thumb.png
og: img/ui-1.png
date: 2025-02-06
credits: yjh
---

**1. Inertia Router 사용하기**

가장 기본적인 방법으로, Inertia의 router를 직접 사용하여 데이터를 가져옵니다.

``` html numbered
import { router } from '@inertiajs/vue3'

async function fetchPageData(url) {
  try {
    const response = await router.visit(url, {
      method: 'get',
      preserveState: true,
      preserveScroll: true,
      only: ['someData']
    })
    return response
  } catch (error) {
    console.error('데이터 가져오기 실패:', error)
    throw error
  }
}
```

**2. API 엔드포인트 사용하기**

별도의 API 엔드포인트를 만들어 데이터만 가져오는 방법입니다.

Laravel 컨트롤러

``` html numbered
class PageDataController extends Controller
{
    public function getData()
    {
        $data = [
            'title' => '페이지 제목',
            'content' => '페이지 내용',
        ];

        return response()->json($data);
    }
}
```

Vue 컴포넌트

``` html numbered
<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'

const pageData = ref(null)
const loading = ref(false)
const error = ref(null)

const fetchPageData = async () => {
    loading.value = true
    try {
        const response = await axios.get('/api/page-data')
        pageData.value = response.data
    } catch (e) {
        error.value = '데이터를 가져오는데 실패했습니다.'
    } finally {
        loading.value = false
    }
}

onMounted(() => {
    fetchPageData()
})
</script>
```

**3. Pinia Store 사용하기**

Pinia를 사용하여 상태를 전역적으로 관리하는 방법입니다.

``` html numbered
// stores/pageData.js
import { defineStore } from 'pinia'
import { router } from '@inertiajs/vue3'

export const usePageDataStore = defineStore('pageData', {
  state: () => ({
    data: null,
    loading: false,
    error: null
  }),

  actions: {
    async fetchPageData(url) {
      this.loading = true
      try {
        const response = await router.visit(url, {
          method: 'get',
          preserveState: true,
          preserveScroll: true,
          only: ['pageData']
        })
        this.data = response.props.pageData
      } catch (error) {
        this.error = '데이터를 가져오는데 실패했습니다'
      } finally {
        this.loading = false
      }
    }
  }
})
```

**4. 전역 공유 데이터 사용하기**

Laravel의 미들웨어를 통해 모든 페이지에서 데이터를 공유하는 방법입니다.

``` html numbered
// app/Http/Middleware/HandleInertiaRequests.php
class HandleInertiaRequests extends Middleware
{
    public function share(Request $request): array
    {
        return array_merge(parent::share($request), [
            'global' => [
                'commonData' => $this->getCommonData(),
                'userSettings' => $this->getUserSettings($request),
            ],
        ]);
    }
}
```

**5. Composables 사용하기**

Vue 3의 Composition API를 활용한 재사용 가능한 로직을 만드는 방법입니다.

> Put in the work upfront, and your system will pay dividends down the road.

``` .blue
// composables/usePageData.js
import { ref } from 'vue'
import { router } from '@inertiajs/vue3'

export function usePageData() {
  const data = ref(null)
  const loading = ref(false)
  const error = ref(null)

>  const fetchPageData = async (url) => {
>    loading.value = true
>    try {
>      const response = await router.visit(url, {
>        method: 'get',
>        preserveState: true,
>        preserveScroll: true,
>        only: ['pageData']
>      })
>      data.value = response.props.pageData
>    } catch (e) {
>      error.value = '데이터를 가져오는데 실패했습니다'
>    } finally {
>      loading.value = false
>    }
>  }

  return {
    data,
    loading,
    error,
    fetchPageData
  }
}
```

## 각 방법의 장단점

**API 엔드포인트**

장점: 성능이 좋음, 유연성 높음
단점: 추가 API 엔드포인트 관리 필요

**Pinia Store**

장점: 전역 상태 관리, 데이터 공유 용이
단점: 작은 규모에선 과도할 수 있음

전역 공유 데이터

장점: 자동 데이터 공유, 초기 로드시 포함
단점: 초기 로딩 크기 증가 가능성

**Composables**

장점: 코드 재사용성, 관심사 분리 깔끔
단점: 컴포넌트 간 상태 공유 어려움

**선택 가이드**

단일 컴포넌트에서만 사용하는 데이터 → Composables
여러 컴포넌트에서 공유하는 데이터 → Pinia Store
애플리케이션 전체 설정 → 전역 공유 데이터
특정 API 데이터만 필요한 경우 → API 엔드포인트

