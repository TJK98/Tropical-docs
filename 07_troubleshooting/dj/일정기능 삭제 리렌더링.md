# 일정 삭제 후 리렌더링 문제 해결

## 문제 상황

### 증상
- 일정 삭제 버튼을 클릭하면 API 호출은 성공하여 데이터베이스에서는 삭제됨
- 하지만 UI가 즉시 업데이트되지 않음
- 브라우저를 새로고침해야만 삭제된 결과가 화면에 반영됨
- 일정 생성이나 수정은 정상적으로 리렌더링됨

### 로그 분석
- `🔄 서버로부터 최신 데이터 refetch 시작` 로그가 출력되지 않음
- `✅ 일정 삭제 워크플로우 완료 - UI 갱신됨` 로그도 출력되지 않음
- 삭제 관련 워크플로우 자체가 실행되지 않는 상황

## 근본 원인 분석

### 핵심 문제: 컴포넌트 간 책임 분리 미흡

1. **ScheduleItem 컴포넌트**가 IntegratedCalendar의 `handleEventUpdate` 워크플로우를 거치지 않고 **직접 useSchedule의 `deleteSchedule` 함수를 호출**
2. 이로 인해 IntegratedCalendar에서 구현한 **서버 데이터 refetch 로직이 실행되지 않음**

### 상세 원인

#### 1. 잘못된 삭제 호출 경로
```javascript
// ScheduleItem.jsx - 문제가 있던 코드
const handleConfirmDelete = useCallback(async (event) => {
  // ...
  await deleteSchedule(scheduleId); // 직접 호출 - 문제!
}, [schedule, deleteSchedule]);
```

#### 2. 일관되지 않은 CRUD 패턴
- **생성/수정**: IntegratedCalendar의 `handleEventUpdate` → useSchedule 함수 호출
- **삭제**: ScheduleItem에서 직접 useSchedule 함수 호출 ← 불일치!

## 해결 방법

### 1. 책임 분리 원칙 적용

#### useSchedule Hook
```javascript
// API 통신만 담당 - 상태 관리 로직 제거
const deleteSchedule = useCallback(async (scheduleId) => {
  console.log('🗑️ 일정 삭제 API 호출 시작:', scheduleId);
  
  try {
    // 오직 API 통신만 담당
    await scheduleApi.delete(scheduleId);
    console.log('✅ 일정 삭제 API 성공:', scheduleId);
  } catch (error) {
    console.error('❌ 일정 삭제 API 실패:', error);
    throw error;
  }
}, []);
```

#### IntegratedCalendar 컴포넌트
```javascript
// 워크플로우 지휘 - Single Source of Truth 패턴
} else if (action === 'delete') {
  console.log('📅 일정 삭제 워크플로우 시작:', { scheduleId, data });

  // 1. API를 통해 서버에서 삭제
  await deleteSchedule(scheduleId);
  console.log('✅ 서버에서 일정 삭제 완료');

  // 2. 서버로부터 최신 데이터 refetch (가장 중요!)
  console.log('🔄 서버로부터 최신 데이터 refetch 시작');
  await fetchSchedulesByMonth(currentYear, currentMonth, true);
  console.log('✅ 일정 삭제 워크플로우 완료 - UI 갱신됨');
}
```

### 2. 전역 핸들러를 통한 컴포넌트 연결

#### ScheduleItem 수정
```javascript
// 전역 핸들러를 통해 IntegratedCalendar 워크플로우 사용
const handleConfirmDelete = useCallback(async (event) => {
  // ...
  try {
    if (window.handleScheduleDelete) {
      await window.handleScheduleDelete(schedule); // 올바른 워크플로우
    } else {
      await deleteSchedule(scheduleId); // 백업
    }
  } catch (error) {
    console.error('일정 삭제 실패:', error);
  }
}, [schedule, deleteSchedule]);
```

#### IntegratedCalendar에 전역 함수 등록
```javascript
useEffect(() => {
  // 일정 삭제 핸들러 등록
  window.handleScheduleDelete = async (schedule) => {
    console.log('🎯 전역 삭제 핸들러 호출됨:', schedule);
    await handleEventUpdate('schedule', 'delete', schedule);
  };

  return () => {
    delete window.handleScheduleDelete;
  };
}, [handleEventUpdate]);
```

## 해결 결과

### ✅ 정상 동작하는 삭제 워크플로우

1. **ScheduleItem 삭제 버튼 클릭**
2. **`window.handleScheduleDelete` 호출**
3. **IntegratedCalendar의 `handleEventUpdate` 실행**
4. **로그**: `🎯 전역 삭제 핸들러 호출됨`
5. **로그**: `📅 일정 삭제 워크플로우 시작`
6. **API 삭제**: `await deleteSchedule(scheduleId)`
7. **로그**: `✅ 서버에서 일정 삭제 완료`
8. **서버 refetch**: `await fetchSchedulesByMonth(year, month, true)`
9. **로그**: `🔄 서버로부터 최신 데이터 refetch 시작`
10. **UI 갱신**: 새로운 데이터로 리렌더링
11. **로그**: `✅ 일정 삭제 워크플로우 완료 - UI 갱신됨`

### ✅ 확인된 개선사항

- **즉시 리렌더링**: 삭제 버튼 클릭 시 바로 UI 업데이트
- **정확한 개수 표시**: 로그에 `6개 → 5개` 정확히 표시
- **일관된 CRUD 패턴**: 생성/수정/삭제 모두 동일한 워크플로우
- **상태 동기화**: 날짜 이동 후에도 삭제 상태 유지

## 교훈 및 개선사항

### 1. 아키텍처 설계 원칙

- **Single Source of Truth**: 서버를 신뢰의 원천으로 삼기
- **책임 분리**: API 통신과 상태 관리 로직 분리
- **일관된 패턴**: 모든 CRUD 작업에 동일한 워크플로우 적용

### 2. 디버깅 전략

- **로그 기반 디버깅**: 각 단계별 명확한 로그로 문제 지점 파악
- **워크플로우 추적**: 데이터 흐름을 따라가며 문제 원인 분석
- **컴포넌트 간 통신 검증**: 전역 함수 등록 여부 확인

### 3. 코드 품질 개선

- **명시적 에러 처리**: try-catch로 각 단계별 에러 처리
- **타입 안전성**: ID 필드 다중 매칭으로 호환성 보장
- **성능 최적화**: 강제 새로고침(`forceRefresh=true`)으로 캐시 문제 해결

## 관련 파일

- `/src/components/schedule/ScheduleItem.jsx` - 삭제 버튼 로직 수정
- `/src/components/calendar/IntegratedCalendar.jsx` - 워크플로우 및 전역 핸들러 추가
- `/src/hooks/schedule/useSchedule.js` - deleteSchedule 함수 단순화

## 태그

`#react` `#state-management` `#rerendering` `#debugging` `#architecture` `#troubleshooting`

---

**해결 일자**: 2025-09-22  
**해결자**: 신동준  
**리뷰어**: -  
**소요 시간**: 약 3시간
