# 1. アーキテクチャ概要

## 原理原則

- **DDD（Domain-Driven Design）**: ビジネスロジックをドメイン層に集約し、技術的関心事から分離
- **レイヤードアーキテクチャ**: 責任を層ごとに分離し、依存関係を一方向に保つ
- **SOLID原則**: 特に依存性逆転原則（DIP）により、上位層が下位層の実装に依存しない設計
- **関数型エラーハンドリング**: neverthrowのResult型により例外を使わない明示的なエラーハンドリング
- **Railway Oriented Programming**: 成功と失敗のトラックを型で表現し、エラーハンドリングを合成可能にする

```typescript
// レイヤー構成
├── Domain Layer        // ビジネスロジック・エンティティ
├── Application Layer   // ユースケース・サービス
├── Infrastructure Layer // DB・外部サービス実装
└── Presentation Layer  // UI・コントローラー
```

## 依存パッケージ

```json
{
  "dependencies": {
    "neverthrow": "^6.0.0",
    "@supabase/supabase-js": "^2.39.0",
    "date-fns": "^3.0.0",
    "zod": "^3.22.0"
  }
}
```

# 2. エラー型の定義

## 原則：型安全なエラーハンドリング

neverthrowのResult型を使用して、すべてのエラーを型で表現する

```typescript
// domain/errors/DomainError.ts
export type DomainErrorCode =
  | "ORDER_DEADLINE_PASSED"
  | "DUPLICATE_ORDER"
  | "ORDER_NOT_FOUND"
  | "MENU_NOT_FOUND"
  | "MENU_NOT_AVAILABLE"
  | "UNAUTHORIZED"
  | "VALIDATION_ERROR"
  | "INVALID_AMOUNT"
  | "CURRENCY_MISMATCH"
  | "INVALID_ORDER_NUMBER"
  | "ALREADY_CANCELLED"
  | "SESSION_EXPIRED"
  | "RATE_LIMIT_EXCEEDED"
  | "INTERNAL_ERROR";

export class DomainError {
  constructor(
    public readonly message: string,
    public readonly code: DomainErrorCode,
    public readonly statusCode: number = 400,
    public readonly details?: any
  ) {}

  // ファクトリーメソッド（原則：エラーの標準化）
  static orderDeadlinePassed(): DomainError {
    return new DomainError(
      "注文締切を過ぎています",
      "ORDER_DEADLINE_PASSED",
      400
    );
  }

  static duplicateOrder(): DomainError {
    return new DomainError("すでに注文済みです", "DUPLICATE_ORDER", 409);
  }

  static orderNotFound(orderNumber?: string): DomainError {
    return new DomainError(
      `注文が見つかりません${orderNumber ? `: ${orderNumber}` : ""}`,
      "ORDER_NOT_FOUND",
      404,
      { orderNumber }
    );
  }

  static menuNotFound(menuId?: string): DomainError {
    return new DomainError("メニューが見つかりません", "MENU_NOT_FOUND", 404, {
      menuId,
    });
  }

  static menuNotAvailable(): DomainError {
    return new DomainError(
      "このメニューは現在利用できません",
      "MENU_NOT_AVAILABLE",
      400
    );
  }

  static unauthorized(message: string = "認証が必要です"): DomainError {
    return new DomainError(message, "UNAUTHORIZED", 401);
  }

  static validation(errors: ValidationError[]): DomainError {
    return new DomainError(
      "入力内容に誤りがあります",
      "VALIDATION_ERROR",
      400,
      errors
    );
  }

  static internal(
    message: string = "システムエラーが発生しました"
  ): DomainError {
    return new DomainError(message, "INTERNAL_ERROR", 500);
  }
}

export interface ValidationError {
  field: string;
  message: string;
  code: string;
}
```
