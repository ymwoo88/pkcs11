# plcs11 주요 import 객체 정보
- import iaik.pkcs.pkcs11.DefaultInitializeArgs;
- import iaik.pkcs.pkcs11.Module;
- import iaik.pkcs.pkcs11.Slot;
- import iaik.pkcs.pkcs11.TokenException;
- import iaik.pkcs.pkcs11.wrapper.CK_SLOT_INFO;
- import iaik.pkcs.pkcs11.wrapper.CK_TOKEN_INFO;
- import iaik.pkcs.pkcs11.wrapper.PKCS11Constants;
- import iaik.pkcs.pkcs11.wrapper.PKCS11Exception;

# 환경정보
- webflux 환경에서 pkcs11모듈 활용

## 1. 모듈 등록

```
Mono.fromCallable(() -> {
   AtomicReference<Module> moduleRef = new AtomicReference<>();
   moduleRef.set(Module.getInstance(libraryPath)); // dll 위치정보를 받아서 getInstance한다.
   
   DefaultInitializeArgs defaultInitializeArgs = new DefaultInitializeArgs(); // pkcs11에서 사용하는 아규먼트 관리객체
   defaultInitializeArgs.setOsLockingOk(Boolean.TRUE); // Locking 옵션 true
   
   moduleRef.get().initialize(defaultInitializeArgs); // AtomicReference객체에 initialize를 한다.
   return moduleRef.get(); // initialize한 객체를 반환 (타입 Module)
}))
.doOnNext(module -> { // 모듈 정보 확인
   log.info("A module is added. moduleName[{}], libraryPath[{}]", name, libraryPath);
})
```
