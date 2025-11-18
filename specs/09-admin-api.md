# 10. 管理者API仕様（neverthrow版）

```typescript
import { ResultAsync, ok, err, combine } from "neverthrow";

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
    return this.menuRepository
      .findById(input.menuId)
      .andThen((menu) => {
        if (!menu) {
          return err(DomainError.menuNotFound());
        }
        return ok(menu);
      })
      .andThen((menu) =>
        this.orderRepository
          .countByMenuIdAndOptions(input.menuId, input.filters)
          .map((counts) => ({ menu, counts }))
      )
      .map(({ menu, counts }) => {
        // 集計処理
        const totalOrders = counts.reduce((sum, c) => sum + c.count, 0);

        const byDepartment = this.groupBy(counts, "department");
        const byGender = this.groupBy(counts, "gender");
        const byAgeGroup = this.groupBy(counts, "ageGroup");
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
            byOptions,
          },
          generatedAt: new Date(),
        };
      });
  }

  // CSVエクスポート
  exportOrders(
    input: ExportOrdersInput
  ): ResultAsync<ExportResult, DomainError> {
    return ResultAsync.combine([
      this.menuRepository.findById(input.menuId),
      this.orderRepository.findByMenuId(input.menuId),
    ]).andThen(([menu, orders]) => {
      if (!menu) {
        return err(DomainError.menuNotFound());
      }

      const data = orders.map((order) => ({
        注文番号: order.orderNumber,
        注文日時: format(order.orderedAt, "yyyy-MM-dd HH:mm:ss"),
        部署: order.userInfo.department,
        名前: order.userInfo.name,
        性別: this.translateGender(order.userInfo.gender),
        年齢層: this.translateAgeGroup(order.userInfo.ageGroup),
        ...this.flattenOptions(order.selectedOptions),
        ステータス: this.translateStatus(order.status),
        変更日時: order.modifiedAt
          ? format(order.modifiedAt, "yyyy-MM-dd HH:mm:ss")
          : "",
        キャンセル日時: order.cancelledAt
          ? format(order.cancelledAt, "yyyy-MM-dd HH:mm:ss")
          : "",
      }));

      const exportPromise =
        input.format === "CSV"
          ? this.exportService.exportToCSV(data)
          : this.exportService.exportToExcel(data, {
              sheetName: `注文一覧_${format(menu.availableDate, "yyyyMMdd")}`,
              autoFilter: true,
              freezePane: { row: 1, column: 3 },
            });

      return ResultAsync.fromPromise(exportPromise, (error) =>
        DomainError.internal("エクスポートに失敗しました")
      ).map((buffer) => ({
        fileName: `orders_${input.menuId}_${format(new Date(), "yyyyMMddHHmmss")}.${input.format.toLowerCase()}`,
        mimeType:
          input.format === "CSV"
            ? "text/csv"
            : "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        buffer,
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
      description: input.description || "",
      imageUrl: input.imageUrl || "",
      price: priceResult.value,
      availableDate: new Date(input.availableDate),
      orderDeadline: new Date(input.orderDeadline),
      maxQuantity: input.maxQuantity,
      status: MenuStatus.DRAFT,
      options: [],
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    return this.menuRepository.save(menu).andThen((savedMenu) => {
      if (input.options && input.options.length > 0) {
        return this.setupMenuOptions(savedMenu.id, input.options).map(
          () => savedMenu
        );
      }
      return ok(savedMenu);
    });
  }

  updateMenuStatus(
    menuId: string,
    status: MenuStatus
  ): ResultAsync<Menu, DomainError> {
    return this.menuRepository
      .findById(menuId)
      .andThen((menu) => {
        if (!menu) {
          return err(DomainError.menuNotFound());
        }

        // ステータス変更のビジネスルール
        if (menu.status === MenuStatus.CANCELLED) {
          return err(
            new DomainError(
              "キャンセル済みのメニューは変更できません",
              "INVALID_STATUS_TRANSITION"
            )
          );
        }

        if (status === MenuStatus.ACTIVE && new Date() > menu.orderDeadline) {
          return err(
            new DomainError(
              "締切を過ぎたメニューは公開できません",
              "INVALID_STATUS_TRANSITION"
            )
          );
        }

        menu.status = status;
        menu.updatedAt = new Date();

        return ok(menu);
      })
      .andThen((menu) => this.menuRepository.update(menu));
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

    const total = Object.values(grouped).reduce(
      (sum: number, count: any) => sum + count,
      0
    );

    return Object.entries(grouped).map(([key, count]) => ({
      key,
      count: count as number,
      percentage: total > 0 ? ((count as number) / total) * 100 : 0,
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
        count,
      })),
    }));
  }

  private flattenOptions(options: SelectedOption[]): any {
    const flattened: any = {};
    for (const option of options) {
      const value = Array.isArray(option.selectedValue)
        ? option.selectedValue.join(", ")
        : option.selectedValue;
      flattened[`オプション_${option.optionGroupId}`] = value;
    }
    return flattened;
  }

  private translateGender(gender: Gender): string {
    const translations = {
      [Gender.MALE]: "男性",
      [Gender.FEMALE]: "女性",
      [Gender.OTHER]: "その他",
    };
    return translations[gender] || gender;
  }

  private translateAgeGroup(ageGroup: AgeGroup): string {
    const translations = {
      [AgeGroup.UNDER_20]: "20歳未満",
      [AgeGroup.TWENTIES]: "20代",
      [AgeGroup.THIRTIES]: "30代",
      [AgeGroup.FORTIES]: "40代",
      [AgeGroup.FIFTIES]: "50代",
      [AgeGroup.OVER_60]: "60歳以上",
    };
    return translations[ageGroup] || ageGroup;
  }

  private translateStatus(status: OrderStatus): string {
    const translations = {
      [OrderStatus.CONFIRMED]: "確定",
      [OrderStatus.MODIFIED]: "変更済",
      [OrderStatus.CANCELLED]: "キャンセル",
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
