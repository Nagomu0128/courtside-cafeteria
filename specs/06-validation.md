# 7. バリデーション仕様（neverthrow版）

## 原則：Zodとneverthrowを組み合わせた型安全なバリデーション

```typescript
import { Result, ok, err } from "neverthrow";
import { z } from "zod";

// application/validators/schemas.ts
export const CreateOrderSchema = z.object({
  menuId: z.string().uuid("メニューIDが不正です"),
  department: z
    .string()
    .min(1, "部署は必須です")
    .max(100, "部署は100文字以内で入力してください"),
  name: z
    .string()
    .min(1, "名前は必須です")
    .max(100, "名前は100文字以内で入力してください")
    .refine((val) => !/<[^>]*>/g.test(val), "HTMLタグは使用できません"),
  gender: z.nativeEnum(Gender, {
    errorMap: () => ({ message: "性別の選択が不正です" }),
  }),
  ageGroup: z.nativeEnum(AgeGroup, {
    errorMap: () => ({ message: "年齢層の選択が不正です" }),
  }),
  selectedOptions: z.array(
    z.object({
      optionGroupId: z.string().uuid(),
      selectedValue: z.union([z.string(), z.array(z.string())]),
    })
  ),
});

// application/validators/OrderValidator.ts
export class OrderValidator {
  static validateCreateOrder(
    input: CreateOrderInput
  ): Result<void, DomainError> {
    const result = CreateOrderSchema.safeParse(input);

    if (!result.success) {
      const errors: ValidationError[] = result.error.errors.map((error) => ({
        field: error.path.join("."),
        message: error.message,
        code: "VALIDATION_ERROR",
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
    const requiredOptions = menuOptions.filter((opt) => opt.isRequired);
    for (const reqOpt of requiredOptions) {
      const selected = selectedOptions.find(
        (s) => s.optionGroupId === reqOpt.optionGroupId
      );
      if (!selected || !selected.selectedValue) {
        errors.push({
          field: `option_${reqOpt.optionGroupId}`,
          message: `${reqOpt.optionGroup.name}は必須選択です`,
          code: "REQUIRED_OPTION",
        });
      }
    }

    // 選択値の妥当性確認
    for (const selected of selectedOptions) {
      const menuOption = menuOptions.find(
        (m) => m.optionGroupId === selected.optionGroupId
      );

      if (!menuOption) {
        errors.push({
          field: `option_${selected.optionGroupId}`,
          message: "無効なオプションが選択されています",
          code: "INVALID_OPTION",
        });
        continue;
      }

      // 選択肢の存在確認
      const validValues = menuOption.optionGroup.options.map((o) => o.value);
      const valuesToCheck = Array.isArray(selected.selectedValue)
        ? selected.selectedValue
        : [selected.selectedValue];

      for (const value of valuesToCheck) {
        if (!validValues.includes(value)) {
          errors.push({
            field: `option_${selected.optionGroupId}`,
            message: `無効な値が選択されています: ${value}`,
            code: "INVALID_OPTION_VALUE",
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
  name: z
    .string()
    .min(1, "メニュー名は必須です")
    .max(200, "メニュー名は200文字以内で入力してください"),
  description: z.string().optional(),
  imageUrl: z.string().url("画像URLの形式が不正です").optional(),
  price: z
    .number()
    .nonnegative("価格は0以上で設定してください")
    .int("価格は整数で入力してください"),
  availableDate: z.string().datetime(),
  orderDeadline: z.string().datetime(),
  maxQuantity: z.number().positive().optional(),
  options: z
    .array(
      z.object({
        name: z.string(),
        type: z.nativeEnum(OptionType),
        isRequired: z.boolean(),
        items: z.array(
          z.object({
            value: z.string(),
            label: z.string(),
          })
        ),
      })
    )
    .optional(),
});

export class MenuValidator {
  static validateCreateMenu(input: CreateMenuInput): Result<void, DomainError> {
    const result = CreateMenuSchema.safeParse(input);

    if (!result.success) {
      const errors: ValidationError[] = result.error.errors.map((error) => ({
        field: error.path.join("."),
        message: error.message,
        code: "VALIDATION_ERROR",
      }));

      return err(DomainError.validation(errors));
    }

    // ビジネスルールの検証
    const availableDate = new Date(input.availableDate);
    const orderDeadline = new Date(input.orderDeadline);

    if (orderDeadline >= availableDate) {
      return err(
        DomainError.validation([
          {
            field: "orderDeadline",
            message: "注文締切は受け渡し日より前に設定してください",
            code: "INVALID_DEADLINE",
          },
        ])
      );
    }

    // 前々日17時の確認
    const expectedDeadline = new Date(availableDate);
    expectedDeadline.setDate(expectedDeadline.getDate() - 2);
    expectedDeadline.setHours(17, 0, 0, 0);

    if (
      Math.abs(orderDeadline.getTime() - expectedDeadline.getTime()) > 60000
    ) {
      return err(
        DomainError.validation([
          {
            field: "orderDeadline",
            message: "注文締切は受け渡し日の前々日17時に設定してください",
            code: "INVALID_DEADLINE_TIME",
          },
        ])
      );
    }

    return ok(undefined);
  }
}
```
