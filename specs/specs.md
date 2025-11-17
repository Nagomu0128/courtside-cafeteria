# お弁当予約注文システム 詳細仕様書

## 1. アーキテクチャ概要

### 原理原則
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

### 依存パッケージ

```json
{
  "dependencies": {
    "neverthrow": "^6.0.0",
    "@supabase/supabase-js": "^2.39.0",
    "date-fns": "^3.0.0",
    "zod": "^3.22.0",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.2.0",
    "lucide-react": "^0.344.0",
    "@radix-ui/react-slot": "^1.0.2",
    "@radix-ui/react-dialog": "^1.0.5",
    "@radix-ui/react-dropdown-menu": "^2.0.6",
    "@radix-ui/react-select": "^2.0.0",
    "@radix-ui/react-checkbox": "^1.0.4",
    "@radix-ui/react-label": "^2.0.2",
    "@radix-ui/react-toast": "^1.1.5",
    "@radix-ui/react-tabs": "^1.0.4"
  },
  "devDependencies": {
    "prettier": "^3.2.5",
    "prettier-plugin-tailwindcss": "^0.5.11",
    "@ianvs/prettier-plugin-sort-imports": "^4.1.1"
  }
}
```

**UI Component Library:**
- **shadcn/ui**: Tailwind CSSベースの再利用可能なコンポーネントライブラリ
  - Radix UIプリミティブを使用した高品質なアクセシブルコンポーネント
  - class-variance-authority (CVA)によるバリアント管理
  - tailwind-mergeで競合するTailwindクラスを適切にマージ
  - lucide-reactアイコンライブラリ

**Code Formatting:**
- **Prettier**: コードフォーマッター（統一されたコードスタイル）
  - prettier-plugin-tailwindcss: Tailwindクラスの自動ソート
  - @ianvs/prettier-plugin-sort-imports: import文の自動ソート

## 2. エラー型の定義

### 原則：型安全なエラーハンドリング
neverthrowのResult型を使用して、すべてのエラーを型で表現する

```typescript
// domain/errors/DomainError.ts
export type DomainErrorCode = 
  | 'ORDER_DEADLINE_PASSED'
  | 'DUPLICATE_ORDER'
  | 'ORDER_NOT_FOUND'
  | 'MENU_NOT_FOUND'
  | 'MENU_NOT_AVAILABLE'
  | 'UNAUTHORIZED'
  | 'VALIDATION_ERROR'
  | 'INVALID_AMOUNT'
  | 'CURRENCY_MISMATCH'
  | 'INVALID_ORDER_NUMBER'
  | 'ALREADY_CANCELLED'
  | 'SESSION_EXPIRED'
  | 'RATE_LIMIT_EXCEEDED'
  | 'INTERNAL_ERROR';

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
      '注文締切を過ぎています',
      'ORDER_DEADLINE_PASSED',
      400
    );
  }
  
  static duplicateOrder(): DomainError {
    return new DomainError(
      'すでに注文済みです',
      'DUPLICATE_ORDER',
      409
    );
  }
  
  static orderNotFound(orderNumber?: string): DomainError {
    return new DomainError(
      `注文が見つかりません${orderNumber ? `: ${orderNumber}` : ''}`,
      'ORDER_NOT_FOUND',
      404,
      { orderNumber }
    );
  }
  
  static menuNotFound(menuId?: string): DomainError {
    return new DomainError(
      'メニューが見つかりません',
      'MENU_NOT_FOUND',
      404,
      { menuId }
    );
  }
  
  static menuNotAvailable(): DomainError {
    return new DomainError(
      'このメニューは現在利用できません',
      'MENU_NOT_AVAILABLE',
      400
    );
  }
  
  static unauthorized(message: string = '認証が必要です'): DomainError {
    return new DomainError(
      message,
      'UNAUTHORIZED',
      401
    );
  }
  
  static validation(errors: ValidationError[]): DomainError {
    return new DomainError(
      '入力内容に誤りがあります',
      'VALIDATION_ERROR',
      400,
      errors
    );
  }
  
  static internal(message: string = 'システムエラーが発生しました'): DomainError {
    return new DomainError(
      message,
      'INTERNAL_ERROR',
      500
    );
  }
}

export interface ValidationError {
  field: string;
  message: string;
  code: string;
}
```

## 3. ドメインモデル設計

### 3.1 エンティティ定義

```typescript
import { Result, ok, err } from 'neverthrow';

// domain/entities/User.ts
export interface User {
  id: string;  // UUID (匿名認証ID)
  sessionToken: string;  // セッション管理用
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
  price: Money;  // 値オブジェクト
  availableDate: Date;  // 受け渡し日
  orderDeadline: Date;  // 注文締切（前々日17時）
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
  orderNumber: string;  // 変更・キャンセル用の注文番号
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
    cancelledAt: undefined
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
  name: string;  // "ご飯の種類"
  type: OptionType;  // SELECT, RADIO, CHECKBOX
  options: OptionItem[];
  templateId?: string;  // テンプレート参照
}

export interface OptionItem {
  value: string;
  label: string;
  sortOrder: number;
}

// domain/types/Enums.ts
export enum Gender {
  MALE = 'MALE',
  FEMALE = 'FEMALE',
  OTHER = 'OTHER'
}

export enum AgeGroup {
  UNDER_20 = 'UNDER_20',
  TWENTIES = '20-29',
  THIRTIES = '30-39',
  FORTIES = '40-49',
  FIFTIES = '50-59',
  OVER_60 = 'OVER_60'
}

export enum MenuStatus {
  DRAFT = 'DRAFT',
  ACTIVE = 'ACTIVE',
  CLOSED = 'CLOSED',
  CANCELLED = 'CANCELLED'
}

export enum OrderStatus {
  CONFIRMED = 'CONFIRMED',
  MODIFIED = 'MODIFIED',
  CANCELLED = 'CANCELLED'
}

export enum OptionType {
  SELECT = 'SELECT',
  RADIO = 'RADIO',
  CHECKBOX = 'CHECKBOX'
}
```

### 3.2 値オブジェクト

```typescript
import { Result, ok, err } from 'neverthrow';
import { format } from 'date-fns';

// domain/valueObjects/Money.ts
export class Money {
  private constructor(
    private readonly amount: number,
    private readonly currency: string = 'JPY'
  ) {}
  
  // 原則：ファクトリーメソッドでバリデーション
  static create(amount: number, currency: string = 'JPY'): Result<Money, DomainError> {
    if (amount < 0) {
      return err(new DomainError('金額は0以上で設定してください', 'INVALID_AMOUNT'));
    }
    if (!['JPY', 'USD', 'EUR'].includes(currency)) {
      return err(new DomainError('サポートされていない通貨です', 'INVALID_AMOUNT'));
    }
    return ok(new Money(amount, currency));
  }
  
  getValue(): number { return this.amount; }
  getCurrency(): string { return this.currency; }
  
  add(other: Money): Result<Money, DomainError> {
    if (this.currency !== other.currency) {
      return err(new DomainError('異なる通貨は加算できません', 'CURRENCY_MISMATCH'));
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
      return err(new DomainError(
        '注文番号の形式が不正です',
        'INVALID_ORDER_NUMBER'
      ));
    }
    return ok(new OrderNumber(value));
  }
  
  static generate(date: Date, sequence: number): Result<OrderNumber, DomainError> {
    if (sequence < 0 || sequence > 9999) {
      return err(new DomainError(
        'シーケンス番号が範囲外です',
        'INVALID_ORDER_NUMBER'
      ));
    }
    const dateStr = format(date, 'yyyyMMdd');
    const seq = sequence.toString().padStart(4, '0');
    return OrderNumber.create(`${dateStr}-${seq}`);
  }
  
  private static validate(value: string): boolean {
    return /^\d{8}-\d{4}$/.test(value);
  }
  
  getValue(): string { return this.value; }
  toString(): string { return this.value; }
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
  selectedValue: string | string[];  // CHECKBOXの場合は複数選択
}
```

## 4. リポジトリインターフェース

### 原則：依存性逆転原則（DIP）とneverthrowのResultAsyncによる非同期エラーハンドリング

```typescript
import { ResultAsync } from 'neverthrow';

// domain/repositories/IOrderRepository.ts
export interface IOrderRepository {
  save(order: Order): ResultAsync<Order, DomainError>;
  findById(id: string): ResultAsync<Order | null, DomainError>;
  findByOrderNumber(orderNumber: string): ResultAsync<Order | null, DomainError>;
  findByUserIdAndMenuId(
    userId: string, 
    menuId: string
  ): ResultAsync<Order | null, DomainError>;
  findByUserId(
    userId: string, 
    options?: PaginationOptions
  ): ResultAsync<Order[], DomainError>;
  findByMenuId(menuId: string): ResultAsync<Order[], DomainError>;
  update(order: Order): ResultAsync<Order, DomainError>;
  delete(id: string): ResultAsync<void, DomainError>;
  countByMenuIdAndOptions(
    menuId: string, 
    options?: FilterOptions
  ): ResultAsync<OptionCount[], DomainError>;
  getNextSequence(date: Date): ResultAsync<number, DomainError>;
}

// domain/repositories/IMenuRepository.ts
export interface IMenuRepository {
  save(menu: Menu): ResultAsync<Menu, DomainError>;
  findById(id: string): ResultAsync<Menu | null, DomainError>;
  findAvailableMenus(date: Date): ResultAsync<Menu[], DomainError>;
  findByDateRange(
    startDate: Date, 
    endDate: Date
  ): ResultAsync<Menu[], DomainError>;
  update(menu: Menu): ResultAsync<Menu, DomainError>;
  delete(id: string): ResultAsync<void, DomainError>;
}

// domain/repositories/IUserRepository.ts
export interface IUserRepository {
  save(user: User): ResultAsync<User, DomainError>;
  findById(id: string): ResultAsync<User | null, DomainError>;
  findBySessionToken(token: string): ResultAsync<User | null, DomainError>;
  update(user: User): ResultAsync<User, DomainError>;
  updateLastAccessed(id: string): ResultAsync<void, DomainError>;
  saveProfile(
    userId: string, 
    profile: UserProfile
  ): ResultAsync<void, DomainError>;
}

// domain/repositories/IOptionTemplateRepository.ts
export interface IOptionTemplateRepository {
  findAll(): ResultAsync<OptionGroupTemplate[], DomainError>;
  findById(id: string): ResultAsync<OptionGroupTemplate | null, DomainError>;
  save(template: OptionGroupTemplate): ResultAsync<OptionGroupTemplate, DomainError>;
  update(template: OptionGroupTemplate): ResultAsync<OptionGroupTemplate, DomainError>;
  delete(id: string): ResultAsync<void, DomainError>;
}

// Supporting Types
export interface PaginationOptions {
  limit: number;
  offset: number;
  orderBy?: string;
  orderDirection?: 'ASC' | 'DESC';
}

export interface FilterOptions {
  department?: string;
  gender?: Gender;
  ageGroup?: AgeGroup;
  selectedOptions?: { [key: string]: string };
}

export interface OptionCount {
  optionGroupId: string;
  optionValue: string;
  count: number;
  metadata?: {
    department?: string;
    gender?: Gender;
    ageGroup?: AgeGroup;
  };
}

export interface OptionGroupTemplate {
  id: string;
  name: string;
  type: OptionType;
  options: OptionItem[];
  isSystem: boolean;
  createdAt: Date;
}
```

## 5. アプリケーションサービス

### 原則：neverthrowのメソッドチェーンによる合成可能なエラーハンドリング

```typescript
import { Result, ResultAsync, ok, err, combine } from 'neverthrow';
import { generateUUID, generateSecureToken } from '../utils';

// application/services/OrderService.ts
export class OrderService {
  constructor(
    private orderRepository: IOrderRepository,
    private menuRepository: IMenuRepository,
    private userRepository: IUserRepository,
    private eventBus: IEventBus
  ) {}
  
  createOrder(input: CreateOrderInput): ResultAsync<CreateOrderOutput, DomainError> {
    // 入力検証
    const validationResult = OrderValidator.validateCreateOrder(input);
    if (validationResult.isErr()) {
      return ResultAsync.fromSafePromise(Promise.resolve(err(validationResult.error)));
    }
    
    // neverthrowのandThenチェーンを使用したパイプライン処理
    return this.menuRepository.findById(input.menuId)
      .andThen(menu => {
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
      .andThen(menu => 
        // 既存注文の確認
        this.orderRepository.findByUserIdAndMenuId(input.userId, input.menuId)
          .andThen(existingOrder => {
            if (existingOrder && existingOrder.status !== OrderStatus.CANCELLED) {
              return err(DomainError.duplicateOrder());
            }
            return ok(menu);
          })
      )
      .andThen(menu =>
        // 注文番号の生成
        this.orderRepository.getNextSequence(menu.availableDate)
          .andThen(sequence => 
            OrderNumber.generate(menu.availableDate, sequence)
              .map(orderNumber => ({ menu, orderNumber }))
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
          selectedOptions: input.selectedOptions
        });
        
        if (orderResult.isErr()) {
          return err(orderResult.error);
        }
        
        return ok({ order: orderResult.value, menu });
      })
      .andThen(({ order, menu }) =>
        // 注文を保存
        this.orderRepository.save(order)
          .andThen(savedOrder =>
            // プロファイル保存（エラーは無視）
            this.userRepository.saveProfile(input.userId, input.userInfo)
              .map(() => savedOrder)
              .orElse(() => ok(savedOrder))  // プロファイル保存エラーは無視
          )
          .andThen(savedOrder =>
            // イベント発行
            ResultAsync.fromSafePromise(
              this.eventBus.publish(new OrderCreatedEvent(savedOrder))
            ).map(() => ({
              order: savedOrder,
              orderNumber: savedOrder.orderNumber,
              message: '注文が確定しました'
            }))
          )
      );
  }
  
  modifyOrder(input: ModifyOrderInput): ResultAsync<ModifyOrderOutput, DomainError> {
    // 複数の非同期処理を組み合わせる
    const orderResult = this.orderRepository.findByOrderNumber(input.orderNumber);
    const userResult = this.userRepository.findBySessionToken(input.sessionToken);
    
    return ResultAsync.combine([orderResult, userResult])
      .andThen(([order, user]) => {
        if (!order) {
          return err(DomainError.orderNotFound(input.orderNumber));
        }
        if (!user || user.id !== order.userId) {
          return err(DomainError.unauthorized('注文の変更権限がありません'));
        }
        return ok({ order, user });
      })
      .andThen(({ order, user }) =>
        // メニューの取得と締切確認
        this.menuRepository.findById(order.menuId)
          .andThen(menu => {
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
          modifiedAt: new Date()
        };
        
        return this.orderRepository.update(updatedOrder)
          .andThen(savedOrder =>
            // プロファイル更新
            this.userRepository.saveProfile(user.id, input.userInfo)
              .map(() => savedOrder)
              .orElse(() => ok(savedOrder))
          )
          .andThen(savedOrder =>
            // イベント発行
            ResultAsync.fromSafePromise(
              this.eventBus.publish(new OrderModifiedEvent(savedOrder))
            ).map(() => ({
              order: savedOrder,
              message: '注文を変更しました'
            }))
          );
      });
  }
  
  cancelOrder(input: CancelOrderInput): ResultAsync<CancelOrderOutput, DomainError> {
    return this.orderRepository.findByOrderNumber(input.orderNumber)
      .andThen(order => {
        if (!order) {
          return err(DomainError.orderNotFound(input.orderNumber));
        }
        if (order.status === OrderStatus.CANCELLED) {
          return err(new DomainError('すでにキャンセル済みです', 'ALREADY_CANCELLED'));
        }
        return ok(order);
      })
      .andThen(order =>
        // 権限確認
        this.userRepository.findBySessionToken(input.sessionToken)
          .andThen(user => {
            if (!user || user.id !== order.userId) {
              return err(DomainError.unauthorized('キャンセル権限がありません'));
            }
            return ok(order);
          })
      )
      .andThen(order =>
        // 締切確認
        this.menuRepository.findById(order.menuId)
          .andThen(menu => {
            if (!menu) {
              return err(DomainError.menuNotFound());
            }
            if (new Date() > menu.orderDeadline) {
              return err(DomainError.orderDeadlinePassed());
            }
            return ok(order);
          })
      )
      .andThen(order => {
        // キャンセル処理
        const cancelledOrder: Order = {
          ...order,
          status: OrderStatus.CANCELLED,
          cancelledAt: new Date()
        };
        
        return this.orderRepository.update(cancelledOrder)
          .andThen(savedOrder =>
            ResultAsync.fromSafePromise(
              this.eventBus.publish(new OrderCancelledEvent(savedOrder))
            ).map(() => ({
              success: true,
              message: '注文をキャンセルしました'
            }))
          );
      });
  }
  
  getOrderHistory(userId: string): ResultAsync<Order[], DomainError> {
    return this.orderRepository.findByUserId(userId, {
      limit: 10,
      offset: 0,
      orderBy: 'orderedAt',
      orderDirection: 'DESC'
    });
  }
}

// application/services/UserService.ts
export class UserService {
  constructor(
    private userRepository: IUserRepository
  ) {}
  
  anonymousLogin(): ResultAsync<AnonymousLoginOutput, DomainError> {
    const sessionToken = generateSecureToken();
    const user: User = {
      id: generateUUID(),
      sessionToken,
      createdAt: new Date(),
      lastAccessedAt: new Date()
    };
    
    return this.userRepository.save(user)
      .map(savedUser => ({
        userId: savedUser.id,
        sessionToken: savedUser.sessionToken
      }));
  }
  
  getUserProfile(userId: string): ResultAsync<UserProfile | null, DomainError> {
    return this.userRepository.findById(userId)
      .map(user => user?.profile || null);
  }
}
```

## 6. インフラストラクチャ層の実装例

### 原則：Supabaseクライアントのエラーハンドリングをneverthrowでラップ

```typescript
import { ResultAsync, ok, err } from 'neverthrow';
import { createClient } from '@supabase/supabase-js';

// infrastructure/repositories/OrderRepository.ts
export class OrderRepository implements IOrderRepository {
  constructor(private supabase: SupabaseClient) {}
  
  save(order: Order): ResultAsync<Order, DomainError> {
    return ResultAsync.fromPromise(
      this.supabase
        .from('orders')
        .insert({
          id: order.id,
          order_number: order.orderNumber,
          user_id: order.userId,
          menu_id: order.menuId,
          department: order.userInfo.department,
          name: order.userInfo.name,
          gender: order.userInfo.gender,
          age_group: order.userInfo.ageGroup,
          status: order.status,
          ordered_at: order.orderedAt
        })
        .select()
        .single(),
      (error) => this.handleError(error)
    ).andThen(result => {
      if (result.error) {
        return err(this.handleError(result.error));
      }
      
      // 選択されたオプションの保存
      const optionPromises = order.selectedOptions.map(option =>
        this.supabase
          .from('order_options')
          .insert({
            order_id: order.id,
            option_group_id: option.optionGroupId,
            selected_value: Array.isArray(option.selectedValue)
              ? option.selectedValue.join(',')
              : option.selectedValue
          })
      );
      
      return ResultAsync.fromPromise(
        Promise.all(optionPromises),
        (error) => this.handleError(error)
      ).map(() => this.toDomainOrder(result.data));
    });
  }
  
  findByOrderNumber(orderNumber: string): ResultAsync<Order | null, DomainError> {
    return ResultAsync.fromPromise(
      this.supabase
        .from('orders')
        .select(`
          *,
          order_options (*)
        `)
        .eq('order_number', orderNumber)
        .single(),
      (error) => this.handleError(error)
    ).andThen(result => {
      if (result.error) {
        if (result.error.code === 'PGRST116') {
          // データが見つからない場合
          return ok(null);
        }
        return err(this.handleError(result.error));
      }
      return ok(result.data ? this.toDomainOrder(result.data) : null);
    });
  }
  
  findByUserIdAndMenuId(
    userId: string, 
    menuId: string
  ): ResultAsync<Order | null, DomainError> {
    return ResultAsync.fromPromise(
      this.supabase
        .from('orders')
        .select(`
          *,
          order_options (*)
        `)
        .eq('user_id', userId)
        .eq('menu_id', menuId)
        .neq('status', 'CANCELLED')
        .single(),
      (error) => this.handleError(error)
    ).andThen(result => {
      if (result.error) {
        if (result.error.code === 'PGRST116') {
          return ok(null);
        }
        return err(this.handleError(result.error));
      }
      return ok(result.data ? this.toDomainOrder(result.data) : null);
    });
  }
  
  getNextSequence(date: Date): ResultAsync<number, DomainError> {
    const dateStr = format(date, 'yyyy-MM-dd');
    
    return ResultAsync.fromPromise(
      this.supabase.rpc('get_next_order_sequence', { order_date: dateStr }),
      (error) => this.handleError(error)
    ).andThen(result => {
      if (result.error) {
        return err(this.handleError(result.error));
      }
      return ok(result.data as number);
    });
  }
  
  // ヘルパーメソッド
  private handleError(error: any): DomainError {
    console.error('Repository error:', error);
    
    // Supabaseのエラーコードに基づいてDomainErrorに変換
    if (error?.code === '23505') {
      return DomainError.duplicateOrder();
    }
    if (error?.code === '23503') {
      return DomainError.menuNotFound();
    }
    
    return DomainError.internal(error?.message || 'データベースエラーが発生しました');
  }
  
  private toDomainOrder(data: any): Order {
    return {
      id: data.id,
      userId: data.user_id,
      menuId: data.menu_id,
      orderNumber: data.order_number,
      userInfo: {
        department: data.department,
        name: data.name,
        gender: data.gender as Gender,
        ageGroup: data.age_group as AgeGroup
      },
      selectedOptions: data.order_options?.map((opt: any) => ({
        optionGroupId: opt.option_group_id,
        selectedValue: opt.selected_value.includes(',')
          ? opt.selected_value.split(',')
          : opt.selected_value
      })) || [],
      status: data.status as OrderStatus,
      orderedAt: new Date(data.ordered_at),
      modifiedAt: data.modified_at ? new Date(data.modified_at) : undefined,
      cancelledAt: data.cancelled_at ? new Date(data.cancelled_at) : undefined
    };
  }
}
```

## 7. バリデーション仕様（neverthrow版）

### 原則：Zodとneverthrowを組み合わせた型安全なバリデーション

```typescript
import { Result, ok, err } from 'neverthrow';
import { z } from 'zod';

// application/validators/schemas.ts
export const CreateOrderSchema = z.object({
  menuId: z.string().uuid('メニューIDが不正です'),
  department: z.string()
    .min(1, '部署は必須です')
    .max(100, '部署は100文字以内で入力してください'),
  name: z.string()
    .min(1, '名前は必須です')
    .max(100, '名前は100文字以内で入力してください')
    .refine(val => !/<[^>]*>/g.test(val), 'HTMLタグは使用できません'),
  gender: z.nativeEnum(Gender, {
    errorMap: () => ({ message: '性別の選択が不正です' })
  }),
  ageGroup: z.nativeEnum(AgeGroup, {
    errorMap: () => ({ message: '年齢層の選択が不正です' })
  }),
  selectedOptions: z.array(z.object({
    optionGroupId: z.string().uuid(),
    selectedValue: z.union([
      z.string(),
      z.array(z.string())
    ])
  }))
});

// application/validators/OrderValidator.ts
export class OrderValidator {
  static validateCreateOrder(input: CreateOrderInput): Result<void, DomainError> {
    const result = CreateOrderSchema.safeParse(input);
    
    if (!result.success) {
      const errors: ValidationError[] = result.error.errors.map(error => ({
        field: error.path.join('.'),
        message: error.message,
        code: 'VALIDATION_ERROR'
      }));
      
      return err(DomainError.validation(errors));
    }
    
    return ok(undefined);
  }
  
  static validateMenuOption(
    selectedOptions: SelectedOption[],
    menuOptions: MenuOption[]
  ): Result<void, DomainError> {
    const errors: ValidationError[] = [];
    
    // 必須オプションの確認
    const requiredOptions = menuOptions.filter(opt => opt.isRequired);
    for (const reqOpt of requiredOptions) {
      const selected = selectedOptions.find(
        s => s.optionGroupId === reqOpt.optionGroupId
      );
      if (!selected || !selected.selectedValue) {
        errors.push({ 
          field: `option_${reqOpt.optionGroupId}`, 
          message: `${reqOpt.optionGroup.name}は必須選択です`,
          code: 'REQUIRED_OPTION'
        });
      }
    }
    
    // 選択値の妥当性確認
    for (const selected of selectedOptions) {
      const menuOption = menuOptions.find(
        m => m.optionGroupId === selected.optionGroupId
      );
      
      if (!menuOption) {
        errors.push({ 
          field: `option_${selected.optionGroupId}`, 
          message: '無効なオプションが選択されています',
          code: 'INVALID_OPTION'
        });
        continue;
      }
      
      // 選択肢の存在確認
      const validValues = menuOption.optionGroup.options.map(o => o.value);
      const valuesToCheck = Array.isArray(selected.selectedValue) 
        ? selected.selectedValue 
        : [selected.selectedValue];
      
      for (const value of valuesToCheck) {
        if (!validValues.includes(value)) {
          errors.push({ 
            field: `option_${selected.optionGroupId}`, 
            message: `無効な値が選択されています: ${value}`,
            code: 'INVALID_OPTION_VALUE'
          });
        }
      }
    }
    
    if (errors.length > 0) {
      return err(DomainError.validation(errors));
    }
    
    return ok(undefined);
  }
}

// application/validators/MenuValidator.ts
export const CreateMenuSchema = z.object({
  name: z.string()
    .min(1, 'メニュー名は必須です')
    .max(200, 'メニュー名は200文字以内で入力してください'),
  description: z.string().optional(),
  imageUrl: z.string().url('画像URLの形式が不正です').optional(),
  price: z.number()
    .nonnegative('価格は0以上で設定してください')
    .int('価格は整数で入力してください'),
  availableDate: z.string().datetime(),
  orderDeadline: z.string().datetime(),
  maxQuantity: z.number().positive().optional(),
  options: z.array(z.object({
    name: z.string(),
    type: z.nativeEnum(OptionType),
    isRequired: z.boolean(),
    items: z.array(z.object({
      value: z.string(),
      label: z.string()
    }))
  })).optional()
});

export class MenuValidator {
  static validateCreateMenu(input: CreateMenuInput): Result<void, DomainError> {
    const result = CreateMenuSchema.safeParse(input);
    
    if (!result.success) {
      const errors: ValidationError[] = result.error.errors.map(error => ({
        field: error.path.join('.'),
        message: error.message,
        code: 'VALIDATION_ERROR'
      }));
      
      return err(DomainError.validation(errors));
    }
    
    // ビジネスルールの検証
    const availableDate = new Date(input.availableDate);
    const orderDeadline = new Date(input.orderDeadline);
    
    if (orderDeadline >= availableDate) {
      return err(DomainError.validation([{
        field: 'orderDeadline',
        message: '注文締切は受け渡し日より前に設定してください',
        code: 'INVALID_DEADLINE'
      }]));
    }
    
    // 前々日17時の確認
    const expectedDeadline = new Date(availableDate);
    expectedDeadline.setDate(expectedDeadline.getDate() - 2);
    expectedDeadline.setHours(17, 0, 0, 0);
    
    if (Math.abs(orderDeadline.getTime() - expectedDeadline.getTime()) > 60000) {
      return err(DomainError.validation([{
        field: 'orderDeadline',
        message: '注文締切は受け渡し日の前々日17時に設定してください',
        code: 'INVALID_DEADLINE_TIME'
      }]));
    }
    
    return ok(undefined);
  }
}
```

## 8. セキュリティ仕様（neverthrow版）

### 原則：最小権限の原則、深層防御、neverthrowによる安全なエラーハンドリング

```typescript
import { Result, ResultAsync, ok, err } from 'neverthrow';
import * as crypto from 'crypto';

// infrastructure/security/SecurityMiddleware.ts
export class SecurityMiddleware {
  constructor(
    private userRepository: IUserRepository,
    private config: SecurityConfig
  ) {}
  
  // セッショントークン検証
  validateSession(token: string): ResultAsync<User, DomainError> {
    // 原則：トークンの有効期限チェック
    if (!token || token.length < 32) {
      return ResultAsync.fromSafePromise(
        Promise.resolve(err(DomainError.unauthorized('無効なトークンです')))
      );
    }
    
    return this.userRepository.findBySessionToken(token)
      .andThen(user => {
        if (!user) {
          return err(DomainError.unauthorized('セッションが見つかりません'));
        }
        
        const lastAccess = new Date(user.lastAccessedAt);
        const now = new Date();
        const hoursSinceLastAccess = 
          (now.getTime() - lastAccess.getTime()) / (1000 * 60 * 60);
        
        if (hoursSinceLastAccess > this.config.sessionTimeoutHours) {
          return err(new DomainError('セッションの有効期限が切れました', 'SESSION_EXPIRED'));
        }
        
        return ok(user);
      })
      .andThen(user =>
        // アクセス時刻更新（エラーは無視）
        this.userRepository.updateLastAccessed(user.id)
          .map(() => user)
          .orElse(() => ok(user))
      );
  }
  
  // レート制限
  checkRateLimit(
    userId: string, 
    action: string
  ): ResultAsync<void, DomainError> {
    const key = `rate_limit:${userId}:${action}`;
    const limit = this.config.rateLimits[action] || 100;
    const window = 3600; // 1時間
    
    return this.getActionCount(key, window)
      .andThen(count => {
        if (count >= limit) {
          return err(new DomainError(
            `リクエストが多すぎます。しばらくお待ちください`,
            'RATE_LIMIT_EXCEEDED',
            429
          ));
        }
        return ok(undefined);
      })
      .andThen(() => this.incrementActionCount(key, window));
  }
  
  // CSRFトークン生成
  generateCSRFToken(): Result<string, DomainError> {
    try {
      const token = crypto.randomBytes(32).toString('hex');
      return ok(token);
    } catch (error) {
      return err(DomainError.internal('CSRFトークンの生成に失敗しました'));
    }
  }
  
  // CSRFトークン検証
  validateCSRFToken(
    token: string, 
    sessionToken: string
  ): Result<void, DomainError> {
    if (!token || !sessionToken) {
      return err(DomainError.unauthorized('CSRFトークンが無効です'));
    }
    
    // トークンペアの検証ロジック
    const isValid = this.verifyTokenPair(token, sessionToken);
    if (!isValid) {
      return err(DomainError.unauthorized('CSRFトークンが一致しません'));
    }
    
    return ok(undefined);
  }
  
  private getActionCount(
    key: string, 
    window: number
  ): ResultAsync<number, DomainError> {
    // Redis実装例（簡略化）
    return ResultAsync.fromSafePromise(Promise.resolve(ok(0)));
  }
  
  private incrementActionCount(
    key: string, 
    window: number
  ): ResultAsync<void, DomainError> {
    // Redis実装例（簡略化）
    return ResultAsync.fromSafePromise(Promise.resolve(ok(undefined)));
  }
  
  private verifyTokenPair(csrfToken: string, sessionToken: string): boolean {
    // トークンペアの検証ロジック
    return true;
  }
}

// infrastructure/security/InputSanitizer.ts
export class InputSanitizer {
  // XSS対策
  static sanitizeHtml(input: string): Result<string, DomainError> {
    try {
      const sanitized = input
        .replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;')
        .replace(/"/g, '&quot;')
        .replace(/'/g, '&#x27;')
        .replace(/\//g, '&#x2F;');
      return ok(sanitized);
    } catch (error) {
      return err(DomainError.internal('入力のサニタイズに失敗しました'));
    }
  }
  
  // SQLインジェクション対策
  static validateSqlInput(input: string): Result<string, DomainError> {
    if (/[';--]/.test(input)) {
      return err(DomainError.validation([{
        field: 'input',
        message: '不正な文字が含まれています',
        code: 'INVALID_CHARACTERS'
      }]));
    }
    return ok(input);
  }
}

// infrastructure/security/EncryptionService.ts
export class EncryptionService {
  private algorithm = 'aes-256-gcm';
  private key: Buffer;
  
  constructor() {
    const keyString = process.env.ENCRYPTION_KEY;
    if (!keyString) {
      throw new Error('暗号化キーが設定されていません');
    }
    this.key = Buffer.from(keyString, 'hex');
  }
  
  // 個人情報の暗号化
  encrypt(text: string): Result<EncryptedData, DomainError> {
    try {
      const iv = crypto.randomBytes(16);
      const cipher = crypto.createCipheriv(this.algorithm, this.key, iv);
      
      let encrypted = cipher.update(text, 'utf8', 'hex');
      encrypted += cipher.final('hex');
      
      const authTag = cipher.getAuthTag();
      
      return ok({
        encrypted,
        iv: iv.toString('hex'),
        authTag: authTag.toString('hex')
      });
    } catch (error) {
      return err(DomainError.internal('暗号化に失敗しました'));
    }
  }
  
  // 復号化
  decrypt(data: EncryptedData): Result<string, DomainError> {
    try {
      const decipher = crypto.createDecipheriv(
        this.algorithm,
        this.key,
        Buffer.from(data.iv, 'hex')
      );
      
      decipher.setAuthTag(Buffer.from(data.authTag, 'hex'));
      
      let decrypted = decipher.update(data.encrypted, 'hex', 'utf8');
      decrypted += decipher.final('utf8');
      
      return ok(decrypted);
    } catch (error) {
      return err(DomainError.internal('復号化に失敗しました'));
    }
  }
}

// Types
export interface SecurityConfig {
  sessionTimeoutHours: number;
  rateLimits: { [action: string]: number };
  encryptionKey?: string;
}

export interface EncryptedData {
  encrypted: string;
  iv: string;
  authTag: string;
}
```

## 9. プレゼンテーション層（コントローラー）

### 原則：neverthrowのmatchによるパターンマッチング

```typescript
import { Request, Response } from 'express';
import { ResultAsync } from 'neverthrow';

// presentation/controllers/OrderController.ts
export class OrderController {
  constructor(
    private orderService: OrderService,
    private securityMiddleware: SecurityMiddleware
  ) {}
  
  async createOrder(req: Request, res: Response): Promise<void> {
    const sessionToken = req.headers['x-session-token'] as string;
    
    // セッション検証とレート制限を組み合わせる
    await this.securityMiddleware.validateSession(sessionToken)
      .andThen(user =>
        this.securityMiddleware.checkRateLimit(user.id, 'create_order')
          .map(() => user)
      )
      .andThen(user =>
        // 注文作成
        this.orderService.createOrder({
          userId: user.id,
          menuId: req.body.menuId,
          userInfo: {
            department: req.body.department,
            name: req.body.name,
            gender: req.body.gender,
            ageGroup: req.body.ageGroup
          },
          selectedOptions: req.body.options
        })
      )
      .match(
        // 成功時
        (result) => {
          res.status(201).json({
            success: true,
            data: {
              orderNumber: result.orderNumber,
              message: result.message
            }
          });
        },
        // エラー時
        (error) => {
          res.status(error.statusCode).json({
            success: false,
            error: {
              code: error.code,
              message: error.message,
              details: error.details
            }
          });
        }
      );
  }
  
  async modifyOrder(req: Request, res: Response): Promise<void> {
    const sessionToken = req.headers['x-session-token'] as string;
    
    await this.orderService.modifyOrder({
      orderNumber: req.params.orderNumber,
      sessionToken,
      userInfo: {
        department: req.body.department,
        name: req.body.name,
        gender: req.body.gender,
        ageGroup: req.body.ageGroup
      },
      selectedOptions: req.body.options
    })
    .match(
      (result) => {
        res.json({
          success: true,
          data: {
            message: result.message
          }
        });
      },
      (error) => {
        res.status(error.statusCode).json({
          success: false,
          error: {
            code: error.code,
            message: error.message,
            details: error.details
          }
        });
      }
    );
  }
  
  async cancelOrder(req: Request, res: Response): Promise<void> {
    const sessionToken = req.headers['x-session-token'] as string;
    
    await this.orderService.cancelOrder({
      orderNumber: req.params.orderNumber,
      sessionToken
    })
    .match(
      (result) => {
        res.json({
          success: true,
          data: {
            message: result.message
          }
        });
      },
      (error) => {
        res.status(error.statusCode).json({
          success: false,
          error: {
            code: error.code,
            message: error.message
          }
        });
      }
    );
  }
  
  async getOrderHistory(req: Request, res: Response): Promise<void> {
    const sessionToken = req.headers['x-session-token'] as string;
    
    await this.securityMiddleware.validateSession(sessionToken)
      .andThen(user => this.orderService.getOrderHistory(user.id))
      .match(
        (orders) => {
          res.json({
            success: true,
            data: orders
          });
        },
        (error) => {
          res.status(error.statusCode).json({
            success: false,
            error: {
              code: error.code,
              message: error.message
            }
          });
        }
      );
  }
}
```

## 10. 管理者API仕様（neverthrow版）

```typescript
import { ResultAsync, ok, err, combine } from 'neverthrow';

// application/admin/AdminOrderService.ts
export class AdminOrderService {
  constructor(
    private orderRepository: IOrderRepository,
    private menuRepository: IMenuRepository,
    private exportService: IExportService
  ) {}
  
  // 注文集計
  getOrderSummary(
    input: OrderSummaryInput
  ): ResultAsync<OrderSummaryOutput, DomainError> {
    return this.menuRepository.findById(input.menuId)
      .andThen(menu => {
        if (!menu) {
          return err(DomainError.menuNotFound());
        }
        return ok(menu);
      })
      .andThen(menu =>
        this.orderRepository.countByMenuIdAndOptions(
          input.menuId,
          input.filters
        ).map(counts => ({ menu, counts }))
      )
      .map(({ menu, counts }) => {
        // 集計処理
        const totalOrders = counts.reduce((sum, c) => sum + c.count, 0);
        
        const byDepartment = this.groupBy(counts, 'department');
        const byGender = this.groupBy(counts, 'gender');
        const byAgeGroup = this.groupBy(counts, 'ageGroup');
        const byOptions = this.groupByOptions(counts);
        
        return {
          menuId: menu.id,
          menuName: menu.name,
          availableDate: menu.availableDate,
          totalOrders,
          summaries: {
            byDepartment,
            byGender,
            byAgeGroup,
            byOptions
          },
          generatedAt: new Date()
        };
      });
  }
  
  // CSVエクスポート
  exportOrders(
    input: ExportOrdersInput
  ): ResultAsync<ExportResult, DomainError> {
    return ResultAsync.combine([
      this.menuRepository.findById(input.menuId),
      this.orderRepository.findByMenuId(input.menuId)
    ])
    .andThen(([menu, orders]) => {
      if (!menu) {
        return err(DomainError.menuNotFound());
      }
      
      const data = orders.map(order => ({
        注文番号: order.orderNumber,
        注文日時: format(order.orderedAt, 'yyyy-MM-dd HH:mm:ss'),
        部署: order.userInfo.department,
        名前: order.userInfo.name,
        性別: this.translateGender(order.userInfo.gender),
        年齢層: this.translateAgeGroup(order.userInfo.ageGroup),
        ...this.flattenOptions(order.selectedOptions),
        ステータス: this.translateStatus(order.status),
        変更日時: order.modifiedAt 
          ? format(order.modifiedAt, 'yyyy-MM-dd HH:mm:ss') 
          : '',
        キャンセル日時: order.cancelledAt 
          ? format(order.cancelledAt, 'yyyy-MM-dd HH:mm:ss') 
          : ''
      }));
      
      const exportPromise = input.format === 'CSV'
        ? this.exportService.exportToCSV(data)
        : this.exportService.exportToExcel(data, {
            sheetName: `注文一覧_${format(menu.availableDate, 'yyyyMMdd')}`,
            autoFilter: true,
            freezePane: { row: 1, column: 3 }
          });
      
      return ResultAsync.fromPromise(
        exportPromise,
        (error) => DomainError.internal('エクスポートに失敗しました')
      ).map(buffer => ({
        fileName: `orders_${input.menuId}_${format(new Date(), 'yyyyMMddHHmmss')}.${input.format.toLowerCase()}`,
        mimeType: input.format === 'CSV' 
          ? 'text/csv' 
          : 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
        buffer
      }));
    });
  }
  
  // メニュー管理
  createMenu(input: CreateMenuInput): ResultAsync<Menu, DomainError> {
    // バリデーション
    const validationResult = MenuValidator.validateCreateMenu(input);
    if (validationResult.isErr()) {
      return ResultAsync.fromSafePromise(
        Promise.resolve(err(validationResult.error))
      );
    }
    
    // 価格の値オブジェクト作成
    const priceResult = Money.create(input.price);
    if (priceResult.isErr()) {
      return ResultAsync.fromSafePromise(
        Promise.resolve(err(priceResult.error))
      );
    }
    
    const menu: Menu = {
      id: generateUUID(),
      name: input.name,
      description: input.description || '',
      imageUrl: input.imageUrl || '',
      price: priceResult.value,
      availableDate: new Date(input.availableDate),
      orderDeadline: new Date(input.orderDeadline),
      maxQuantity: input.maxQuantity,
      status: MenuStatus.DRAFT,
      options: [],
      createdAt: new Date(),
      updatedAt: new Date()
    };
    
    return this.menuRepository.save(menu)
      .andThen(savedMenu => {
        if (input.options && input.options.length > 0) {
          return this.setupMenuOptions(savedMenu.id, input.options)
            .map(() => savedMenu);
        }
        return ok(savedMenu);
      });
  }
  
  updateMenuStatus(
    menuId: string,
    status: MenuStatus
  ): ResultAsync<Menu, DomainError> {
    return this.menuRepository.findById(menuId)
      .andThen(menu => {
        if (!menu) {
          return err(DomainError.menuNotFound());
        }
        
        // ステータス変更のビジネスルール
        if (menu.status === MenuStatus.CANCELLED) {
          return err(new DomainError(
            'キャンセル済みのメニューは変更できません',
            'INVALID_STATUS_TRANSITION'
          ));
        }
        
        if (status === MenuStatus.ACTIVE && new Date() > menu.orderDeadline) {
          return err(new DomainError(
            '締切を過ぎたメニューは公開できません',
            'INVALID_STATUS_TRANSITION'
          ));
        }
        
        menu.status = status;
        menu.updatedAt = new Date();
        
        return ok(menu);
      })
      .andThen(menu => this.menuRepository.update(menu));
  }
  
  // ヘルパーメソッド
  private groupBy(data: any[], key: string): GroupedSummary[] {
    const grouped = data.reduce((acc, item) => {
      const value = item.metadata?.[key];
      if (value) {
        if (!acc[value]) acc[value] = 0;
        acc[value] += item.count;
      }
      return acc;
    }, {});
    
    const total = Object.values(grouped).reduce((sum: number, count: any) => sum + count, 0);
    
    return Object.entries(grouped).map(([key, count]) => ({
      key,
      count: count as number,
      percentage: total > 0 ? ((count as number) / total) * 100 : 0
    }));
  }
  
  private groupByOptions(data: OptionCount[]): OptionGroupSummary[] {
    const grouped = new Map<string, Map<string, number>>();
    
    for (const item of data) {
      if (!grouped.has(item.optionGroupId)) {
        grouped.set(item.optionGroupId, new Map());
      }
      const optionMap = grouped.get(item.optionGroupId)!;
      optionMap.set(item.optionValue, item.count);
    }
    
    return Array.from(grouped.entries()).map(([groupId, values]) => ({
      optionGroupId: groupId,
      values: Array.from(values.entries()).map(([value, count]) => ({
        value,
        count
      }))
    }));
  }
  
  private flattenOptions(options: SelectedOption[]): any {
    const flattened: any = {};
    for (const option of options) {
      const value = Array.isArray(option.selectedValue)
        ? option.selectedValue.join(', ')
        : option.selectedValue;
      flattened[`オプション_${option.optionGroupId}`] = value;
    }
    return flattened;
  }
  
  private translateGender(gender: Gender): string {
    const translations = {
      [Gender.MALE]: '男性',
      [Gender.FEMALE]: '女性',
      [Gender.OTHER]: 'その他'
    };
    return translations[gender] || gender;
  }
  
  private translateAgeGroup(ageGroup: AgeGroup): string {
    const translations = {
      [AgeGroup.UNDER_20]: '20歳未満',
      [AgeGroup.TWENTIES]: '20代',
      [AgeGroup.THIRTIES]: '30代',
      [AgeGroup.FORTIES]: '40代',
      [AgeGroup.FIFTIES]: '50代',
      [AgeGroup.OVER_60]: '60歳以上'
    };
    return translations[ageGroup] || ageGroup;
  }
  
  private translateStatus(status: OrderStatus): string {
    const translations = {
      [OrderStatus.CONFIRMED]: '確定',
      [OrderStatus.MODIFIED]: '変更済',
      [OrderStatus.CANCELLED]: 'キャンセル'
    };
    return translations[status] || status;
  }
  
  private setupMenuOptions(
    menuId: string, 
    options: any[]
  ): ResultAsync<void, DomainError> {
    // オプション設定の実装
    return ResultAsync.fromSafePromise(Promise.resolve(ok(undefined)));
  }
}
```

## 11. データベーススキーマ

### 原則：正規化とパフォーマンスのバランス

```sql
-- ユーザー（匿名認証）
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_token VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
  last_accessed_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- ユーザープロファイル（原則：個人情報の分離）
CREATE TABLE user_profiles (
  user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  department VARCHAR(100),
  name VARCHAR(100),
  gender VARCHAR(20),
  age_group VARCHAR(20),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- メニュー
CREATE TABLE menus (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(200) NOT NULL,
  description TEXT,
  image_url VARCHAR(500),
  price DECIMAL(10, 2) NOT NULL,
  available_date DATE NOT NULL,
  order_deadline TIMESTAMP WITH TIME ZONE NOT NULL,
  max_quantity INTEGER,
  status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
  
  CONSTRAINT check_deadline_before_available CHECK (order_deadline < available_date),
  CONSTRAINT check_price_positive CHECK (price >= 0),
  CONSTRAINT check_max_quantity_positive CHECK (max_quantity IS NULL OR max_quantity > 0)
);

-- インデックス
CREATE INDEX idx_menus_available_date ON menus(available_date);
CREATE INDEX idx_menus_status ON menus(status);
CREATE INDEX idx_menus_order_deadline ON menus(order_deadline);

-- 注文
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_number VARCHAR(20) UNIQUE NOT NULL,
  user_id UUID NOT NULL REFERENCES users(id),
  menu_id UUID NOT NULL REFERENCES menus(id),
  department VARCHAR(100) NOT NULL,
  name VARCHAR(100) NOT NULL,
  gender VARCHAR(20) NOT NULL,
  age_group VARCHAR(20) NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'CONFIRMED',
  ordered_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
  modified_at TIMESTAMP WITH TIME ZONE,
  cancelled_at TIMESTAMP WITH TIME ZONE,
  
  CONSTRAINT check_status CHECK (status IN ('CONFIRMED', 'MODIFIED', 'CANCELLED')),
  CONSTRAINT check_gender CHECK (gender IN ('MALE', 'FEMALE', 'OTHER')),
  CONSTRAINT unique_active_order EXCLUDE USING btree (user_id WITH =, menu_id WITH =) 
    WHERE (status != 'CANCELLED')
);

-- インデックス
CREATE INDEX idx_orders_order_number ON orders(order_number);
CREATE INDEX idx_orders_user_menu ON orders(user_id, menu_id);
CREATE INDEX idx_orders_menu_status ON orders(menu_id, status);
CREATE INDEX idx_orders_ordered_at ON orders(ordered_at DESC);

-- オプショングループテンプレート
CREATE TABLE option_group_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL UNIQUE,
  type VARCHAR(20) NOT NULL,
  is_system BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
  
  CONSTRAINT check_type CHECK (type IN ('SELECT', 'RADIO', 'CHECKBOX'))
);

-- オプションアイテムテンプレート
CREATE TABLE option_item_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  group_template_id UUID NOT NULL REFERENCES option_group_templates(id) ON DELETE CASCADE,
  value VARCHAR(100) NOT NULL,
  label VARCHAR(100) NOT NULL,
  sort_order INTEGER NOT NULL DEFAULT 0,
  
  CONSTRAINT unique_template_value UNIQUE (group_template_id, value)
);

-- メニューオプション設定
CREATE TABLE menu_option_groups (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  menu_id UUID NOT NULL REFERENCES menus(id) ON DELETE CASCADE,
  name VARCHAR(100) NOT NULL,
  type VARCHAR(20) NOT NULL,
  is_required BOOLEAN DEFAULT FALSE,
  template_id UUID REFERENCES option_group_templates(id),
  sort_order INTEGER NOT NULL DEFAULT 0,
  
  CONSTRAINT check_type CHECK (type IN ('SELECT', 'RADIO', 'CHECKBOX'))
);

-- メニューオプションアイテム
CREATE TABLE menu_option_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  option_group_id UUID NOT NULL REFERENCES menu_option_groups(id) ON DELETE CASCADE,
  value VARCHAR(100) NOT NULL,
  label VARCHAR(100) NOT NULL,
  sort_order INTEGER NOT NULL DEFAULT 0,
  
  CONSTRAINT unique_option_value UNIQUE (option_group_id, value)
);

-- 選択された注文オプション
CREATE TABLE order_options (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  option_group_id UUID NOT NULL REFERENCES menu_option_groups(id),
  selected_value VARCHAR(255) NOT NULL, -- 複数選択の場合はカンマ区切り
  
  CONSTRAINT unique_order_option UNIQUE (order_id, option_group_id)
);

-- インデックス
CREATE INDEX idx_order_options_order ON order_options(order_id);

-- 注文番号シーケンス管理（ストアドファンクション）
CREATE OR REPLACE FUNCTION get_next_order_sequence(order_date DATE)
RETURNS INTEGER AS $$
DECLARE
  next_seq INTEGER;
BEGIN
  INSERT INTO order_sequences (date, last_sequence)
  VALUES (order_date, 1)
  ON CONFLICT (date) DO UPDATE
  SET last_sequence = order_sequences.last_sequence + 1
  RETURNING last_sequence INTO next_seq;
  
  RETURN next_seq;
END;
$$ LANGUAGE plpgsql;

CREATE TABLE order_sequences (
  date DATE PRIMARY KEY,
  last_sequence INTEGER NOT NULL DEFAULT 0
);

-- RLS (Row Level Security) ポリシー
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE order_options ENABLE ROW LEVEL SECURITY;

-- ユーザーは自分のデータのみアクセス可能
CREATE POLICY users_own_orders ON orders
  FOR ALL
  USING (auth.uid()::TEXT = user_id::TEXT);

CREATE POLICY users_own_profiles ON user_profiles
  FOR ALL
  USING (auth.uid()::TEXT = user_id::TEXT);

CREATE POLICY users_own_order_options ON order_options
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM orders 
      WHERE orders.id = order_options.order_id 
      AND orders.user_id::TEXT = auth.uid()::TEXT
    )
  );

-- 管理者は全データアクセス可能
CREATE POLICY admin_all_access_orders ON orders
  FOR ALL
  USING (
    EXISTS (
      SELECT 1 FROM auth.users 
      WHERE auth.uid() = id 
      AND raw_user_meta_data->>'role' = 'admin'
    )
  );
```

## 12. テスト仕様（neverthrow版）

### 原則：Result型を活用した予測可能なテスト

```typescript
import { describe, it, expect, beforeEach, jest } from '@jest/globals';
import { ok, err } from 'neverthrow';

// tests/services/OrderService.test.ts
describe('OrderService', () => {
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
      publish: jest.fn()
    };
    
    orderService = new OrderService(
      mockOrderRepository,
      mockMenuRepository,
      mockUserRepository,
      mockEventBus
    );
  });
  
  describe('createOrder', () => {
    it('正常に注文を作成できる', async () => {
      // Arrange
      const input: CreateOrderInput = {
        userId: 'user-123',
        menuId: 'menu-456',
        userInfo: {
          department: '営業部',
          name: '山田太郎',
          gender: Gender.MALE,
          ageGroup: AgeGroup.THIRTIES
        },
        selectedOptions: []
      };
      
      const mockMenu: Menu = {
        id: 'menu-456',
        name: '日替わり弁当',
        price: Money.create(500).value,
        availableDate: new Date('2024-01-20'),
        orderDeadline: new Date('2024-01-18T17:00:00'),
        status: MenuStatus.ACTIVE,
        // ... その他のフィールド
      };
      
      mockMenuRepository.findById.mockResolvedValue(ok(mockMenu));
      mockOrderRepository.findByUserIdAndMenuId.mockResolvedValue(ok(null));
      mockOrderRepository.getNextSequence.mockResolvedValue(ok(1));
      mockOrderRepository.save.mockResolvedValue(ok({
        id: 'order-789',
        orderNumber: '20240120-0001',
        // ... その他のフィールド
      } as Order));
      mockUserRepository.saveProfile.mockResolvedValue(ok(undefined));
      
      // Act
      const result = await orderService.createOrder(input);
      
      // Assert
      expect(result.isOk()).toBe(true);
      if (result.isOk()) {
        expect(result.value.orderNumber).toBe('20240120-0001');
        expect(result.value.message).toBe('注文が確定しました');
      }
      expect(mockEventBus.publish).toHaveBeenCalled();
    });
    
    it('締切を過ぎている場合はエラーを返す', async () => {
      // Arrange
      const input: CreateOrderInput = {
        // ... 入力データ
      };
      
      const mockMenu: Menu = {
        // ... メニューデータ
        orderDeadline: new Date('2020-01-01'), // 過去の日付
      };
      
      mockMenuRepository.findById.mockResolvedValue(ok(mockMenu));
      
      // Act
      const result = await orderService.createOrder(input);
      
      // Assert
      expect(result.isErr()).toBe(true);
      if (result.isErr()) {
        expect(result.error.code).toBe('ORDER_DEADLINE_PASSED');
      }
    });
    
    it('既に注文済みの場合はエラーを返す', async () => {
      // Arrange
      const existingOrder: Order = {
        // ... 既存注文データ
        status: OrderStatus.CONFIRMED
      };
      
      mockMenuRepository.findById.mockResolvedValue(ok(mockMenu));
      mockOrderRepository.findByUserIdAndMenuId.mockResolvedValue(ok(existingOrder));
      
      // Act
      const result = await orderService.createOrder(input);
      
      // Assert
      expect(result.isErr()).toBe(true);
      if (result.isErr()) {
        expect(result.error.code).toBe('DUPLICATE_ORDER');
      }
    });
  });
});
```

## 13. 拡張性の考慮

### 原則：オープン・クローズド原則（OCP）とneverthrowによる合成可能な設計

```typescript
import { Result, ResultAsync, ok, err } from 'neverthrow';

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
    return this.menuRepository.findById(menuId)
      .andThen(menu => {
        if (!menu) return err(DomainError.menuNotFound());
        return ok(menu);
      })
      .andThen(menu =>
        // 在庫情報を取得
        this.inventoryService.checkAvailability(menu.options)
          .andThen(availableOptions =>
            // AIによる推奨オプション
            this.aiService.getRecommendations(menuId, availableOptions)
              .map(recommendations =>
                this.mergeOptions(availableOptions, recommendations)
              )
          )
      );
  }
  
  validateOptions(options: SelectedOption[]): Result<void, DomainError> {
    // リアルタイムの在庫確認
    return this.inventoryService.validateStock(options)
      .match(
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
    available.forEach(opt => merged.set(opt.id, opt));
    recommended.forEach(opt => merged.set(opt.id, { ...opt, isRecommended: true }));
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
    return this.userPreferenceService.getPreferences(notification.recipient)
      .andThen(preferences => {
        const notifications: ResultAsync<void, DomainError>[] = [];
        
        if (preferences.email) {
          notifications.push(
            this.emailService.send({
              to: preferences.email,
              subject: notification.subject,
              body: notification.content
            })
          );
        }
        
        if (preferences.phone && this.shouldSendSMS(notification.type)) {
          notifications.push(
            this.smsService.send({
              to: preferences.phone,
              message: this.formatForSMS(notification.content)
            })
          );
        }
        
        if (preferences.pushToken) {
          notifications.push(
            this.pushService.send({
              token: preferences.pushToken,
              title: notification.subject,
              body: notification.content
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
      NotificationType.ORDER_CANCELLED
    ].includes(type);
  }
  
  private formatForSMS(content: string): string {
    return content.length > 140 
      ? content.substring(0, 137) + '...' 
      : content;
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