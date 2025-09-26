# 트러블슈팅: React State 업데이트 충돌 문제

본 문서는 **Tropical 프론트엔드** 개발 과정에서 발생한 **드롭다운 선택값이 버튼 클릭 시 반영되지 않는** React State 업데이트 충돌 문제의 원인과 해결 과정을 정리한 기술 문서입니다.

FullCalendar와 제어 컴포넌트 간 상태 충돌로 인한 예측하지 못한 동작을 해결하여 React 상태 관리의 복잡성을 이해하고 체계적 디버깅 방법론을 확립하는 것을 목표로 합니다.

**작성자:** [왕택준](https://github.com/TJK98)

**작성일:** 2025년 9월 19일

**문서 버전:** v1.0

---

## 1. 문제 현상

### 1-1. 드롭다운 선택값 미반영

* **증상**: 드롭다운에서 연도/월 선택 시 onChange 이벤트는 정상 발생
* **문제**: "이동" 버튼 클릭 시 이전 값으로 동작
* **추가 증상**: 화면상 드롭다운 표시값도 변경되지 않음

### 1-2. 예상 vs 실제 동작

```javascript
// 예상 동작: 2030년 2월 선택 → 이동 버튼 → 2030년 2월로 이동
// 실제 동작: 2030년 2월 선택 → 이동 버튼 → 2025년 9월로 이동 (이전값)
```

### 1-4. 환경 정보

- **프론트엔드**: React 19.1.1 + Vite 7.1.2, Vanilla JS (ES6+)
- **추가 라이브러리**: FullCalendar, SCSS Modules
- **브라우저**: Chrome 140+
- **운영체제**: Windows 11, macOS Sequoia

---

## 2. 원인 분석

### 2-1. 1단계: 기본적인 State 업데이트 문제 의심

**가설**: `setSelectedYear/setSelectedMonth`가 정상 호출되지 않음

```javascript
onChange = {(e)
=>
{
    const newYear = Number(e.target.value);
    console.log('연도 선택:', newYear);  // 정상 출력 확인
    setSelectedYear(newYear);
}
}
```

**결과**: onChange는 정상 호출되지만 state 반영 안됨

### 2-2. 2단계: React 배치 업데이트 문제 의심

**가설**: React 18의 자동 배치로 인한 지연

```javascript
const handleMove = () => {
    setTimeout(() => {
        console.log('지연된 state:', selectedYear, selectedMonth);
        // 여전히 이전 값
    }, 0);
};
```

**결과**: 지연해도 state 업데이트 안됨

### 2-3. 3단계: 이벤트 순서 문제 의심

**가설**: mousedown → click → change 순서로 인한 문제

```javascript
const yearRef = useRef();
<select ref={yearRef} value={selectedYear}>
```

**결과**: `ref.current.value`도 이전 값 반환

### 2-4. 4단계: 제어/비제어 컴포넌트 충돌 의심

**가설**: `value` prop과 `ref` 동시 사용으로 인한 충돌

```javascript
<select ref={yearRef} defaultValue={selectedYear}>// value 대신 defaultValue
```

**결과**: 여전히 해결되지 않음

### 2-5. 5단계: 근본 원인 발견

**핵심 발견**: `handleDatesSet`에서 지속적으로 state 덮어쓰기

```javascript
const handleDatesSet = async (arg) => {
    // FullCalendar 뷰 변경 시마다 실행
    setSelectedYear(y);  // 드롭다운 선택값을 강제로 덮어씀
    setSelectedMonth(m); // 드롭다운 선택값을 강제로 덮어씀
};
```

**원인**: FullCalendar가 렌더링될 때마다 `datesSet` 이벤트 발생 → 현재 캘린더 뷰 기준으로 드롭다운 state 강제 리셋

---

## 3. 디버깅 과정

### 3-1. 체계적 디버깅 방법론

**값 추적**: 각 단계에서 실제 값을 콘솔 로그로 확인

```javascript
const debugCurrentState = () => {
    console.log('=== 현재 상태 ===');
    console.log('selectedYear state:', selectedYear);
    console.log('selectedMonth state:', selectedMonth);
    console.log('yearRef.current:', yearRef.current);
    console.log('DOM value:', yearSelectRef.current?.value);
};
```

**이벤트 추적**: 어떤 이벤트가 언제 발생하는지 로깅

```javascript
const handleDatesSet = async (arg) => {
    console.log('handleDatesSet 호출:', new Date().toISOString());
    console.log('덮어쓰기 전 selectedYear:', selectedYear);
    setSelectedYear(y);
    console.log('덮어쓰기 후 목표값:', y);
};
```

**상태 충돌 확인**: 여러 곳에서 동일한 state 업데이트 여부 점검

---

## 4. 해결 과정

### 4-1. 실패한 해결책들

**A. setTimeout 지연**

```javascript
const handleMove = () => {
    setTimeout(() => {
        doMove(selectedYear, selectedMonth);
    }, 100);
};
```

실패 이유: React 배치 문제가 아니었음

**B. ref 직접 읽기**

```javascript
const handleMove = () => {
    const year = yearRef.current.value;
    doMove(year, month);
};
```

실패 이유: 제어 컴포넌트에서 ref가 제대로 작동하지 않음

**C. 비제어 컴포넌트 전환**

```javascript
<select ref={yearRef} defaultValue={selectedYear}>
```

실패 이유: `handleDatesSet`의 덮어쓰기 문제가 근본 원인

### 4-2. 최종 해결책: 별도 ref + setTimeout 조합

```javascript
// 1. 간단한 별도 ref 사용
const yearRef = useRef();
const monthRef = useRef();

// 2. onChange에서 즉시 ref 업데이트
onChange = {(e)
=>
{
    const newYear = Number(e.target.value);
    setSelectedYear(newYear);
    yearRef.current = newYear;  // 즉시 ref 저장
}
}

// 3. 버튼에서 setTimeout + ref 조합
const handleMove = () => {
    setTimeout(() => {
        const y = yearRef.current;  // ref에서 최신값 읽기
        const m = monthRef.current;
        doMove(y, m);
    }, 0);
};

// 4. handleDatesSet에서 드롭다운 state 업데이트 제거
const handleDatesSet = async (arg) => {
    setYear(y);    // 캘린더 표시용만 업데이트
    setMonth(m);
    // setSelectedYear/setSelectedMonth 제거
};
```

**성공 이유**:

- ref는 `handleDatesSet`의 영향을 받지 않음
- setTimeout으로 이벤트 순서 문제 회피
- 상태 충돌 근본 원인인 중복 업데이트 제거

---

## 5. 테스트 검증

### 5-1. 상태 업데이트 테스트

```javascript
describe('드롭다운 상태 업데이트', () => {
    test('onChange 시 ref와 state 모두 업데이트', () => {
        const {getByTestId} = render(<CalendarComponent/>);
        const yearSelect = getByTestId('year-select');

        fireEvent.change(yearSelect, {target: {value: '2030'}});

        // state 업데이트 확인
        expect(yearSelect.value).toBe('2030');

        // ref 업데이트 확인
        expect(yearRef.current).toBe(2030);
    });

    test('이동 버튼 클릭 시 최신 선택값 사용', async () => {
        const {getByTestId} = render(<CalendarComponent/>);

        fireEvent.change(getByTestId('year-select'), {target: {value: '2030'}});
        fireEvent.click(getByTestId('move-button'));

        await waitFor(() => {
            expect(mockDoMove).toHaveBeenCalledWith(2030, expect.any(Number));
        });
    });
});
```

### 5-2. 상태 충돌 방지 테스트

```javascript
test('handleDatesSet이 드롭다운 선택값을 덮어쓰지 않음', () => {
    const {getByTestId} = render(<CalendarComponent/>);

    // 사용자 선택
    fireEvent.change(getByTestId('year-select'), {target: {value: '2030'}});

    // FullCalendar 이벤트 시뮬레이션
    act(() => {
        handleDatesSet({start: new Date(2025, 8, 1)}); // 2025년 9월
    });

    // 사용자 선택값 유지 확인
    expect(yearRef.current).toBe(2030); // 덮어쓰기 안됨
});
```

---

## 6. 성능 영향 분석

### 6-1. 변경 전후 성능 비교

**변경 전**:

- 상태 충돌로 인한 불필요한 리렌더링 발생
- 사용자 액션이 예측대로 동작하지 않음
- 디버깅을 위한 추가 코드 오버헤드

**변경 후**:

- setTimeout 0ms 지연: 무시할 수 있는 수준
- ref 추가 사용: 메모리 사용량 미미한 증가
- 상태 충돌 제거로 예측 가능한 동작

### 6-2. 메모리 사용량

- 추가 ref 2개: 8 bytes (Number 타입)
- setTimeout 콜백: 일시적 클로저 생성
- 전체적으로 무시할 수 있는 오버헤드

---

## 7. 관련 이슈 및 예방책

### 7-1. React 상태 관리 안티패턴

**위험한 패턴들**:

```javascript
// 동일한 데이터를 여러 state로 중복 관리
const handleSomeEvent = () => {
    setUserSelection(newValue);     // 사용자 선택값
    setDisplayValue(newValue);      // 화면 표시값
    setInternalState(newValue);     // 내부 상태값
};

// 여러 곳에서 같은 state 업데이트
const ComponentA = () => setSharedState(valueA);
const ComponentB = () => setSharedState(valueB);
```

**안전한 패턴들**:

```javascript
// 단일 소스 관리
const handleSomeEvent = () => {
    setUserSelection(newValue);     // 단일 소스
    // 다른 값들은 derived state나 useMemo로 계산
};

// 명확한 상태 소유권
const ParentComponent = () => {
    const [sharedState, setSharedState] = useState();
    return (
        <ComponentA onUpdate={setSharedState}/>
    <ComponentB value={sharedState}/>
)
    ;
};
```

### 7-2. 트러블슈팅 체크리스트

**즉시 확인할 사항**:

- 콘솔에 의도한 값이 출력되는가?
- state 업데이트가 여러 곳에서 일어나는가?
- 이벤트 핸들러가 예상 순서대로 실행되는가?
- useEffect나 다른 사이드 이펙트가 state를 덮어쓰는가?

**단계별 디버깅 방법**:

1. 값 추적: 각 단계에서 실제 값을 콘솔 로그로 확인
2. 이벤트 추적: 어떤 이벤트가 언제 발생하는지 로깅
3. 상태 추적: state 변경이 일어나는 모든 지점에 로그 추가
4. 타이밍 추적: setTimeout이나 useEffect 등 비동기 동작 확인

### 7-3. 코드 리뷰 가이드라인

**상태 관리 검토 포인트**:

- 하나의 데이터를 여러 state로 관리하고 있지 않은가?
- state 업데이트가 예측 가능한 단일 경로로 이루어지는가?
- useEffect의 dependency array가 올바르게 설정되었는가?
- 제어 컴포넌트와 비제어 컴포넌트가 혼재되지 않는가?

**이벤트 핸들러 검토 포인트**:

- 이벤트 핸들러에서 최신 state 값을 올바르게 참조하는가?
- 비동기 작업에서 클로저 문제가 발생하지 않는가?
- 이벤트 버블링이나 캡처링으로 인한 의도치 않은 동작은 없는가?

---

## 8. 결론 및 배운 점

### 8-1. 주요 성과

1. **상태 충돌 해결**: FullCalendar와 드롭다운 간 state 업데이트 충돌 제거
2. **예측 가능한 동작**: 사용자 선택값이 의도대로 반영되는 UI 구현
3. **디버깅 방법론 확립**: 체계적인 React 상태 문제 해결 프로세스 구축
4. **성능 최적화**: 불필요한 리렌더링과 상태 충돌 제거

### 8-2. 기술적 학습

**React 상태 관리의 복잡성**:

- 제어 컴포넌트에서 `value` prop 사용 시 React가 완전히 제어
- ref vs state: 제어 컴포넌트에서 ref는 예상과 다르게 작동할 수 있음
- 이벤트 순서: 브라우저 이벤트 순서가 예상과 다를 수 있음
- 상태 동기화: 여러 상태가 서로 영향을 미치는 복잡한 케이스 주의

**상태 충돌의 근본 원인**:

- 동일한 데이터를 여러 state로 중복 관리
- 명확하지 않은 상태 소유권과 업데이트 경로
- 외부 라이브러리(FullCalendar)와 React 상태 간 동기화 문제

### 8-3. 프로세스 개선

**체계적 문제 해결 방법론**:

1. 현상 파악: 예상 동작과 실제 동작의 차이 명확히 정의
2. 가설 수립: 단계별로 가능한 원인 가설 설정
3. 검증 실험: 각 가설에 대한 최소한의 테스트 코드 작성
4. 근본 원인 발견: 표면적 증상이 아닌 구조적 문제 식별
5. 해결책 구현: 근본 원인을 제거하는 최소한의 변경
6. 회귀 테스트: 수정이 다른 기능에 영향을 주지 않는지 확인

**설계 단계에서 고려사항**:

- 단일 책임 원칙: 하나의 state는 한 곳에서만 관리
- 명확한 데이터 플로우: state 업데이트 경로를 명확히 문서화
- 이벤트 핸들러 분리: UI state와 비즈니스 로직 state 구분

### 8-4. 장기적 개선 방향

**상태 관리 라이브러리 도입 검토**:

- 복잡한 상태 로직을 위한 Zustand나 Jotai 고려
- 전역 상태와 로컬 상태의 명확한 구분

**컴포넌트 아키텍처 개선**:

- Container-Presenter 패턴으로 상태 관리와 UI 로직 분리
- 커스텀 훅으로 상태 로직 캡슐화

**테스트 전략 강화**:

- 상태 변경 시나리오별 단위 테스트 작성
- 사용자 인터랙션 플로우 기반 통합 테스트

이러한 해결 과정을 통해 **React 상태 관리의 복잡성**을 깊이 이해하게 되었으며, 향후 유사한 상태 충돌 문제 발생 시 체계적이고 신속한 해결이 가능한 방법론을 확립했습니다.
