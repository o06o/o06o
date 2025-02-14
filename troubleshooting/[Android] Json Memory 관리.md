## **📄 JSON 파일 OOM (Out of Memory) 문제 해결 과정**

### **🛠 문제 정의**
- **상황**:  
  - `JSONObject`를 사용해 **100MB 이상의 JSON 파일**을 메모리에 한 번에 로드하면서 **OutOfMemoryError(OOM)** 발생.  
  - 복원 과정에서 **JSON 데이터가 중첩된 구조**를 가지고 있어 **메모리 사용량이 파일 크기 대비 2~3배 증가**.

- **증상**:  
  - **앱이 크래시되며 OOM 로그 발생:**  
    ```
    Failed to allocate... target footprint 512MB
    ```

---

### **🔍 원인 분석**
1. **`readBytes()`와 `JSONObject` 사용으로 전체 JSON 파일을 메모리에 적재**  
   - 100MB JSON 파일이 **전체 메모리를 차지**하며 **GC가 메모리 회수에 실패**.  
   - **중첩된 JSON 구조로 인해 실제 메모리 사용량은 2~3배 증가**.  

2. **메모리 누수 가능성**  
   - 복원 과정에서 **중첩된 `Map` 및 `List`가 메모리에 오래 유지**되어 누수 발생.  
   - GC가 불필요한 객체를 제때 해제하지 못함.  

---

### **🚀 해결 과정 (트러블 슈팅 단계)**

#### **1️⃣ `JsonReader`와 스트리밍 방식으로 전환**
- **대규모 JSON 데이터를 한 번에 읽지 않고 스트리밍 처리**.  
- **`CipherInputStream`과 `JsonReader`**를 결합해 **메모리 사용량을 최소화**.  
- **100MB 이상의 데이터도 OOM 없이 안정적으로 처리**.

```kotlin
private suspend fun processJson(reader: Reader) {
    JsonReader(reader).use {
        it.beginObject()
        while (it.hasNext()) {
            currentCoroutineContext().ensureActive()  // Coroutine 취소 상태 확인
            when (it.nextName()) {
                "name" -> processNameArray(it)
                "gender" -> processGenderArray(it)
                else -> it.skipValue()
            }
        }
        it.endObject()
    }
}
```

#### **2️⃣ 메모리 누수 방지 (`mutableMapOf`와 `WeakReference`)**
- 중첩된 데이터를 **`mutableMapOf()`로 재할당하거나 `WeakReference`로 약한 참조** 처리.  
- **`ensureActive()`를 각 suspend 함수에 추가**해 **취소 상태 확인** 및 **데이터 해제**.

```kotlin
private suspend fun processName(name: Map<String, Any?>) {
    val tmpName = name.filterKeys { it != "imageNameId" }
    val imageNameId = annotation["imageNameId"] as? String
    // 작업 처리 후 메모리 해제
}
```

#### **3️⃣ Coroutine 취소 관리 (`ensureActive` + Flow 상태 관리)**
- ViewModel에서 **취소 버튼으로 `CoroutineScope` 전체를 안전하게 취소**.  
- **`_isActive`와 `ensureActive()` 결합**으로 취소 신호를 빠르게 감지.

---

### **🔑 최종 코드 (Before & After)**

#### **Before (`JSONObject` 사용)**
```kotlin
val names = JSONObject(decryptedData).getJSONArray("name")
for (i in 0 until names.length()) {
    val nameObj = names.getJSONObject(i)
    processAnnotation(nameObj)
}
```

#### **After (`JsonReader` + `CipherInputStream`)**
```kotlin
private suspend fun processJson(reader: Reader) {
    JsonReader(reader).use {
        it.beginObject()
        while (it.hasNext()) {
            currentCoroutineContext().ensureActive()
            when (it.nextName()) {
                "name" -> processNameArray(it)
                "gender" -> processGenderArray(it)
                else -> it.skipValue()
            }
        }
        it.endObject()
    }
}
```

---

### **📚 교훈 및 개선사항**
- **`JSONObject`는 작은 데이터에 적합**, **대규모 데이터는 스트리밍 방식 사용**이 필수.
- **CipherInputStream과 JsonReader 조합**은 대규모 JSON 데이터를 처리할 때 최고의 선택.
- **Coroutine 취소 관리 (`ensureActive`)**로 비동기 작업의 안전성을 확보할 것.  
- **Heap Dump 분석 및 메모리 사용량 모니터링**을 주기적으로 수행.  


---

### **📈 향후 개선 방향**
1. **스트리밍 데이터 처리 최적화** 및 **IO 작업 속도 개선**.  
2. **자동화된 메모리 검사 도구** 도입.

---

### **✍️ 작성자**  
- 이 문서는 **JSON 복원 과정에서 발생한 OOM 문제 해결 과정**을 기록한 **트러블 슈팅 사례**입니다.
