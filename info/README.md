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
- 여기서는 모듈을 메모리에 관리하고, 메모리 키는 입력받은 모듈명으로 사용하였다. 값은 SampleModule로 부가 정보를 담아서 값을 넣었다.
- SampleModule은 아래와 같이 구성 됨
```
private Module          module;
private List<Slot>      slots  = List.of();
```

```
private final Map<String, SampleModule> modules        = Collections.synchronizedMap(new HashMap<>()); // 메모리 관리용

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
private Mono<SampleModule> getModule(final String name)
{
  return Mono.fromCallable(() -> {
      return this.modules.get(name);
  });
}

public Flux<SampleModule> getAllModules()
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

## 4. slot 관리
- 내부에서 사용하는 용도 SampleSlot객체로 값 받기
```
@Override
public Flux<SampleSlot> getSlotList(final String moduleName)
{
  return getModule(moduleName).flatMapMany(module -> {
      return Flux.fromIterable(module.getSlots())
                 .map(slot -> {
                     try
                     {
                         CK_SLOT_INFO slotInfo = slot.getModule()
                                                     .getPKCS11Module()
                                                     .C_GetSlotInfo(slot.getSlotID());
                         CK_TOKEN_INFO tokenInfo = slot.getModule()
                                                       .getPKCS11Module()
                                                       .C_GetTokenInfo(slot.getSlotID());
                         return SampleSlot.builder()
                                        .moduleName(moduleName)
                                        .slotId(slot.getSlotID())
                                        .tokenInfo(tokenInfo)
                                        .slotInfo(slotInfo)
                                        .build();
                     }
                     catch ( PKCS11Exception e )
                     {
                         throw new Exception(e);
                     }
                 });
  })
                              .cast(Slot.class);

}
```

- pkcs11 SampleSlot으로 Slot정보 받기
```
protected Mono<Slot> getIaikSlot(final SampleSlot sampleSlot)
{
  return getModule(sampleSlot.getModuleName()).map(SampleModule::getSlots) // 메모리의 모듈을 읽어와서 해당 모듈의 슬으로 for
                                            .flatMapMany(Flux::fromIterable)
                                            .filter(slot -> slot.getSlotID() == sampleSlot.getSlotId()) // 요청슬롯과 메모리 슬롯이 동일한 것을 리턴
                                            .next();
}
```

- label값으로 slot리스트정보 받기
```
public Flux<SampleSlot> findSlot(final String tokenLabel)
{
  return Flux.fromIterable(new HashSet<>(this.modules.keySet()))
             .flatMap(this::getSlotList)
             .filter(sampleSlot -> StringUtils.equals(tokenLabel, sampleSlot.getTokenLabel()));
}
```

- label AND slot시리얼넘버로 slot단일정보 받기 (리스트로 떨어지는 경우가 있는지 체크 못해봄..)
```
public Mono<SampleSlot> findSlot(final String tokenLabel, final String serialNumber)
{
  return findSlot(tokenLabel).filter(sampleSlot -> StringUtils.equals(serialNumber, sampleSlot.getSerialNumber()))
                             .next(); // 다수일 수가 있을까..?
}
```

- slot시이얼넘버로 slot정보 받기
```
public Mono<SampleSlot> findSlotBySerialNumber(final String serialNumber)
{
  return getAllSlot().filter(sampleSlot -> StringUtils.equals(serialNumber, sampleSlot.getSerialNumber()))
                     .next();
}
```

- 모든 slot를 조회 하기
```
public Flux<SampleSlot> getAllSlot()
{
  return getAllModules().map(LinkModule::getName)
                        .flatMap(this::getSlotList);
}
```
