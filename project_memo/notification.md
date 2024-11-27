# Notification 서비스 확장 구조 기록

```typescript
// Some code
import {MessagePayloadWithTextRequired} from "@server/gateways/SlackNotificationSender";


export interface NotificationSender {
    sendSlackMessage(messagePayload: MessagePayloadWithTextRequired): Promise<void>;
}

export class NotificationService {
    constructor(private sender: NotificationSender) {}

    async notifySlack( messagePayload : MessagePayloadWithTextRequired) {
        return this.sender.sendSlackMessage(messagePayload);
    }
}

```





```typescript
// Some code
import {MessagePayloadWithTextRequired} from "@server/gateways/SlackNotificationSender";
import {KakaoMessagePayload, KakaoMessageResponse} from "@server/gateways/KakaoNotificationSender";


export interface SlackNotification {
    sendSlackMessage(messagePayload: MessagePayloadWithTextRequired): Promise<void>;
    
}
export interface KakaoNotification{
    sendKakaoMessage(messagePayload:KakaoMessagePayload):Promise<KakaoMessageResponse>
}

export interface NotificationSenders{
    slack?:SlackNotification;
    kakao?:KakaoNotification;
}

export class NotificationService {
    constructor(private readonly sender: NotificationSenders,
    ) {}

    async notifySlack( messagePayload : MessagePayloadWithTextRequired) {
        return this.sender.slack?.sendSlackMessage(messagePayload);
    }

    async notifyKakao(messagePayload:KakaoMessagePayload):Promise<KakaoMessageResponse|undefined>{
        return this.sender.kakao?.sendKakaoMessage(messagePayload);
    }
}



```



1. **다중 알림 채널 지원**:
   * **기존 코드**: 기존 구현은 Slack을 통한 알림 전송만 지원했습니다.
   * **변경된 코드**: 변경된 코드는 Slack과 Kakao를 통한 알림 전송을 모두 지원합니다. 이를 위해 각 알림 유형에 대해 별도의 인터페이스(`SlackNotification` 및 `KakaoNotification`)를 정의하고, 이를 `NotificationSenders` 인터페이스로 결합했습니다.
2. **인터페이스 분리**:
   * Slack과 Kakao 알림에 대해 별도의 인터페이스를 정의함으로써 인터페이스 분리 원칙을 준수합니다. 이를 통해 각 알림 유형을 독립적으로 구현할 수 있으며 불필요한 종속성을 피할 수 있습니다.
3. **선택적 알림 채널**:
   * `NotificationSenders` 인터페이스는 선택적 속성(`slack?` 및 `kakao?`)을 사용합니다. 이 설계는 하나 또는 두 개의 알림 채널을 유연하게 사용할 수 있도록 합니다.
4. **기능 확장**:
   * `NotificationService` 클래스는 이제 Slack(`notifySlack`)과 Kakao(`notifyKakao`) 알림을 모두 처리하는 메서드를 포함하여 기능이 확장되었습니다.
