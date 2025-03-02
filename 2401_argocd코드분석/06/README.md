# ArgoCD Notification Controller 분석

TODO: 북마크별로 상세 설명 필요

## 1. Notification Controller 생성 과정

### 1-1. `NewFactory()`
`NewFactory()`는 `Settings` 기반으로 `apiFactory`를 초기화하고, ConfigMap/Secret을 조회하는 Lister를 설정한다.

- **Informer에 이벤트 핸들러를 등록**하여 Secret/ConfigMap 변경을 감시한다.
- 특정 Secret 또는 ConfigMap이 추가/삭제/수정될 경우 `invalidateIfHasName()`을 실행한다.
  - `obj`의 이름이 감시 대상(`name`)과 일치하면, 해당 네임스페이스의 API 캐시를 무효화한다.
- **캐시를 무효화(`apiMap[namespace] = nil`)하여 변경된 설정을 반영할 수 있도록 한다.**

### 1-2. `skipProcessingOpt()`
Object를 알림에서 제외할지 확인한다.
- `true`면 skip하고, `false`면 skip하지 않는다.

#### Skip 조건
1. `obj`가 `*unstructured.Unstructured` 타입이 아닌 경우
2. 애플리케이션이 `namespace` 또는 `applicationNamespaces`에 속하지 않은 경우
3. 애플리케이션의 `sync status`가 최신 상태가 아닌 경우

### 1-3. `alertDestinations()`
- 알림 목적지를 가져와서 `alertDestinationsOpt`에 할당한다.
- **legacy annotations와 새로운 annotations를 합쳐서 가져온다.**
  - 과거 버전 프로젝트에서도 알림 설정이 정상 동작하도록 보장하기 위함.

### 1-4. Controller 생성
`NewControllerWithNamespaceSupport()`
- `namespaceSupport`를 `true`로 설정하고 `NewController`를 호출.
- 내부적으로 **큐(queue)를 활용하여 이벤트 기반 알림 처리**를 수행함.
- **Informer에서 변경이 감지되면 `queue.Add()`를 호출하여 이벤트를 큐에 추가함.**

## 2. Notification Controller 실행 과정

### 2-1. `processQueueItem()` 실행
- Queue에서 이벤트를 꺼내와서 **EventSequence를 초기화**한다.
- **EventSequence는 알림 처리를 위한 이벤트 시퀀스 정보를 저장하는 구조체**이다.

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| `Delivered` | `[]NotificationDelivery` | 성공적으로 전송된 알림 목록 |
| `Errors` | `[]error` | 처리 중 발생한 오류 목록 |
| `Warnings` | `[]error` | 경고 메시지 목록 |

### 2-2. 처리 흐름
1. **Resource 조회** → 정상 조회되면 `resource` 할당.
2. **Skip 여부 판단** → 특정 조건 충족 시 알림 처리 스킵.
3. **Namespace 기반 API 설정 로드**
   - `namespaceSupport == true`일 때 전역 네임스페이스 활용.
   - `GetAPI()` 실행 시 `GetAPIsFromNamespace(DefaultNamespace)` 호출.
   - `GetAPIsFromNamespace(namespace)` 실행하여 특정 네임스페이스 API 설정 로드.
4. **ConfigMap 및 Secret 파일 로드** → `NewService`를 생성.
5. **API 객체 생성 완료 후 트리거 실행** → `Trigger svc.Run()`

## 3. 알림 전송 흐름 (`res` 처리)
```go
for _, cr := range res {
    c.metricsRegistry.IncTriggerEvaluationsCounter(trigger, cr.Triggered)
    
    if !cr.Triggered {
        for _, to := range destinations {
            notificationsState.SetAlreadyNotified(c.isSelfServiceConfigureApi(api), apiNamespace, trigger, cr, to, false)
        }
        continue
    }
    
    for _, to := range destinations {
        if changed := notificationsState.SetAlreadyNotified(c.isSelfServiceConfigureApi(api), apiNamespace, trigger, cr, to, true); !changed {
            logEntry.Infof("Notification about condition '%s.%s' already sent to '%v' using the configuration in namespace %s", trigger, cr.Key, to, apiNamespace)
            eventSequence.addDelivered(NotificationDelivery{
                Trigger:         trigger,
                Destination:     to,
                AlreadyNotified: true,
            })
        } else {
            logEntry.Infof("Sending notification about condition '%s.%s' to '%v' using the configuration in namespace %s", trigger, cr.Key, to, apiNamespace)
            if err := api.Send(un.Object, cr.Templates, to); err != nil {
                logEntry.Errorf("Failed to notify recipient %s defined in resource %s/%s: %v using the configuration in namespace %s", to, resource.GetNamespace(), resource.GetName(), err, apiNamespace)
                notificationsState.SetAlreadyNotified(c.isSelfServiceConfigureApi(api), apiNamespace, trigger, cr, to, false)
                c.metricsRegistry.IncDeliveriesCounter(trigger, to.Service, false)
                eventSequence.addError(fmt.Errorf("failed to deliver notification %s to %s: %v using the configuration in namespace %s", trigger, to, err, apiNamespace))
            } else {
                logEntry.Debugf("Notification %s was sent using the configuration in namespace %s", to.Recipient, apiNamespace)
                c.metricsRegistry.IncDeliveriesCounter(trigger, to.Service, true)
                eventSequence.addDelivered(NotificationDelivery{
                    Trigger:         trigger,
                    Destination:     to,
                    AlreadyNotified: false,
                })
            }
        }
    }
}
```

### 3-1. 실행 흐름 분석
1. `res`의 각 요소 `cr`을 순회하며, `trigger`가 발생했는지 확인.
2. 트리거가 발생하지 않았다면 해당 목적지(`to`)에 대해 **알림이 이미 전송되었음을 기록하고 종료**.
3. 트리거가 발생한 경우:
   - `notificationsState.SetAlreadyNotified()`를 호출하여 이미 알림이 전송된 상태인지 확인.
   - **이미 알림이 전송되었다면**, 로그를 남기고 `eventSequence.addDelivered()` 호출.
   - **새롭게 알림을 보내야 하는 경우**, `api.Send()`를 호출하여 알림 전송.
     - 전송 실패 시 로그를 남기고 실패 카운터 증가.
     - 전송 성공 시 성공 카운터 증가 및 `eventSequence.addDelivered()` 호출.

## 4. 결론
ArgoCD Notification Controller는 Informer를 활용하여 설정 변경을 감지하고, Queue를 이용하여 이벤트를 비동기적으로 처리한다. `processQueueItem()`을 통해 알림 전송 여부를 판단하고, 알림을 전송하며 그 결과를 `eventSequence`에 기록하는 방식으로 동작한다. 이를 통해 안정적으로 알림을 관리하고, 실패한 알림에 대한 재시도를 할 수 있도록 구성되어 있다.

