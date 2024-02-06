# base-lazy-days

💡 학습 내용: 하나 이상의 페이지에서 작동하는 원리
- 쿼리 설정
- 페칭, 에러, 로딩 콜백 중앙화
- useQuery를 바로 대입하는 대신 커스텀 훅 사용
- 데이터 리페칭(데이터 새로고침 제어)
- Auth 통합 단계에서 인증을 진행하기 위해 서버와 통신할 때 React Query가 어떤 기능을 하는지
- 의존성 쿼리 (조건에 따라 작동하는 쿼리)
- 변이와 React Query에 쓰인 쿼리 테스트
- 변이, 페이지 매김, 프리페칭

<br />

### 1. 프로젝트 구조
***auth***
- 로그인한 사용자 ID는 컨텍스트에 의해 추적되어 로컬 스토리지에 저장
- 사용자 ID와 해당 JWT가 포함된 로그인 데이터 유형이 존재
- 누구인지 증명할때 사용하는 JSON웹 토큰
- 리액트 쿼리와 가장 관련이 높은 것: useAuthAction

***useAuthAction***
- 아무것도 반환하지 않는 useUser 훅을 실행하는데 이것이 ESlint오류가 발생하는 이유
- useUser는 이름, 주소와 같은 서버의 사용자 데이터를 유지 관리하고 사용자가 로그인하거나 로그아웃할 때 이를 업데이트하거나 지움

***axiosInstance***
- 다른곳에서 사용할 수 있는 상수 존재
- baseURL사용
- 필요한 경로에 대해 JWT header(JSON웹 토근 헤더)를 가져오는 방법도 있음

<br />

react-query, devtools 설치
```tsx
npm install @tanstack/react-query @tanstack/react-query-devtools
```

dev dependency성이 아닌 일반 dependency로 설치하는 이유
✔️ 앱 컴포넌트들에서 dev tools를 가져와 추가한 다음 dev tools를 사용하여 node 환경을 테스트하고 node 환경이 development인 경우 이를 포함하지 않아서 이다.

<br/>

ESLint plugin 설치
```tsx
npm i -D @tanstack/eslint-plugin-query
```

exhaustive dependencies 규칙: 쿼리 키에 쿼리 함수의 모든 dependencies가 포함되어 dependencies가 변경되는 경우 쿼리가 다시 실행되도록 할 수 있다.

```tsx
{
	"extends" : ["plugin:@tanstask/eslint-plugin-query/recommendede"]
}
```

eslint config의 extends array에 한줄을 추가해야 한다.

eslint config, eclent rxjs에 이 기능이 있으며 uncomment하면 된다.

<br />

### 2. queryClient 설정

```tsx
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient();
```

components/app/App.tsx

```tsx
import { AuthContextProvider } from "@/auth/AuthContext";
import { Calendar } from "@/components/appointments/Calendar";
import { AllStaff } from "@/components/staff/AllStaff";
import { Treatments } from "@/components/treatments/Treatments";
import { Signin } from "@/components/user/Signin";
import { UserProfile } from "@/components/user/UserProfile";
import { theme } from "@/theme";
```

소스 directory에 이 @ alias를 사용하고 있다.

tsconfig.json에 보면 확인 가능하다.

```tsx
    "paths": {
      "@/*": ["src/*"],
      "@shared/*": ["../shared/*"]
    }
```

paths에서 @는 소스 디렉터리, @share는 typr definitions가 있는 한 수준 위의 공유 directory를 의미

✔️ 이를 통해 많은 …을 사용하거나 폴더 내에서 탐색하지 않아도 가져오기를 수행할 수 있다.

이러한 aliases를 사용하면 소스 폴더나 공유 폴더로 바로 이동할 수 있다.

components/app/App.tsx

```tsx
import { queryClient } from '@/react-query/queryClient';
```

queryClient를 가져와 해당 @ alias를 사용하여 찾을 수 있다.

저장을 할 때 몇 가지 경고, 몇 가지 eslint 경고가 표시되고 항목이 재정렬된다.

```tsx
import { QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
```

QueryClientProvider와 ReactQueryDevtools를 가져온다.

```tsx
export function App() {
  return (
    <ChakraProvider theme={theme}>
      <QueryClientProvider client={queryClient}>
        <AuthContextProvider>
          <Loading />
          <BrowserRouter>
            <Navbar />
            <Routes>
              <Route path="/" element={<Home />} />
              <Route path="/Staff" element={<AllStaff />} />
              <Route path="/Calendar" element={<Calendar />} />
              <Route path="/Treatments" element={<Treatments />} />
              <Route path="/signin" element={<Signin />} />
              <Route path="/user/:id" element={<UserProfile />} />
            </Routes>
          </BrowserRouter>
          <ToastContainer />
        </AuthContextProvider>
        <ReactQueryDevtools></ReactQueryDevtools>
      </QueryClientProvider>
    </ChakraProvider>
  );
}
```

인증 컨텍스트는 리액트 쿼리 도구를 사용하여 사용자가 로그인하고 로그아웃 할 때 사용자에 대한 서버 정보로 캐시를 업데이트 할 것이다. 따라서 query client provider 안에 있어야 한다.
하지만 error handler가 chakra toasts를 사용할 것이기 때문에 query client provider가 chakra provider 안에 있어야 한다.
따라서 queryClient를 위해 chakra에 접속할 수 있어야 한다.
Devtools는 query client provider 안에 있는 한 어디에 추가되는지 중요하지 않다.
상관없이 페이지 오른쪽 하단에 표시된다.

<br />

### 3. 커스텀 쿼리 훅 : useTreatments

대규모 앱에서는 각 데이터 유형에 대해 커스텀 훅을 만드는 것이 일반적이다.
- 여러 컴포넌트들의 데이터에 액세스해야 하는 경우 useQuery 호출을 다시 작성할 필요가 없다.
- 여러 개의 쿼리 호출들을 사용하는 경우 사용 중인 키가 혼동될 수 있다. 여기서 커스텀 훅을 사용하여 매번 호출을을 하면 키를 혼동할 위험이 없다.
- 사용하려는 쿼리 함수를 혼동할 위험이 없다. 커스텀 훅에 바로 넣을 수 있으며 여러 컴포넌트들에서 가져올 필요가 없다.
- 일반적으로 우리는 디스플레이 레이어에서 데이터를 얻는 방법의 implementation을 추출, 따라서 implementation을 변경하기로 결정한 경우 훅을 업데이트하기만 하면 된다. 컴포넌트들을 업데이트할 필요가 없다.

useTreatments에서 getTreatments 쿼리 함수는 Axios 인스턴스와 endpoint treatments를 사용하여 데이터를 가져온다. Axios 인스턴스를 보면 제 constants 파일의 base URL을 사용하고 있음을 알 수 있다.
이것은 http://localhost:3030 즉 제 서버가 실행 중인 포트이다. 따라서 http://localhost:3030/treatments로 이동한다. 브라우저에서 호출하여 살제로 반환되는 내용을 확인할 수 있다.
http://localhost:3030/treatments에서 endpoint가 반환하는 데이터를 볼 수 있다. 브라우저에 JSON extension이 있으면 더 보기 좋게 볼 수 있다. 일반적으로 서버는 대략적으로 데이터베이스라고 할 수 있는 것에서 항목을 반환하는 익스프레스 서버이지만 실제로는 JSON 파일이다. 따라서 treatments.JSON을 볼 수 있다.

```tsx
import { useQuery } from '@tanstack/react-query';
```

useQuery를 가져온다. treatments설정을 사용한 방식은 일련의 treatments를 반환하는 것이다. endpoint는 실제로 일련의 treatments를 반환한다. 따라서 실제로 반환해야 하는 것은 쿼리 함수에서 얻는 데이터 뿐이다.

```tsx
export function useTreatments(): Treatment[] {
  const { data } = useQuery({
    queryKey: [queryKeys.treatments],
    queryFn: getTreatments,
  });
  return data;
}
```

useQuery에서 반환된 개체에서 해당 데이터를 분해할 수 있다.
userQuery는 입력할 키와 GetTreatments쿼리 함수가 필요하다. GetTreatments는 인수를 사용하지 않으므로 인수를 전달하기 위해 익명 함수를 통해 호출할 필요가 없다.
함수에 대한 그대로 getTreatments를 추가하면 된다.
실제로 쿼리 키의 constant를 가져왔다. 단순히 속성을 문자열로 변환하는 것이다. 모든 호출에서 쿼리 키가 일관되게 유지되도록 하기 위해 이 작업을 수행하고 있다. 이렇게 하면 문자열에 오타가 생길 걱정을 할 필요가 없다. 속성에 오타가 있으면 오류가 발생한다.
쿼리키는 리액트 쿼리가 최대한 효율적으로 작동하도록 하는 핵심 요소이다. 데이터 확보 후 반환이 가능하다.

<br />

### 4. 풀백 데이터

Treatments.tsx에서 treatments가 정의되지 않은 것을 볼 수 있다. 그리고 정의되지 않은 맵 속성을 읽을 수 없다. 즉 실제로 useQuery를 호출할 때 사용 처리가 정의되지 않은 상태로 반환된다는 의미이다. 반환된 객체에서 구조화된 이 데이터는 쿼리 함수가 해결될 때 까지 정의되지 않았다.
데이터가 정의되지 않은 경우, 더 구체적으로 말하면 아직 로딩 중인 경우 조기 반환을 수행하여 Boologham opsum에서 이를 처리했었다, 여기서는 실제로 각 컴포넌트를 개별적으로 처리하는 것이 아니라 중앙에서 로드 및 오류를 처리하려고 한다.
다른 방법으로는 데이터에 대한 대체 값을 설정한다.

```tsx
const fallback = [];
```

그 대체값은 빈 배열로 만든다.
서버에서 아무런 treatments도 받지 못한 경우 아무것도 표지하지 않는다. 그리고 캐시에는 아무것도 없다. 

```tsx
const { data = fallback } = useQuery({
```

이를 구조화된 데이터의 기본값으로 설정한다. 대체 처리 배열로 명시적으로 입력했기 때문에 여기서 TypeScript 문제가 발생한다. 어떤 종류의 배열이라도 될 수 있다.

```tsx
const fallback: Treatment[] = [];
```

따라서 데이터를 반환할 때 대체 데이터가 될 경우 모든 유형의 배열이 될 수 있으며 이를 빈 Treatment 배열로 명시적으로 입력한다.
확인하면 화면에 표시되기 전에 잠시동안 비어있는것을 볼 수 있다.

<br />

### 5. useIsFetching을 사용하는 중앙 집중식 페칭 표시기(Indicator)

리액트 쿼리 훅인 useIdFetching을 사용한다.
- 소규모 앱에서는 useQuery 반환 객체에서 isFetching을 사용했다. useQuery 반환 객체에서 isFetching을 분해했다.
- isLoading은 isFetching과 동일하며 캐시된 데이터는 없다. isFetching은 더 큰 카테고리이고 isLoading은 가져오는 작은 카테고리이며 해당 쿼리에 대해 캐시된 데이터가 없다.
- 더 큰 앱에서는 로딩 스피너를 표시해야 한다. 쿼리가 데이터를 가져오는 과정에 있는 경우 중앙 집중식 로딩 스피너가 앱 컴포넌트의 일부가 될 것이다. 가져오는 쿼리가 있으면 이 기능을 켜도 가져오는 쿼리가 없으면 이 기능을 끈다.
- useIsFetching은 현재 가져오는 쿼리가 있는지 알려주는 훅이다. 즉 각 커스텀 훅에 대해 isFetching을 사용할 필요가 없다. 대신 useIsFetching을 사용한다. 로딩 컴포넌트에서 이 훅을 사용할 수 있으며 useIsFetching의 값은 스피너를 표시할지 여부를 알려준다.

App.tsx에 가면 네비게이션바, 경로, dev tools와 함께 Loading이 배치되어 있다.

Loading.tsx에서

```tsx
const isFetching = false;
```

이것은 표시하지 않는것을 의미한다. 하지만 이젠 useIsFetching 훅을 사용하여 표시할지 여부를 결정하는 것이다.

```tsx
import { useIsFetching } from '@tanstack/react-query';
```

useIsFetching훅을 가져온다.

이제 가져오지 않는다는 잘못된 의미를 useIsFetching이 반환하는 무엇이든으로 바꿀 수 있다.

```tsx
const isFetching = useIsFetching();
```

현재 가져오기 상태인 쿼리 호출의 수를 나타내는 정수를 반환한다.
isFetching이 0보다 크면 페칭 상태의 호출이 있으며 참으로 평가된다는 의미이다. 이 경우 디스플레이는 상속으로 설정된다. 즉 로딩 스피너가 표시된다.
현재 가져오기 항목이 없는 경우, 0은 거짓이므로 isFetching은 거짓으로 평가되고 로딩 스피너가 표시되지 않는다.
이렇게 하면 데이터가 로딩되기 전에 스피너가 나오는 것을 볼 수 있다.
창을 클릭하면 바로 로드되는 이유는 기본적으로 리액트 쿼리와 함께 제공되는 리페치 구성으로 창에 다시 초점을 맞추면 데이터를 다시 가져온다.

<br />

### 6. 쿼리 클라이언트에 대한 onError 기본 값

쿼리 캐시에 onError 콜백을 추가하여 쿼리 호출 사용시 오류가 발생할 경우 차크라 토스트 메시지를 표시할 수 있게 하는 방법

- 왜 리액트 쿼리가 사용 오류 훅을 제공하지 않는가?
    - useError 훅을 사용하는 것이 이치에 맞지 않는다. 반환하려면 단순한 정수 이상이 필요하다. 사용자에게 이러한 오류를 표시하려면 각 오류에 대한 문자열이 필요하다. 언제든 나타날 수 있는 오류를 개별 문자로 구현하는 방법이 불분명하다. 현재 가져오는 쿼리의 수만 알려주는 useIsFetching 만큼 깔끔하지 않다.
- 중앙 집중식 훅 대신 리액트 쿼리는 쿼리 캐시에 대해 설정할 수 있는 onError 콜백을 제공한다.
    
    ```tsx
    const queryClient = new QueryClient({
    	queryCache: new QueryCache({
    		onError: (error) =>
    			{ handle the error }
    	})
    })
    ```
    
    - 이는 queryClient를 생성할 때 쿼리 캐시의 기본값이다. 이 새 queryClient를 사용해 쿼리 캐시를 추가한 다음 오류 콜백을 추가할 수 있다.
    - 오류 콜백은 useQuery에서 발생하는 오류에 관계 없이 전달되며 콜백 본문 내에서 오류를 처리할 수 있다.

react-query > queryClient.ts에서 

```tsx
function errorHandler(errorMsg: string) {
  // https://chakra-ui.com/docs/components/toast#preventing-duplicate-toast
  // one message per page load, not one message per query
  // the user doesn't care that there were three failed queries on the staff page
  //    (staff, treatments, user)
  const id = 'react-query-toast';

  if (!toast.isActive(id)) {
    const action = 'fetch';
    const title = `could not ${action} data: ${
      errorMsg ?? 'error connecting to server'
    }`;
    toast({ id, title, status: 'error', variant: 'subtle', isClosable: true });
  }
```

이것을 onError 콜백에 사용한다.
오류 메시지라는 문자열을 사용한다.
중복된 토스트가 표시되지 않는지 확인할 수 있ㄷ게 토스트에 ID를 지정한다.
해당 ID로 활성 상태인 토스트가 없는 한 제목을 만든다.
현재 액션은 페치이지만, 이 함수를 돌연변이 오류에 재사용할 예정이다. 따라서 쿼리 오류인지 돌연변이 오류인지에 따라 이 작업을 업데이트 한다. 현재는 쿼리 오류만 다루고 있으므로 페치로 하드 코딩 한다. 데이터를 가져올 수 없다고 할 수 있다. 오류 메시지가 있으면 이를 사용하고 그렇지 않으면 서버에 연결하는 동안 오류가 발생했다고 할 것이다.
다음 옵션이 포함된 토스트 메시지를 표시한다.
error handler를 작성한 우에 이를 queryClient에 추가하는 데 많은 키 입력이 필요하지 않다. 따라서 queryClient에 쿼리 캐시 옵션을 제공한다. 그리고 그것은 리액트 쿼리에서 가져와야 하는 새로운 쿼리 캐시가 될 것이다.

```tsx
export const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error) => {
      errorHandler(error.message);
    },
  }),
});
```

확인을 위해 서버를 중지하면 서버에서 데이터를 가져오려고 시도하고 서버에 대한 연결이 실패하기 때문에 treatment 컴포넌트를 확인하려고 하면 오류가 발생할 것이다.
페이지를 새로고침하면 먼저 로딩을 시도하고 해당 treatment를 세번 시도한다. 이것은 리액트 쿼리의 기본 재시도 횟수이다. 그리고 데이터를 가져올 수 없다고 메시지가 뜬다. 이것이 토스트이다.

📌 파일 분리
- 앱 컴포넌트가 아닌 자체 파일에 모든 항목을 두는 것이 좋다.
