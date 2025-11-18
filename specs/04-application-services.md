# 5. アプリケーションサービス

## 原則：neverthrowのメソッドチェーンによる合成可能なエラーハンドリング

```typescript
import { Result, ResultAsync, ok, err, combine } from "neverthrow";
import { generateUUID, generateSecureToken } from "../utils";

// application/services/OrderService.ts
export class OrderService {
  constructor(
    private orderRepository: IOrderRepository,
    private menuRepository: IMenuRepository,
    private userRepository: IUserRepository,
    private eventBus: IEventBus
  ) {}

  createOrder(
    input: CreateOrderInput
  ): ResultAsync<CreateOrderOutput, DomainError> {
    // 入力検証
    const validationResult = OrderValidator.validateCreateOrder(input);
    if (validationResult.isErr()) {
      return ResultAsync.fromSafePromise(
        Promise.resolve(err(validationResult.error))
      );
    }

    // neverthrowのandThenチェーンを使用したパイプライン処理
    return this.menuRepository
      .findById(input.menuId)
      .andThen((menu) => {
        if (!menu) {
          return err(DomainError.menuNotFound(input.menuId));
        }
        if (new Date() > menu.orderDeadline) {
          return err(DomainError.orderDeadlinePassed());
        }
        if (menu.status !== MenuStatus.ACTIVE) {
          return err(DomainError.menuNotAvailable());
        }
        return ok(menu);
      })
      .andThen((menu) =>
        // 既存注文の確認
        this.orderRepository
          .findByUserIdAndMenuId(input.userId, input.menuId)
          .andThen((existingOrder) => {
            if (
              existingOrder &&
              existingOrder.status !== OrderStatus.CANCELLED
            ) {
              return err(DomainError.duplicateOrder());
            }
            return ok(menu);
          })
      )
      .andThen((menu) =>
        // 注文番号の生成
        this.orderRepository
          .getNextSequence(menu.availableDate)
          .andThen((sequence) =>
            OrderNumber.generate(menu.availableDate, sequence).map(
              (orderNumber) => ({ menu, orderNumber })
            )
          )
      )
      .andThen(({ menu, orderNumber }) => {
        // 注文作成
        const orderResult = createOrder({
          id: generateUUID(),
          userId: input.userId,
          menuId: input.menuId,
          orderNumber: orderNumber.toString(),
          userInfo: input.userInfo,
          selectedOptions: input.selectedOptions,
        });

        if (orderResult.isErr()) {
          return err(orderResult.error);
        }

        return ok({ order: orderResult.value, menu });
      })
      .andThen(({ order, menu }) =>
        // 注文を保存
        this.orderRepository
          .save(order)
          .andThen(
            (savedOrder) =>
              // プロファイル保存（エラーは無視）
              this.userRepository
                .saveProfile(input.userId, input.userInfo)
                .map(() => savedOrder)
                .orElse(() => ok(savedOrder)) // プロファイル保存エラーは無視
          )
          .andThen((savedOrder) =>
            // イベント発行
            ResultAsync.fromSafePromise(
              this.eventBus.publish(new OrderCreatedEvent(savedOrder))
            ).map(() => ({
              order: savedOrder,
              orderNumber: savedOrder.orderNumber,
              message: "注文が確定しました",
            }))
          )
      );
  }

  modifyOrder(
    input: ModifyOrderInput
  ): ResultAsync<ModifyOrderOutput, DomainError> {
    // 複数の非同期処理を組み合わせる
    const orderResult = this.orderRepository.findByOrderNumber(
      input.orderNumber
    );
    const userResult = this.userRepository.findBySessionToken(
      input.sessionToken
    );

    return ResultAsync.combine([orderResult, userResult])
      .andThen(([order, user]) => {
        if (!order) {
          return err(DomainError.orderNotFound(input.orderNumber));
        }
        if (!user || user.id !== order.userId) {
          return err(DomainError.unauthorized("注文の変更権限がありません"));
        }
        return ok({ order, user });
      })
      .andThen(({ order, user }) =>
        // メニューの取得と締切確認
        this.menuRepository.findById(order.menuId).andThen((menu) => {
          if (!menu) {
            return err(DomainError.menuNotFound());
          }
          if (new Date() > menu.orderDeadline) {
            return err(DomainError.orderDeadlinePassed());
          }
          return ok({ order, user, menu });
        })
      )
      .andThen(({ order, user }) => {
        // 注文の更新
        const updatedOrder: Order = {
          ...order,
          userInfo: input.userInfo,
          selectedOptions: input.selectedOptions,
          status: OrderStatus.MODIFIED,
          modifiedAt: new Date(),
        };

        return this.orderRepository
          .update(updatedOrder)
          .andThen((savedOrder) =>
            // プロファイル更新
            this.userRepository
              .saveProfile(user.id, input.userInfo)
              .map(() => savedOrder)
              .orElse(() => ok(savedOrder))
          )
          .andThen((savedOrder) =>
            // イベント発行
            ResultAsync.fromSafePromise(
              this.eventBus.publish(new OrderModifiedEvent(savedOrder))
            ).map(() => ({
              order: savedOrder,
              message: "注文を変更しました",
            }))
          );
      });
  }

  cancelOrder(
    input: CancelOrderInput
  ): ResultAsync<CancelOrderOutput, DomainError> {
    return this.orderRepository
      .findByOrderNumber(input.orderNumber)
      .andThen((order) => {
        if (!order) {
          return err(DomainError.orderNotFound(input.orderNumber));
        }
        if (order.status === OrderStatus.CANCELLED) {
          return err(
            new DomainError("すでにキャンセル済みです", "ALREADY_CANCELLED")
          );
        }
        return ok(order);
      })
      .andThen((order) =>
        // 権限確認
        this.userRepository
          .findBySessionToken(input.sessionToken)
          .andThen((user) => {
            if (!user || user.id !== order.userId) {
              return err(
                DomainError.unauthorized("キャンセル権限がありません")
              );
            }
            return ok(order);
          })
      )
      .andThen((order) =>
        // 締切確認
        this.menuRepository.findById(order.menuId).andThen((menu) => {
          if (!menu) {
            return err(DomainError.menuNotFound());
          }
          if (new Date() > menu.orderDeadline) {
            return err(DomainError.orderDeadlinePassed());
          }
          return ok(order);
        })
      )
      .andThen((order) => {
        // キャンセル処理
        const cancelledOrder: Order = {
          ...order,
          status: OrderStatus.CANCELLED,
          cancelledAt: new Date(),
        };

        return this.orderRepository
          .update(cancelledOrder)
          .andThen((savedOrder) =>
            ResultAsync.fromSafePromise(
              this.eventBus.publish(new OrderCancelledEvent(savedOrder))
            ).map(() => ({
              success: true,
              message: "注文をキャンセルしました",
            }))
          );
      });
  }

  getOrderHistory(userId: string): ResultAsync<Order[], DomainError> {
    return this.orderRepository.findByUserId(userId, {
      limit: 10,
      offset: 0,
      orderBy: "orderedAt",
      orderDirection: "DESC",
    });
  }
}

// application/services/UserService.ts
export class UserService {
  constructor(private userRepository: IUserRepository) {}

  anonymousLogin(): ResultAsync<AnonymousLoginOutput, DomainError> {
    const sessionToken = generateSecureToken();
    const user: User = {
      id: generateUUID(),
      sessionToken,
      createdAt: new Date(),
      lastAccessedAt: new Date(),
    };

    return this.userRepository.save(user).map((savedUser) => ({
      userId: savedUser.id,
      sessionToken: savedUser.sessionToken,
    }));
  }

  getUserProfile(userId: string): ResultAsync<UserProfile | null, DomainError> {
    return this.userRepository
      .findById(userId)
      .map((user) => user?.profile || null);
  }
}
```
