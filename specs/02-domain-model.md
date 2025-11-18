# 3. ドメインモデル設計

## 3.1 エンティティ定義

```typescript
import { Result, ok, err } from "neverthrow";

// domain/entities/User.ts
export interface User {
  id: string; // UUID (匿名認証ID)
  sessionToken: string; // セッション管理用
  createdAt: Date;
  lastAccessedAt: Date;

  // 原則：値オブジェクトパターンで個人情報を管理
  profile?: UserProfile;
}

export interface UserProfile {
  department: string;
  name: string;
  gender: Gender;
  ageGroup: AgeGroup;
}

// domain/entities/Menu.ts
export interface Menu {
  id: string;
  name: string;
  description: string;
  imageUrl: string;
  price: Money; // 値オブジェクト
  availableDate: Date; // 受け渡し日
  orderDeadline: Date; // 注文締切（前々日17時）
  maxQuantity?: number;
  status: MenuStatus;
  options: MenuOption[];
  createdAt: Date;
  updatedAt: Date;
}

// domain/entities/Order.ts
export interface Order {
  id: string;
  userId: string;
  menuId: string;
  orderNumber: string; // 変更・キャンセル用の注文番号
  userInfo: OrderUserInfo;
  selectedOptions: SelectedOption[];
  status: OrderStatus;
  orderedAt: Date;
  modifiedAt?: Date;
  cancelledAt?: Date;
}

// Orderのファクトリー関数（原則：不変性とバリデーション）
export const createOrder = (
  params: CreateOrderParams
): Result<Order, DomainError> => {
  // ビジネスルールの検証
  const validationResult = validateOrderCreation(params);
  if (validationResult.isErr()) {
    return err(validationResult.error);
  }

  const order: Order = {
    id: params.id,
    userId: params.userId,
    menuId: params.menuId,
    orderNumber: params.orderNumber,
    userInfo: params.userInfo,
    selectedOptions: params.selectedOptions,
    status: OrderStatus.CONFIRMED,
    orderedAt: new Date(),
    modifiedAt: undefined,
    cancelledAt: undefined,
  };

  return ok(order);
};

// domain/entities/MenuOption.ts
export interface MenuOption {
  id: string;
  menuId: string;
  optionGroupId: string;
  optionGroup: OptionGroup;
  isRequired: boolean;
  sortOrder: number;
}

export interface OptionGroup {
  id: string;
  name: string; // "ご飯の種類"
  type: OptionType; // SELECT, RADIO, CHECKBOX
  options: OptionItem[];
  templateId?: string; // テンプレート参照
}

export interface OptionItem {
  value: string;
  label: string;
  sortOrder: number;
}

// domain/types/Enums.ts
export enum Gender {
  MALE = "MALE",
  FEMALE = "FEMALE",
  OTHER = "OTHER",
}

export enum AgeGroup {
  UNDER_20 = "UNDER_20",
  TWENTIES = "20-29",
  THIRTIES = "30-39",
  FORTIES = "40-49",
  FIFTIES = "50-59",
  OVER_60 = "OVER_60",
}

export enum MenuStatus {
  DRAFT = "DRAFT",
  ACTIVE = "ACTIVE",
  CLOSED = "CLOSED",
  CANCELLED = "CANCELLED",
}

export enum OrderStatus {
  CONFIRMED = "CONFIRMED",
  MODIFIED = "MODIFIED",
  CANCELLED = "CANCELLED",
}

export enum OptionType {
  SELECT = "SELECT",
  RADIO = "RADIO",
  CHECKBOX = "CHECKBOX",
}
```

## 3.2 値オブジェクト

```typescript
import { Result, ok, err } from "neverthrow";
import { format } from "date-fns";

// domain/valueObjects/Money.ts
export class Money {
  private constructor(
    private readonly amount: number,
    private readonly currency: string = "JPY"
  ) {}

  // 原則：ファクトリーメソッドでバリデーション
  static create(
    amount: number,
    currency: string = "JPY"
  ): Result<Money, DomainError> {
    if (amount < 0) {
      return err(
        new DomainError("金額は0以上で設定してください", "INVALID_AMOUNT")
      );
    }
    if (!["JPY", "USD", "EUR"].includes(currency)) {
      return err(
        new DomainError("サポートされていない通貨です", "INVALID_AMOUNT")
      );
    }
    return ok(new Money(amount, currency));
  }

  getValue(): number {
    return this.amount;
  }
  getCurrency(): string {
    return this.currency;
  }

  add(other: Money): Result<Money, DomainError> {
    if (this.currency !== other.currency) {
      return err(
        new DomainError("異なる通貨は加算できません", "CURRENCY_MISMATCH")
      );
    }
    return Money.create(this.amount + other.amount, this.currency);
  }

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }

  toString(): string {
    return `${this.currency} ${this.amount.toLocaleString()}`;
  }
}

// domain/valueObjects/OrderNumber.ts
export class OrderNumber {
  private constructor(private readonly value: string) {}

  // 原則：バリデーションをResult型で表現
  static create(value: string): Result<OrderNumber, DomainError> {
    if (!this.validate(value)) {
      return err(
        new DomainError("注文番号の形式が不正です", "INVALID_ORDER_NUMBER")
      );
    }
    return ok(new OrderNumber(value));
  }

  static generate(
    date: Date,
    sequence: number
  ): Result<OrderNumber, DomainError> {
    if (sequence < 0 || sequence > 9999) {
      return err(
        new DomainError("シーケンス番号が範囲外です", "INVALID_ORDER_NUMBER")
      );
    }
    const dateStr = format(date, "yyyyMMdd");
    const seq = sequence.toString().padStart(4, "0");
    return OrderNumber.create(`${dateStr}-${seq}`);
  }

  private static validate(value: string): boolean {
    return /^\d{8}-\d{4}$/.test(value);
  }

  getValue(): string {
    return this.value;
  }
  toString(): string {
    return this.value;
  }
}

// domain/valueObjects/OrderUserInfo.ts
export interface OrderUserInfo {
  department: string;
  name: string;
  gender: Gender;
  ageGroup: AgeGroup;
}

// domain/valueObjects/SelectedOption.ts
export interface SelectedOption {
  optionGroupId: string;
  selectedValue: string | string[]; // CHECKBOXの場合は複数選択
}
```
