# Android 대용량 JSON OOM을 스트리밍 처리로 바꾼 기록

## 문제

복원 기능에서 100MB가 넘는 중첩 JSON 파일을 문자열로 전부 읽은 뒤 `JSONObject`로 변환했습니다.
원본 byte/string과 파싱된 object tree가 동시에 메모리에 머물면서 제한된 Android heap에서
`OutOfMemoryError`가 발생했습니다.

```kotlin
val json = file.readText()
val root = JSONObject(json)
val items = root.getJSONArray("items")
```

파일 크기만 보고 필요한 heap을 예측하기도 어려웠습니다. JSON 문자열 외에도 object, key, collection과
중첩 구조를 위한 allocation이 추가되기 때문입니다.

## 해결 방향

전체 document를 메모리 안의 object tree로 만들지 않고 `JsonReader`로 필요한 항목을 순서대로 읽도록 바꿨습니다.
암호화된 파일도 먼저 전부 복호화하지 않고 `FileInputStream → CipherInputStream → Reader` 파이프라인으로 처리했습니다.

```kotlin
private suspend fun processJson(reader: Reader) {
    JsonReader(reader).use { json ->
        json.beginObject()
        while (json.hasNext()) {
            currentCoroutineContext().ensureActive()
            when (json.nextName()) {
                "items" -> processItems(json)
                else -> json.skipValue()
            }
        }
        json.endObject()
    }
}

private suspend fun processItems(json: JsonReader) {
    json.beginArray()
    while (json.hasNext()) {
        currentCoroutineContext().ensureActive()
        val item = readItem(json)
        persist(item)
    }
    json.endArray()
}
```

핵심은 객체 타입을 `JSONObject`에서 `Map`으로 바꾸는 것이 아니라, **동시에 보관하는 데이터의 범위를 한 항목으로
제한하는 것**입니다. 처리한 항목은 즉시 저장하거나 다음 단계로 넘기고, 전체 목록을 다시 메모리에 모으지 않습니다.

## 취소 처리

스트리밍은 긴 작업이므로 사용자가 복원을 취소했을 때 빠르게 중단되어야 합니다. 배열 항목 경계마다
`ensureActive()`를 호출해 coroutine cancellation을 확인하고, stream은 `use`로 닫습니다.

취소를 일반 실패로 변환하거나 삼키지 않아 상위 계층이 취소와 파싱 오류를 구분할 수 있도록 했습니다.

## 사용하지 않은 해결책

### `WeakReference`로 각 항목 감싸기

`WeakReference`는 객체가 필요한 동안에도 GC가 회수할 수 있어 데이터 처리 흐름을 불안정하게 만듭니다.
또한 전체 JSON object tree를 생성한 뒤 일부 참조를 weak하게 만드는 것은 allocation 자체를 없애지 못합니다.
따라서 이 문제의 메모리 경계로 사용하지 않았습니다.

### heap 크기 확대

`largeHeap`은 기기별 동작과 메모리 압박을 근본적으로 해결하지 않습니다. 입력 크기가 다시 커지면 같은 문제가
반복되므로 선택하지 않았습니다.

## 검증해야 할 항목

이 저장소에는 당시 Android Studio Profiler 원본과 측정 로그가 남아 있지 않아 개선률 숫자를 재구성하지 않았습니다.
같은 변경을 검증한다면 다음 조건을 고정해야 합니다.

- 동일 기기 또는 emulator heap 조건
- 동일한 JSON 파일과 암호화 여부
- peak Java/Kotlin heap
- 처리 완료 시간
- 취소 후 stream과 coroutine 종료 여부
- 손상되거나 예상하지 못한 field가 있는 파일의 오류 처리

정량 자료가 없는 상태에서 “메모리가 몇 % 감소했다”는 주장은 하지 않습니다. 확인된 결론은 전체 문자열과
object tree를 동시에 유지하던 구조를 한 항목 단위 스트리밍 구조로 변경했다는 점입니다.

## 배운 점

- 큰 입력에서는 parser API보다 데이터의 **동시 보존 범위**가 더 중요합니다.
- streaming으로 바꿔도 결과를 전체 collection에 쌓으면 같은 문제가 반복됩니다.
- 긴 I/O 작업은 성공 경로뿐 아니라 취소와 손상된 입력의 종료 경로를 함께 설계해야 합니다.
- 문제 해결 문서에는 추정치보다 재현 조건과 측정 근거를 남겨야 합니다.
