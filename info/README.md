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
- 핵심 소스만 정리이며, 모듈과 HMS제어를 위해서 관리하는 설계를 각 환경에서 다를 것이기에 제외한다.

## 1. 모듈 등록
- 여기서는 모듈을 메모리에 관리하고, 메모리 키는 입력받은 모듈명으로 사용하였다. 값은 MemoryModule로 부가 정보를 담아서 값을 넣었다.

```
private final Map<String, MemoryModule> modules        = Collections.synchronizedMap(new HashMap<>()); // 메모리 관리용

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
.doOnNext(module -> this.modules.put(name, module));
```

## 2. 모듈 getAll, get 
- 메모리에서 키의 값으로 모듈 값을 호출

```
private Mono<MemoryModule> getModule(final String name)
{
  return Mono.fromCallable(() -> {
      return this.modules.get(name);
  });
}

public Flux<MemoryModule> getAllModules()
{
  return Flux.fromIterable(new HashSet<>(this.modules.keySet())) //fromIterable은 for문의 역활이다.
             .flatMap(this::getModuleInfo);
}
```

## 3. 모듈 삭제 
- 키 값으로 메모리의 모듈을 삭제 

```
@Override
public Mono<Void> removeModule(final String name)
{
  return Mono.fromCallable(() -> this.modules.remove(name))
             .doOnNext(module -> {
                 try
                 {
                     module.getModule()
                           .finalize(null);
                 }
                 catch ( Throwable throwable )
                 {
                     log.warn("Fail to finalize. moduleName[{}]", name, throwable);
                 }
             })
             .then(Mono.fromRunnable(() -> this.moduleMonitors.remove(name)))
             .then();
}
```
