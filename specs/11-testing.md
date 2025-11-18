# 12. テスト仕様（neverthrow版）

## 原則：Result型を活用した予測可能なテスト

```typescript
import { describe, it, expect, beforeEach, jest } from "@jest/globals";
import { ok, err } from "neverthrow";

// tests/services/OrderService.test.ts
describe("OrderService", () => {
  let orderService: OrderService;
  let mockOrderRepository: jest.Mocked<IOrderRepository>;
  let mockMenuRepository: jest.Mocked<IMenuRepository>;
  let mockUserRepository: jest.Mocked<IUserRepository>;
  let mockEventBus: jest.Mocked<IEventBus>;

  beforeEach(() => {
    mockOrderRepository = {
      save: jest.fn(),
      findByOrderNumber: jest.fn(),
      findByUserIdAndMenuId: jest.fn(),
      getNextSequence: jest.fn(),
      // ... その他のメソッド
    };

    mockMenuRepository = {
      findById: jest.fn(),
      // ... その他のメソッド
    };

    mockUserRepository = {
      saveProfile: jest.fn(),
      // ... その他のメソッド
    };

    mockEventBus = {
      publish: jest.fn(),
    };

    orderService = new OrderService(
      mockOrderRepository,
      mockMenuRepository,
      mockUserRepository,
      mockEventBus
    );
  });

  describe("createOrder", () => {
    it("正常に注文を作成できる", async () => {
      // Arrange
      const input: CreateOrderInput = {
        userId: "user-123",
        menuId: "menu-456",
        userInfo: {
          department: "営業部",
          name: "山田太郎",
          gender: Gender.MALE,
          ageGroup: AgeGroup.THIRTIES,
        },
        selectedOptions: [],
      };

      const mockMenu: Menu = {
        id: "menu-456",
        name: "日替わり弁当",
        price: Money.create(500).value,
        availableDate: new Date("2024-01-20"),
        orderDeadline: new Date("2024-01-18T17:00:00"),
        status: MenuStatus.ACTIVE,
        // ... その他のフィールド
      };

      mockMenuRepository.findById.mockResolvedValue(ok(mockMenu));
      mockOrderRepository.findByUserIdAndMenuId.mockResolvedValue(ok(null));
      mockOrderRepository.getNextSequence.mockResolvedValue(ok(1));
      mockOrderRepository.save.mockResolvedValue(
        ok({
          id: "order-789",
          orderNumber: "20240120-0001",
          // ... その他のフィールド
        } as Order)
      );
      mockUserRepository.saveProfile.mockResolvedValue(ok(undefined));

      // Act
      const result = await orderService.createOrder(input);

      // Assert
      expect(result.isOk()).toBe(true);
      if (result.isOk()) {
        expect(result.value.orderNumber).toBe("20240120-0001");
        expect(result.value.message).toBe("注文が確定しました");
      }
      expect(mockEventBus.publish).toHaveBeenCalled();
    });

    it("締切を過ぎている場合はエラーを返す", async () => {
      // Arrange
      const input: CreateOrderInput = {
        // ... 入力データ
      };

      const mockMenu: Menu = {
        // ... メニューデータ
        orderDeadline: new Date("2020-01-01"), // 過去の日付
      };

      mockMenuRepository.findById.mockResolvedValue(ok(mockMenu));

      // Act
      const result = await orderService.createOrder(input);

      // Assert
      expect(result.isErr()).toBe(true);
      if (result.isErr()) {
        expect(result.error.code).toBe("ORDER_DEADLINE_PASSED");
      }
    });

    it("既に注文済みの場合はエラーを返す", async () => {
      // Arrange
      const existingOrder: Order = {
        // ... 既存注文データ
        status: OrderStatus.CONFIRMED,
      };

      mockMenuRepository.findById.mockResolvedValue(ok(mockMenu));
      mockOrderRepository.findByUserIdAndMenuId.mockResolvedValue(
        ok(existingOrder)
      );

      // Act
      const result = await orderService.createOrder(input);

      // Assert
      expect(result.isErr()).toBe(true);
      if (result.isErr()) {
        expect(result.error.code).toBe("DUPLICATE_ORDER");
      }
    });
  });
});
```
