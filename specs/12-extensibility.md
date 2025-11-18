# 13. 拡張性の考慮

## 原則：オープン・クローズド原則（OCP）とneverthrowによる合成可能な設計

```typescript
import { Result, ResultAsync, ok, err } from "neverthrow";

// domain/interfaces/IMenuOptionProvider.ts
export interface IMenuOptionProvider {
  getOptions(menuId: string): ResultAsync<MenuOption[], DomainError>;
  validateOptions(options: SelectedOption[]): Result<void, DomainError>;
}

// 将来の拡張例：AIによる動的オプションプロバイダー
export class AIOptionProvider implements IMenuOptionProvider {
  constructor(
    private menuRepository: IMenuRepository,
    private aiService: IAIService,
    private inventoryService: IInventoryService
  ) {}

  getOptions(menuId: string): ResultAsync<MenuOption[], DomainError> {
    return this.menuRepository
      .findById(menuId)
      .andThen((menu) => {
        if (!menu) return err(DomainError.menuNotFound());
        return ok(menu);
      })
      .andThen((menu) =>
        // 在庫情報を取得
        this.inventoryService
          .checkAvailability(menu.options)
          .andThen((availableOptions) =>
            // AIによる推奨オプション
            this.aiService
              .getRecommendations(menuId, availableOptions)
              .map((recommendations) =>
                this.mergeOptions(availableOptions, recommendations)
              )
          )
      );
  }

  validateOptions(options: SelectedOption[]): Result<void, DomainError> {
    // リアルタイムの在庫確認
    return this.inventoryService.validateStock(options).match(
      () => ok(undefined),
      (error) => err(error)
    );
  }

  private mergeOptions(
    available: MenuOption[],
    recommended: MenuOption[]
  ): MenuOption[] {
    // 重複を除いてマージ
    const merged = new Map<string, MenuOption>();
    available.forEach((opt) => merged.set(opt.id, opt));
    recommended.forEach((opt) =>
      merged.set(opt.id, { ...opt, isRecommended: true })
    );
    return Array.from(merged.values());
  }
}

// domain/interfaces/INotificationService.ts
export interface INotificationService {
  notify(notification: Notification): ResultAsync<void, DomainError>;
}

// 将来の実装例：マルチチャネル通知
export class MultiChannelNotificationService implements INotificationService {
  constructor(
    private emailService: IEmailService,
    private smsService: ISMSService,
    private pushService: IPushService,
    private slackService: ISlackService,
    private userPreferenceService: IUserPreferenceService
  ) {}

  notify(notification: Notification): ResultAsync<void, DomainError> {
    return this.userPreferenceService
      .getPreferences(notification.recipient)
      .andThen((preferences) => {
        const notifications: ResultAsync<void, DomainError>[] = [];

        if (preferences.email) {
          notifications.push(
            this.emailService.send({
              to: preferences.email,
              subject: notification.subject,
              body: notification.content,
            })
          );
        }

        if (preferences.phone && this.shouldSendSMS(notification.type)) {
          notifications.push(
            this.smsService.send({
              to: preferences.phone,
              message: this.formatForSMS(notification.content),
            })
          );
        }

        if (preferences.pushToken) {
          notifications.push(
            this.pushService.send({
              token: preferences.pushToken,
              title: notification.subject,
              body: notification.content,
            })
          );
        }

        // すべての通知を並列実行（一部失敗しても続行）
        return ResultAsync.combineWithAllErrors(notifications)
          .map(() => undefined)
          .orElse(() => ok(undefined)); // 通知エラーは無視
      });
  }

  private shouldSendSMS(type: NotificationType): boolean {
    return [
      NotificationType.ORDER_CONFIRMATION,
      NotificationType.ORDER_CANCELLED,
    ].includes(type);
  }

  private formatForSMS(content: string): string {
    return content.length > 140 ? content.substring(0, 137) + "..." : content;
  }
}
```

## まとめ

この仕様書は、neverthrowライブラリを使用してtry-catchを完全に排除し、Result型による型安全なエラーハンドリングを実現しています。

### 主な特徴

1. **型安全性**: すべてのエラーが型で表現され、コンパイル時にエラーハンドリングの漏れを検出
2. **合成可能性**: `andThen`、`map`、`orElse`などのメソッドチェーンで処理を組み合わせ可能
3. **明示的なエラー処理**: エラーの可能性がある処理はすべてResult型で表現
4. **Railway Oriented Programming**: 成功と失敗のトラックを明確に分離
5. **非同期処理の統一**: ResultAsyncによる一貫した非同期エラーハンドリング

### neverthrowの利点

- エラーハンドリングが強制される（忘れることがない）
- エラーの型が明確で、どんなエラーが起こりうるかが明示的
- 関数型プログラミングのパターンが適用しやすい
- テストが書きやすく、予測可能な動作を保証

この設計により、保守性が高く、バグの少ない堅牢なシステムを構築することができます。
