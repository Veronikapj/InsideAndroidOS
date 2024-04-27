
# JNI 팁

## 요약

### 1. 규칙을 어기지 말 것
- **불변 객체 변경 시도 피하기**: Java에서 생성된 String 객체는 불변이므로 네이티브 코드에서 변경해서는 안 됩니다.
- **가변성 우회 피하기**: 네이티브 코드에서 Java 객체의 final 필드 변경은 Java 메모리 모델을 위반하며, 스레드 안전성 문제를 초래할 수 있습니다.

### 2. 네이티브의 자바 코드 참조 관련해서는 항상 명확히 문서화 할 것
- **자바 식별자 참조 실패 처리**: 필드, 클래스 혹은 메소드와 같은 자바 식별자 참조가 실패할 경우, 즉시 메소드 실패 처리를 수행해야 합니다.
- **명확한 문서화**: 자바 식별자를 참조하는 코드는 명확하게 문서화하여, 코드 수정 시 발생할 수 있는 크래시를 예방하세요.

### 3. 네이티브 코드로 단순한 타입을 전달할 것
- **복잡한 Java 구조 피하기**: 자바 구조체 대신 원시 타입이나 문자열 배열 등 간단한 타입을 네이티브 코드로 전달합니다.

### 4. 네이티브 코드는 가능한 정적 메소드로 만들 것
- **인스턴스 정보 최소화**: 인스턴스 메소드 대신 정적 메소드를 사용하여 필요한 데이터만 네이티브 코드에 전달하세요.
- **메모리 누수 방지**: 정적 메소드 사용으로 인스턴스 정보의 불필요한 전달을 피하고 메모리 누수 가능성을 줄일 수 있습니다.
- 참조 : 아래 정적 메소드 & 인스턴스 메서드 비교 

### 5. 가비지 컬렉터를 주의할 것
- 가비지 컬렉터는 네이티브 코드가 가지고 있는 네이티브 포인터 데이터를 옮길 수 있고 해제도 가능합니다.
- **LocalRef와 GlobalRef 관리**: 실행 중인 자바 코드가 그 객체에 대한 참조를 얻을 수 있어야 합니다. (GC reachability) 
    1. LocalRef 
        1. 어땐 객체에 대한 참조를(호출 스택 상에서) 가장 가까운 자바 메소드의 호출 스택에 넣는다. 
        2. LocalRef 객체는 네이티브 메소드가 호출했던 자바 호출 위치로 돌아가면 자동으로 삭제된다. (스택에서 빠진다) 
        3. 전체 개수가 제한되기 때문에 삭제해주어야 한다. 
    2. GlobalRef 
        1. 네이티브 코드가 자바 객체에 대한 참조를 JNI 호출 경계를 넘어서 계속 유지할 수 있는 방법
        2. 객체 참조를 영구 정적 배열에 추가된 것처럼 동작 
        3. 명시적인 참조 삭제 때까지 가비지 컬랙터가 객체 메모리를 회수 할 수 없게 해줌
        4. 흔한 사용 상황 : 네이티브 코드가 자바 객체를 생성할 때 
        5. 개수 제한 : 65535 (메모리 유출 개수가 될 수도 있다.)
        6. 비 관리대상 메모리 다루는 것처럼 할 것 

#### Ref. LocalRef와 GlobalRef 이해하기  

`LocalRef`와 `GlobalRef`는 가비지 컬렉터가 메모리 내의 객체를 이동시키는 것을 방지하지 않습니다.   
이는 객체를 직접 네이티브 포인터를 통해 접근할 때 문제를 일으킬 수 있습니다. 하지만, 이 참조들은 필요하지 않을 때까지 객체가 수집되지 않도록 보장합니다.

- 문제 상황 설명

    자바에서는 가비지 컬렉션이 메모리를 압축하여 단편화를 줄일 수 있습니다. 이 과정은 JNI에서 저장되거나 전달된 네이티브 포인터를 무효화할 수 있으며, 무효 메모리 위치에 접근하거나 "Dangling Pointer - 단절된 포인터" 문제를 발생시킬 수 있습니다.

- 해결 방법: Copying / Pinning

    Pinning은 객체의 메모리 주소가 변경되지 않도록 고정하는 기술입니다. 그러나 자바는 객체를 핀할 수 있는 직접적인 방법을 제공하지 않습니다. 대신, JNI는 `GlobalRef` 또는 `LocalRef`를 사용하여 참조를 안전하게 관리합니다. 비록 이 방법들이 객체의 주소가 변경되지 않는 것을 보장하지는 않지만, 안전한 참조 관리를 제공합니다.

- 예제 코드

    **자바 클래스**:
    ```java
    public class SampleObject {
        private int data = 123;

        public native void nativeMethod();
    }
    ```

    **JNI 구현**:
    ```cpp
    #include <jni.h>

    extern "C"
    JNIEXPORT void JNICALL
    Java_SampleObject_nativeMethod(JNIEnv *env, jobject obj) {
        // 자바 객체에 대한 LocalRef 생성 - 가비지 컬렉터에 의해 수집되는 것을 방지, 이동되는 것은 방지하지 않음
        jobject localRef = env->NewLocalRef(obj);
        
        // LocalRef를 사용하여 자바 객체의 필드에 접근
        jclass cls = env->GetObjectClass(localRef);
        jfieldID fid = env->GetFieldID(cls, "data", "I");
        jint data = env->GetIntField(localRef, fid);

        // 자바 객체로부터 데이터 사용
        printf("자바 객체로부터 받은 데이터: %d\n", data);

        // LocalRef 삭제
        env->DeleteLocalRef(localRef);
    }
    ```



### 6. 약한 참조(weak refs)를 유지할 것
- 자바 객체 관리를 네이티브 객체에 맡기면 안됨
- WeakGlobalRef : 객체가 필요 없어지면 가비지 컬렉터가 회수할 수 있도록 허용, 그 객체에 대한 접근이 필요할 때 안전하게 접근 가능 
    1. 가비지 컬렉션에서 제외되지 않음: WeakGlobalRef는 GlobalRef처럼 자바 객체를 가비지 컬렉션으로부터 보호하지 않습니다.
    2. Null 값 안전성: 가비지 컬렉션이 발생하여 객체가 회수된 경우, WeakGlobalRef는 자동으로 null을 가리키게 됩니다. 이는 참조가 더 이상 유효하지 않다는 것을 의미하며, 이를 통해 안전하게 객체의 유효성을 검사할 수 있습니다. (쓰레기 값을 가리키지 않음 - 원시 네이티브 참조와 다름)
    3. LocalRef 변환 사용: 가비지 컬렉션을 막기 위해 필요할 때 임시로 LocalRef로 변환하여 사용할 수 있습니다.

        잘못 사용 예
        ```cpp
        mCompanion = reinterpret_cast<jobject>(env->NewWeakGlobalRef(javaObj));
        if (env->isSameObject(mCompanion, NULL))
            return;
        jClass klass = env->getObjectClass(mCompanion) // mCompanion 은 Null 일 수도 있다.
        ```

        WeakGlobalRef로부터 LocalRef 획득하기 
        ```cpp
        mCompanion = reinterpret_cast<jobject>(env->NewWeakGlobalRef(javaObj));
        companion = reinterpret_cast<jobject>(env->NewLocalRef(mCompanion)); // LocalRef
        if (env->isSameObject(companion, NULL))
            return;
        jClass klass = env->getObjectClass(companion) // 안전!
        ```

    4. Map을 통해 관리 하는 방법: 자바 객체에 대한 참조를 자바/네이티브 경계에 놓인 map 내에서 관리하며, 이를 통해 네이티브 코드가 객체를 안전하게 참조하면서도 메모리 누수를 방지할 수 있습니다.




## Java에서 JNI(Java Native Interface)를 사용하여 네이티브 코드를 호출하는 방법  

### 정적 메소드 & **인스턴스 메소드**

- 인스턴스 메소드: 객체의 인스턴스 상태에 대한 접근이 필요할 때 사용됩니다. 이 방식은 객체의 특정 인스턴스에 종속된 데이터를 처리할 때 사용됩니다.
- 정적 메소드: 전역 상태나 클래스 수준의 데이터를 처리할 때 사용됩니다. 객체의 특정 인스턴스와 무관한 정보를 다룰 때 유용하며, 객체 생성 없이도 호출할 수 있는 장점이 있습니다.

ref. 인스턴스 참조
- 특정 객체의 인스턴스를 가리키는 참조  
- 객체가 Java에서 생성되면, 메모리에 할당되고 해당 객체를 가리키는 참조가 생성   
- 이 참조를 사용하여 프로그램은 객체의 메소드를 호출하거나 필드에 접근

### JNI에서의 인스턴스 참조

JNI를 사용할 때 인스턴스 메소드는 객체의 인스턴스 참조를 메소드의 첫 번째 파라미터로 전달   
이 참조는 `jobject` 타입이며, 네이티브 코드에서 객체를 조작하는 데 사용

## JNI에서 인스턴스 메소드와 정적 메소드 비교

### 인스턴스 메소드

Java에서의 클래스 및 인스턴스 메소드 정의 예시:

```java
public class UserData {
    private int userId;
    private String username;

    public UserData(int userId, String username) {
        this.userId = userId;
        this.username = username;
    }

    public native void updateUserData();
}
```

C/C++에서의 JNI 함수 구현 예시:

```cpp
#include <jni.h>

extern "C"
JNIEXPORT void JNICALL
Java_com_example_myapp_UserData_updateUserData(JNIEnv *env, jobject obj) {
    jclass clazz = env->GetObjectClass(obj);
    jfieldID userIdField = env->GetFieldID(clazz, "userId", "I");
    jfieldID usernameField = env->GetFieldID(clazz, "username", "Ljava/lang/String;");

    // 필드 값 읽기
    jint userId = env->GetIntField(obj, userIdField);
    jstring username = (jstring) env->GetObjectField(obj, usernameField);

    const char* nativeString = env->GetStringUTFChars(username, nullptr);
    // 네이티브 코드에서 데이터 처리

    env->ReleaseStringUTFChars(username, nativeString);
}
```

### 정적 메소드 

Java 코드
```java
public class User {
    private static int lastUserId;
    private static String lastUsername;

    public static void setLastUser(int userId, String username) {
        lastUserId = userId;
        lastUsername = username;
    }

    // 정적 메소드
    public static native void updateLastUserProfile();
}
```

C/C++ 네이티브 코드
```cpp
#include <jni.h>

extern "C"
JNIEXPORT void JNICALL
Java_com_example_myapp_User_updateLastUserProfile(JNIEnv *env, jclass clazz) {
    jfieldID idField = env->GetStaticFieldID(clazz, "lastUserId", "I");
    jfieldID nameField = env->GetStaticFieldID(clazz, "lastUsername", "Ljava/lang/String;");

    jint lastUserId = env->GetStaticIntField(clazz, idField);
    jstring lastUsername = (jstring) env->GetStaticObjectField(clazz, nameField);

    // 사용자 마지막 프로필 데이터를 사용하여 네이티브에서 처리
    // 예: 로깅, 모니터링 등

    // 문자열 메모리 해제
    env->ReleaseStringUTFChars(lastUsername, env->GetStringUTFChars(lastUsername, nullptr));
}
```

### JNI에서 인스턴스 메소드와 정적 메소드 비교

Java Native Interface (JNI)를 사용하여 Java와 네이티브 코드 간의 상호 작용을 구현할 때, 인스턴스 메소드와 정적 메소드는 각각의 유스케이스에 따라 선택될 수 있습니다. 아래 표는 두 메소드 유형의 주요 차이점을 요약합니다.

| 특성               | 인스턴스 메소드                                                   | 정적 메소드                                             |
|--------------------|-----------------------------------------------------------------|---------------------------------------------------------|
| **정의**           | 객체의 인스턴스 상태에 대한 접근이 필요할 때 사용                | 클래스 수준의 데이터를 처리할 때 사용                   |
| **데이터 접근**    | 객체의 특정 인스턴스에 종속된 데이터                             | 클래스 전체와 관련된 데이터                             |
| **사용 사례**      | 객체의 상태를 변경하거나 객체의 정보를 기반으로 연산 필요        | 객체 생성 없이도 호출 가능, 전역 상태 관리               |
| **메모리 관리**    | 객체의 생명주기에 영향을 받음                                    | 객체 생명주기와 무관, 메모리 관리가 더 단순              |
| **성능**           | 메소드 호출 시 객체 참조를 필요로 함                             | 객체 참조 없이 호출되므로 상대적으로 리소스 사용이 덜함  |
| **장점**           | 객체의 상세한 상태 정보에 대한 액세스가 가능                     | 메모리 누수 위험이 적고, 전역 상태를 쉽게 관리할 수 있음 |
| **단점**           | 메모리 누수 위험 있음, 객체가 필요로 하는 리소스가 더 많음        | 객체의 특정 상태를 직접적으로 변경할 수 없음      


### 인스턴스 메소드 대신 정적 메소드 사용의 이점

정적 메소드를 사용하면 특정 객체의 인스턴스 참조 없이도 메소드를 호출할 수 있습니다. 이는 객체의 상태에 영향을 받지 않을 때 유리하며, 객체 전체를 전달할 필요 없이 필요한 데이터만 전달함으로써 메모리 관리와 성능 최적화에 도움이 됩니다. 또한, 네이티브 코드의 복잡성을 줄이고 잠재적 오류 가능성을 감소시킵니다.



## JNI Crash

### SIGSEGV (Segmentation Fault)
- 유효하지 않은 메모리 참조   
- JNI 환경에서 이러한 크래시가 일어나는 주요 원인 중 하나는 네이티브 코드가 이미 해제된 메모리를 참조하려 할때, 또는 잘못된 포인터를 사용할 때 

    ```Kotlin

    class MainActivity : AppCompatActivity() {

        override fun onCreate(savedInstanceState: Bundle?) {
            super.onCreate(savedInstanceState)

            accessInvalidMemory()
        }

        external fun accessInvalidMemory(): String

        companion object {
            // Used to load the 'jnitestapplication' library on application startup.
            init {
                System.loadLibrary("jnitestapplication")
            }
        }
    }
    ```

    ```cpp
    extern "C"
    JNIEXPORT void JNICALL
    Java_app_duckie_jnitestapplication_MainActivity_accessInvalidMemory(JNIEnv *env, jobject thiz) {
        // 임의의 메모리 포인터 생성
        int* invalidPointer = nullptr;

        // nullptr 참조 시도, SIGSEGV 발생
        int crashTrigger = *invalidPointer;
    }
    ```

    ````
    Fatal signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0 in tid 27434 (testapplication), pid 27434 (testapplication)

    2024-04-27 16:58:36.539 27478-27478 DEBUG                   pid-27478                            A  Cmdline: app.duckie.jnitestapplication
    2024-04-27 16:58:36.539 27478-27478 DEBUG                   pid-27478                            A  pid: 27434, tid: 27434, name: testapplication  >>> app.duckie.jnitestapplication <<<
    2024-04-27 16:58:36.539 27478-27478 DEBUG                   pid-27478                            A        #00 pc 000000000002453c  /data/app/~~ixTjeNOM4pw45LSSWYhgPw==/app.duckie.jnitestapplication-PcYURrjjEZ2je1-HdreMRw==/base.apk!libjnitestapplication.so (offset 0x80000) (Java_app_duckie_jnitestapplication_MainActivity_accessInvalidMemory+20) (BuildId: 2295d3b7007bb9071154a13d22bb75853f751b4d)
    2024-04-27 16:58:36.539 27478-27478 DEBUG                   pid-27478                            A        #06 pc 0000000000000894  /data/app/~~ixTjeNOM4pw45LSSWYhgPw==/app.duckie.jnitestapplication-PcYURrjjEZ2je1-HdreMRw==/base.apk (app.duckie.jnitestapplication.MainActivity.onCreate+0)
    ````

### SEGV_ACCERR (Signal 11, code 2) 

- 일반적으로 프로세스가 접근 권한이 없는 메모리 영역에 접근하려 할 때 발생
- 이는 메모리 읽기, 쓰기 또는 실행 권한이 부여되지 않은 주소를 참조할 때 발생
- JNI 환경에서 이러한 오류가 발생하는 예시 : 네이티브 코드에서 적절하지 않게 보호된 메모리 영역에 액세스하려고 할 때 발생


    ```cpp
    #include <jni.h>
    #include <string>
    #include <unistd.h>
    #include <sys/mman.h>

    extern "C"
    JNIEXPORT void JNICALL
    Java_app_duckie_jnitestapplication_MainActivity_attemptUnsafeMemoryModification(JNIEnv *env, jobject obj) {
        // 메모리 페이지 할당
        size_t pageSize = sysconf(_SC_PAGESIZE);
        void* memory = mmap(NULL, pageSize, PROT_READ, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

        if (memory == MAP_FAILED) {
            // 메모리 할당 실패 처리
            return;
        }

        // 할당된 메모리 페이지에 쓰기 시도 - PROT_READ 만 설정되어 있으므로, 쓰기 시도 시 SEGV_ACCERR 발생
        int* p = (int*)memory;
        *p = 123;  // 보호된 메모리에 쓰기 시도

        // 사용 완료 후 메모리 해제
        munmap(memory, pageSize);
    }
    ```

    ```
    2024-04-27 17:37:53.513  1849-1849  libc                    app.duckie.jnitestapplication        A  Fatal signal 11 (SIGSEGV), code 2 (SEGV_ACCERR), fault addr 0x778fbaa000 in tid 1849 (testapplication), pid 1849 (testapplication)

    2024-04-27 17:37:53.680  1908-1908  DEBUG                   pid-1908                             A  Cmdline: app.duckie.jnitestapplication
    2024-04-27 17:37:53.680  1908-1908  DEBUG                   pid-1908                             A  pid: 1849, tid: 1849, name: testapplication  >>> app.duckie.jnitestapplication <<<
    2024-04-27 17:37:53.680  1908-1908  DEBUG                   pid-1908                             A        #00 pc 0000000000024660  /data/app/~~jm-OSTpdilKLX2iHXlUkug==/app.duckie.jnitestapplication-pmP8_lkBdg9rkEwCvDc6Sg==/base.apk!libjnitestapplication.so (offset 0x4000) (Java_app_duckie_jnitestapplication_MainActivity_attemptUnsafeMemoryModification+104) (BuildId: 57c06ce88d0bcdcad8f681196a51ba80e7986583)
    2024-04-27 17:37:53.680  1908-1908  DEBUG                   pid-1908                             A        #06 pc 0000000000000d64  /data/app/~~jm-OSTpdilKLX2iHXlUkug==/app.duckie.jnitestapplication-pmP8_lkBdg9rkEwCvDc6Sg==/base.apk (app.duckie.jnitestapplication.MainActivity.onCreate+0)
    ```
