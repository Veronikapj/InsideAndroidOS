
# Zygote 요약

## 소개
- 새 앱 런타임 환경 시작 및 초기화 제공
- 자바 앱 시작 시간 문제 해결

## 메모리 관리
1. **OS 리소스 확보, 다른 프로그램 디스크로 스왑**
2. **JVM 시작**
3. **기본 자바 라이브러리 rt.jar 로드**
4. **기본 라이브러리 static 초기화**
5. **애플리케이션 재귀적 로드**
6. **새로 로딩된 클래스 초기화**
7. **minor gc 실행**
8. **Main 메소드 실행**

## Zygote와 안드로이드 관계
- `init` 시스템 브링업 단계에서 Zygote 시작
- 전체 안드로이드 프레임워크 미리 로딩 및 스스로 초기화
- 시스템 초기화 시 모든 라이브러리 로드
- 소켓 연결 대기

## COW 기법(Copy-on-Write)
- https://code-lab1.tistory.com/58
- 수정 전 원본 리소스 공유, 필요 시 실제 메모리 페이지 복사
- ex) fork 시스템 호출 사용
- 논란의 원문(170p 하단) : Even if some child process were to write on every single memory page (something that quite literally never happens), the cost of allocating the new memory is **amortized** over the life of the process. It is not incurred at initialization. 

## 애플리케이션 생성 과정
- Zygote 복제(clone)로 새 프로세스 생성
- 메모리 페이지 공유, 필요 시 복사

## Zygote 시작
- 32bit, 64bit 버전 시작
- 자바 프로세스
- 네이티브 코드 래퍼`app_process`

## 런타임 초기화
- ART 런타임 단 한 번 초기화
- 자바 런타임 서비스 제공 목표
- 실행 과정 중략 -> Android VM 시작할 준비 완료
- ART   
  - VM 환경 설정 옵션  
    `-verbose:gc` : 시스템 가비지 컬렉션 관련 정보를 로그로 저장  
    `-generate-debug-info` : 시스템이 업데이트 된 이후 dex2oat 에게 자기가 빌드하는 이미지에 디버그 정보 추가 



