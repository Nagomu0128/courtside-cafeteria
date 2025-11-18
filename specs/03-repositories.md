# 4. リポジトリインターフェース

## 原則：依存性逆転原則（DIP）とneverthrowのResultAsyncによる非同期エラーハンドリング

```typescript
import { ResultAsync } from "neverthrow";

// domain/repositories/IOrderRepository.ts
export interface IOrderRepository {
  save(order: Order): ResultAsync<Order, DomainError>;
  findById(id: string): ResultAsync<Order | null, DomainError>;
  findByOrderNumber(
    orderNumber: string
  ): ResultAsync<Order | null, DomainError>;
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
  save(
    template: OptionGroupTemplate
  ): ResultAsync<OptionGroupTemplate, DomainError>;
  update(
    template: OptionGroupTemplate
  ): ResultAsync<OptionGroupTemplate, DomainError>;
  delete(id: string): ResultAsync<void, DomainError>;
}

// Supporting Types
export interface PaginationOptions {
  limit: number;
  offset: number;
  orderBy?: string;
  orderDirection?: "ASC" | "DESC";
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
