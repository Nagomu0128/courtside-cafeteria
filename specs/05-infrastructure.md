# 6. インフラストラクチャ層の実装例

## 原則：Supabaseクライアントのエラーハンドリングをneverthrowでラップ

```typescript
import { ResultAsync, ok, err } from "neverthrow";
import { createClient } from "@supabase/supabase-js";

// infrastructure/repositories/OrderRepository.ts
export class OrderRepository implements IOrderRepository {
  constructor(private supabase: SupabaseClient) {}

  save(order: Order): ResultAsync<Order, DomainError> {
    return ResultAsync.fromPromise(
      this.supabase
        .from("orders")
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
          ordered_at: order.orderedAt,
        })
        .select()
        .single(),
      (error) => this.handleError(error)
    ).andThen((result) => {
      if (result.error) {
        return err(this.handleError(result.error));
      }

      // 選択されたオプションの保存
      const optionPromises = order.selectedOptions.map((option) =>
        this.supabase.from("order_options").insert({
          order_id: order.id,
          option_group_id: option.optionGroupId,
          selected_value: Array.isArray(option.selectedValue)
            ? option.selectedValue.join(",")
            : option.selectedValue,
        })
      );

      return ResultAsync.fromPromise(Promise.all(optionPromises), (error) =>
        this.handleError(error)
      ).map(() => this.toDomainOrder(result.data));
    });
  }

  findByOrderNumber(
    orderNumber: string
  ): ResultAsync<Order | null, DomainError> {
    return ResultAsync.fromPromise(
      this.supabase
        .from("orders")
        .select(
          `
          *,
          order_options (*)
        `
        )
        .eq("order_number", orderNumber)
        .single(),
      (error) => this.handleError(error)
    ).andThen((result) => {
      if (result.error) {
        if (result.error.code === "PGRST116") {
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
        .from("orders")
        .select(
          `
          *,
          order_options (*)
        `
        )
        .eq("user_id", userId)
        .eq("menu_id", menuId)
        .neq("status", "CANCELLED")
        .single(),
      (error) => this.handleError(error)
    ).andThen((result) => {
      if (result.error) {
        if (result.error.code === "PGRST116") {
          return ok(null);
        }
        return err(this.handleError(result.error));
      }
      return ok(result.data ? this.toDomainOrder(result.data) : null);
    });
  }

  getNextSequence(date: Date): ResultAsync<number, DomainError> {
    const dateStr = format(date, "yyyy-MM-dd");

    return ResultAsync.fromPromise(
      this.supabase.rpc("get_next_order_sequence", { order_date: dateStr }),
      (error) => this.handleError(error)
    ).andThen((result) => {
      if (result.error) {
        return err(this.handleError(result.error));
      }
      return ok(result.data as number);
    });
  }

  // ヘルパーメソッド
  private handleError(error: any): DomainError {
    console.error("Repository error:", error);

    // Supabaseのエラーコードに基づいてDomainErrorに変換
    if (error?.code === "23505") {
      return DomainError.duplicateOrder();
    }
    if (error?.code === "23503") {
      return DomainError.menuNotFound();
    }

    return DomainError.internal(
      error?.message || "データベースエラーが発生しました"
    );
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
        ageGroup: data.age_group as AgeGroup,
      },
      selectedOptions:
        data.order_options?.map((opt: any) => ({
          optionGroupId: opt.option_group_id,
          selectedValue: opt.selected_value.includes(",")
            ? opt.selected_value.split(",")
            : opt.selected_value,
        })) || [],
      status: data.status as OrderStatus,
      orderedAt: new Date(data.ordered_at),
      modifiedAt: data.modified_at ? new Date(data.modified_at) : undefined,
      cancelledAt: data.cancelled_at ? new Date(data.cancelled_at) : undefined,
    };
  }
}
```
