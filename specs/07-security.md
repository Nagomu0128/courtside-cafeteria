# 8. セキュリティ仕様（neverthrow版）

## 原則：最小権限の原則、深層防御、neverthrowによる安全なエラーハンドリング

```typescript
import { Result, ResultAsync, ok, err } from "neverthrow";
import * as crypto from "crypto";

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
        Promise.resolve(err(DomainError.unauthorized("無効なトークンです")))
      );
    }

    return this.userRepository
      .findBySessionToken(token)
      .andThen((user) => {
        if (!user) {
          return err(DomainError.unauthorized("セッションが見つかりません"));
        }

        const lastAccess = new Date(user.lastAccessedAt);
        const now = new Date();
        const hoursSinceLastAccess =
          (now.getTime() - lastAccess.getTime()) / (1000 * 60 * 60);

        if (hoursSinceLastAccess > this.config.sessionTimeoutHours) {
          return err(
            new DomainError(
              "セッションの有効期限が切れました",
              "SESSION_EXPIRED"
            )
          );
        }

        return ok(user);
      })
      .andThen((user) =>
        // アクセス時刻更新（エラーは無視）
        this.userRepository
          .updateLastAccessed(user.id)
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
      .andThen((count) => {
        if (count >= limit) {
          return err(
            new DomainError(
              `リクエストが多すぎます。しばらくお待ちください`,
              "RATE_LIMIT_EXCEEDED",
              429
            )
          );
        }
        return ok(undefined);
      })
      .andThen(() => this.incrementActionCount(key, window));
  }

  // CSRFトークン生成
  generateCSRFToken(): Result<string, DomainError> {
    try {
      const token = crypto.randomBytes(32).toString("hex");
      return ok(token);
    } catch (error) {
      return err(DomainError.internal("CSRFトークンの生成に失敗しました"));
    }
  }

  // CSRFトークン検証
  validateCSRFToken(
    token: string,
    sessionToken: string
  ): Result<void, DomainError> {
    if (!token || !sessionToken) {
      return err(DomainError.unauthorized("CSRFトークンが無効です"));
    }

    // トークンペアの検証ロジック
    const isValid = this.verifyTokenPair(token, sessionToken);
    if (!isValid) {
      return err(DomainError.unauthorized("CSRFトークンが一致しません"));
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
        .replace(/&/g, "&amp;")
        .replace(/</g, "&lt;")
        .replace(/>/g, "&gt;")
        .replace(/"/g, "&quot;")
        .replace(/'/g, "&#x27;")
        .replace(/\//g, "&#x2F;");
      return ok(sanitized);
    } catch (error) {
      return err(DomainError.internal("入力のサニタイズに失敗しました"));
    }
  }

  // SQLインジェクション対策
  static validateSqlInput(input: string): Result<string, DomainError> {
    if (/[';--]/.test(input)) {
      return err(
        DomainError.validation([
          {
            field: "input",
            message: "不正な文字が含まれています",
            code: "INVALID_CHARACTERS",
          },
        ])
      );
    }
    return ok(input);
  }
}

// infrastructure/security/EncryptionService.ts
export class EncryptionService {
  private algorithm = "aes-256-gcm";
  private key: Buffer;

  constructor() {
    const keyString = process.env.ENCRYPTION_KEY;
    if (!keyString) {
      throw new Error("暗号化キーが設定されていません");
    }
    this.key = Buffer.from(keyString, "hex");
  }

  // 個人情報の暗号化
  encrypt(text: string): Result<EncryptedData, DomainError> {
    try {
      const iv = crypto.randomBytes(16);
      const cipher = crypto.createCipheriv(this.algorithm, this.key, iv);

      let encrypted = cipher.update(text, "utf8", "hex");
      encrypted += cipher.final("hex");

      const authTag = cipher.getAuthTag();

      return ok({
        encrypted,
        iv: iv.toString("hex"),
        authTag: authTag.toString("hex"),
      });
    } catch (error) {
      return err(DomainError.internal("暗号化に失敗しました"));
    }
  }

  // 復号化
  decrypt(data: EncryptedData): Result<string, DomainError> {
    try {
      const decipher = crypto.createDecipheriv(
        this.algorithm,
        this.key,
        Buffer.from(data.iv, "hex")
      );

      decipher.setAuthTag(Buffer.from(data.authTag, "hex"));

      let decrypted = decipher.update(data.encrypted, "hex", "utf8");
      decrypted += decipher.final("utf8");

      return ok(decrypted);
    } catch (error) {
      return err(DomainError.internal("復号化に失敗しました"));
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
