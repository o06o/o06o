## **📄 JSON 파일 OOM (Out of Memory) 문제 해결 과정**

### **🛠 문제 정의**
- **상황**:  
  - `JSONObject`를 사용해 **100MB 이상의 JSON 파일**을 메모리에 한 번에 로드하면서 **OutOfMemoryError(OOM)** 발생.  
  - 복원 과정에서 **JSON 데이터가 중첩된 구조**를 가지고 있어 **메모리 사용량이 파일 크기 대비 2~3배 증가**.

- **증상**:  
  - **앱이 크래시되며 OOM 로그가 나오는 토스트 발생:**  
    ```
    Failed to allocate... target footprint 512MB
    ```

---

### **🔍 원인 분석**
1. **파일 전부를 읽어와서 메모리에 적재**  
   - 100MB JSON 파일이 **전체 메모리를 차지**하며 **GC가 메모리 회수에 실패**.  
  
2. **메모리에 적재된 내용을 `JSONObject`로 변환 후 사용으로 전체 JSON 파일을 메모리에 한번 더 적재**  
   - 100MB JSON 파일이 전체 메모리를 차지하며 GC가 메모리 회수에 실패.  
   - 중첩된 JSON 구조로 인해 실제 메모리 사용량은 매우 증가.  

3. **메모리 누수 가능성**
   - 복원 과정에서 파일 내용과 JSONObject가 메모리에 유지 되어 메모리 부족 현상이 나타남.

---

### **🚀 해결 과정 (트러블 슈팅 단계)**

#### **1️⃣ `JsonReader`와 stream 방식으로 전환**
- 대규모 JSON 데이터를 한 번에 읽지 않고 `JsonReader`를 이용해서 stream 처리.  
- 암호화된 경우 `FileInputStream`과 `CipherInputStream`를 결합해서 파일을 읽으면서 복호화하면서 stream 처리.
- 암호화 안된 경우 `FileReader`를 이용해서 파일을 읽으면서 stream 처리
- 100MB 이상의 데이터도 OOM 없이 안정적으로 처리.

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
- `JSONArray` 대신 `Map<String, Any?>` 구조로 변환해 더 가벼운 메모리 사용.  
- 중첩된 JSON 데이터도 재귀적으로 `Map`에 변환해 메모리 최적화.
- 중첩된 데이터 구조**에 `WeakReference`를 사용해 GC가 필요할 때 메모리 해제.

```kotlin
private suspend fun processAnnotationsArray(reader: JsonReader) {
        reader.beginArray()
        while (reader.hasNext()) {
            val nameRef = WeakReference(readObjectAsMap(reader))
            processName(nameRef.get() ?: emptyMap())
            nameRef.clear()
        }
        reader.endArray()
    }
```

#### **3️⃣ Coroutine 취소 관리 (`ensureActive` + Flow 상태 관리)**
- ViewModel에서 취소 버튼으로 `CoroutineScope` 전체를 안전하게 취소.  
- `ensureActive()`로 취소 신호를 빠르게 감지.

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

#### **After (`JsonReader`)**
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
- `JSONObject`는 작은 데이터에 적합, 대규모 데이터는 스트리밍 방식 사용이 필수.  
- `JSONArray` 대신 `Map` 사용으로 메모리 효율성 증가.  
- `WeakReference` 사용으로 메모리 누수 방지.


---

### **📈 향후 개선 방향**
1. **자동화된 메모리 검사 도구** 도입.

---

### **✍️ 작성자**  
- 이 문서는 **JSON 복원 과정에서 발생한 OOM 문제 해결 과정**을 기록한 **트러블 슈팅 사례**입니다.
