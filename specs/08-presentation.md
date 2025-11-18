# 9. プレゼンテーション層（コントローラー）

## 原則：neverthrowのmatchによるパターンマッチング

```typescript
import { Request, Response } from "express";
import { ResultAsync } from "neverthrow";

// presentation/controllers/OrderController.ts
export class OrderController {
  constructor(
    private orderService: OrderService,
    private securityMiddleware: SecurityMiddleware
  ) {}

  async createOrder(req: Request, res: Response): Promise<void> {
    const sessionToken = req.headers["x-session-token"] as string;

    // セッション検証とレート制限を組み合わせる
    await this.securityMiddleware
      .validateSession(sessionToken)
      .andThen((user) =>
        this.securityMiddleware
          .checkRateLimit(user.id, "create_order")
          .map(() => user)
      )
      .andThen((user) =>
        // 注文作成
        this.orderService.createOrder({
          userId: user.id,
          menuId: req.body.menuId,
          userInfo: {
            department: req.body.department,
            name: req.body.name,
            gender: req.body.gender,
            ageGroup: req.body.ageGroup,
          },
          selectedOptions: req.body.options,
        })
      )
      .match(
        // 成功時
        (result) => {
          res.status(201).json({
            success: true,
            data: {
              orderNumber: result.orderNumber,
              message: result.message,
            },
          });
        },
        // エラー時
        (error) => {
          res.status(error.statusCode).json({
            success: false,
            error: {
              code: error.code,
              message: error.message,
              details: error.details,
            },
          });
        }
      );
  }

  async modifyOrder(req: Request, res: Response): Promise<void> {
    const sessionToken = req.headers["x-session-token"] as string;

    await this.orderService
      .modifyOrder({
        orderNumber: req.params.orderNumber,
        sessionToken,
        userInfo: {
          department: req.body.department,
          name: req.body.name,
          gender: req.body.gender,
          ageGroup: req.body.ageGroup,
        },
        selectedOptions: req.body.options,
      })
      .match(
        (result) => {
          res.json({
            success: true,
            data: {
              message: result.message,
            },
          });
        },
        (error) => {
          res.status(error.statusCode).json({
            success: false,
            error: {
              code: error.code,
              message: error.message,
              details: error.details,
            },
          });
        }
      );
  }

  async cancelOrder(req: Request, res: Response): Promise<void> {
    const sessionToken = req.headers["x-session-token"] as string;

    await this.orderService
      .cancelOrder({
        orderNumber: req.params.orderNumber,
        sessionToken,
      })
      .match(
        (result) => {
          res.json({
            success: true,
            data: {
              message: result.message,
            },
          });
        },
        (error) => {
          res.status(error.statusCode).json({
            success: false,
            error: {
              code: error.code,
              message: error.message,
            },
          });
        }
      );
  }

  async getOrderHistory(req: Request, res: Response): Promise<void> {
    const sessionToken = req.headers["x-session-token"] as string;

    await this.securityMiddleware
      .validateSession(sessionToken)
      .andThen((user) => this.orderService.getOrderHistory(user.id))
      .match(
        (orders) => {
          res.json({
            success: true,
            data: orders,
          });
        },
        (error) => {
          res.status(error.statusCode).json({
            success: false,
            error: {
              code: error.code,
              message: error.message,
            },
          });
        }
      );
  }
}
```
