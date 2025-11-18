# 13. QRコード認証仕様

## 概要

QRコードを読み取ることで、ユーザーが自動的に匿名認証され、注文機能を利用できるようにする仕様です。

### 原則

- **フリクションレス**: ログインフォーム不要、QRコードをスキャンするだけで利用開始
- **匿名性**: 個人を特定する情報は不要（注文時に任意で入力）
- **セキュリティ**: セッションベースの認証とCSRF対策
- **型安全**: neverthrowによる明示的なエラーハンドリング

---

## 1. 認証フロー

### 1.1 全体フロー

```
┌─────────────┐
│  QRコード    │  固定URL: https://cafeteria.example.com/
└──────┬──────┘
       │ スキャン
       ▼
┌─────────────────────────────────────────┐
│  GET /                                  │
│  - セッションCookie確認                  │
│  - 既存 → メニュー画面へリダイレクト        │
│  - 新規 → 匿名セッション作成              │
└──────┬──────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  POST /api/auth/anonymous               │
│  - UUID生成                             │
│  - セッショントークン生成                 │
│  - Cookieにセット                        │
│  - メニュー画面へリダイレクト              │
└──────┬──────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  メニュー画面                            │
│  - セッション付きで注文可能               │
└─────────────────────────────────────────┘
```

### 1.2 QRコード設計

```typescript
// QRコードに埋め込む内容（シンプルな固定URL）
const QR_CODE_URL = "https://cafeteria.example.com/";

// または、環境パラメータを含める場合
const QR_CODE_URL = "https://cafeteria.example.com/?source=qr&location=floor2";
```

**設計方針**:

- QRコードには固定URLのみ埋め込む（一時トークン不要）
- シンプルで運用しやすい
- QRコードの印刷・配布が容易

---

## 2. ドメイン層

### 2.1 エラー型拡張

```typescript
// domain/errors/DomainError.ts（既存に追加）
export type DomainErrorCode =
  | "ORDER_DEADLINE_PASSED"
  | "DUPLICATE_ORDER"
  // ... 既存のエラーコード
  | "SESSION_CREATION_FAILED"
  | "INVALID_SESSION_TOKEN"
  | "SESSION_EXPIRED"
  | "SESSION_NOT_FOUND";

export class DomainError {
  // ... 既存のファクトリーメソッド

  static sessionCreationFailed(details?: string): DomainError {
    return new DomainError(
      "セッションの作成に失敗しました",
      "SESSION_CREATION_FAILED",
      500,
      { details }
    );
  }

  static invalidSessionToken(): DomainError {
    return new DomainError(
      "無効なセッショントークンです",
      "INVALID_SESSION_TOKEN",
      401
    );
  }

  static sessionExpired(): DomainError {
    return new DomainError(
      "セッションの有効期限が切れました",
      "SESSION_EXPIRED",
      401
    );
  }

  static sessionNotFound(): DomainError {
    return new DomainError(
      "セッションが見つかりません",
      "SESSION_NOT_FOUND",
      404
    );
  }
}
```

### 2.2 セッションエンティティ（既存Userエンティティを使用）

```typescript
// domain/entities/User.ts（既存）
export interface User {
  id: string; // UUID（匿名ユーザーID）
  sessionToken: string; // セッション管理用トークン（64文字以上）
  createdAt: Date; // アカウント作成日時
  lastAccessedAt: Date; // 最終アクセス日時
  profile?: UserProfile; // ユーザープロファイル（任意）
  metadata?: SessionMetadata; // セッションメタデータ
}

export interface SessionMetadata {
  source?: string; // 'qr', 'web', 'mobile'
  location?: string; // QRコードの設置場所（例: 'floor2'）
  userAgent?: string; // ユーザーエージェント
  ipAddress?: string; // IPアドレス（匿名化推奨）
}
```

---

## 3. アプリケーション層

### 3.1 AnonymousAuthService

```typescript
// application/services/AnonymousAuthService.ts
import { Result, ResultAsync, ok, err } from "neverthrow";
import * as crypto from "crypto";

export class AnonymousAuthService {
  constructor(
    private userRepository: IUserRepository,
    private securityMiddleware: SecurityMiddleware
  ) {}

  /**
   * 匿名セッションを作成
   * QRコードアクセス時に呼び出される
   */
  createAnonymousSession(
    request: CreateSessionRequest
  ): ResultAsync<CreateSessionResponse, DomainError> {
    // セッショントークン生成（原則：暗号学的に安全な乱数）
    const sessionTokenResult = this.generateSecureToken();
    if (sessionTokenResult.isErr()) {
      return ResultAsync.fromSafePromise(
        Promise.resolve(err(sessionTokenResult.error))
      );
    }

    const sessionToken = sessionTokenResult.value;
    const userId = this.generateUUID();

    const user: User = {
      id: userId,
      sessionToken,
      createdAt: new Date(),
      lastAccessedAt: new Date(),
      metadata: {
        source: request.source || "qr",
        location: request.location,
        userAgent: request.userAgent,
        ipAddress: this.anonymizeIP(request.ipAddress),
      },
    };

    return this.userRepository.save(user).map((savedUser) => ({
      userId: savedUser.id,
      sessionToken: savedUser.sessionToken,
      expiresAt: this.calculateExpirationDate(),
      redirectUrl: "/menu",
    }));
  }

  /**
   * 既存セッションを検証
   * すべてのAPIリクエストで呼び出される
   */
  validateSession(sessionToken: string): ResultAsync<User, DomainError> {
    return this.securityMiddleware.validateSession(sessionToken);
  }

  /**
   * セッションを更新（アクセス時刻を更新）
   */
  refreshSession(userId: string): ResultAsync<void, DomainError> {
    return this.userRepository.updateLastAccessed(userId);
  }

  /**
   * セッションを無効化（ログアウト）
   */
  invalidateSession(sessionToken: string): ResultAsync<void, DomainError> {
    return this.userRepository
      .findBySessionToken(sessionToken)
      .andThen((user) => {
        if (!user) {
          return err(DomainError.sessionNotFound());
        }
        return this.userRepository.delete(user.id);
      });
  }

  // Private helpers

  private generateSecureToken(): Result<string, DomainError> {
    try {
      // 64バイト（512ビット）のランダムトークン
      const token = crypto.randomBytes(64).toString("base64url");
      return ok(token);
    } catch (error) {
      return err(DomainError.sessionCreationFailed("トークン生成エラー"));
    }
  }

  private generateUUID(): string {
    return crypto.randomUUID();
  }

  private calculateExpirationDate(): Date {
    // セッション有効期限: 24時間
    const expiresAt = new Date();
    expiresAt.setHours(expiresAt.getHours() + 24);
    return expiresAt;
  }

  private anonymizeIP(ip?: string): string | undefined {
    if (!ip) return undefined;
    // IPv4: 最後のオクテットをマスク（例: 192.168.1.xxx → 192.168.1.0）
    // IPv6: 下位64ビットをマスク
    const parts = ip.split(".");
    if (parts.length === 4) {
      return `${parts[0]}.${parts[1]}.${parts[2]}.0`;
    }
    return ip; // IPv6は簡略化のためそのまま返す
  }
}

// Types
export interface CreateSessionRequest {
  source?: string; // 'qr', 'web', etc.
  location?: string; // QRコードの場所
  userAgent?: string; // User-Agent header
  ipAddress?: string; // クライアントIP
}

export interface CreateSessionResponse {
  userId: string;
  sessionToken: string;
  expiresAt: Date;
  redirectUrl: string;
}
```

---

## 4. プレゼンテーション層（API仕様）

### 4.1 ルートアクセス（QRコード読み取り時）

```typescript
// app/page.tsx（サーバーコンポーネント）
import { cookies } from "next/headers";
import { redirect } from "next/navigation";
import { AnonymousAuthService } from "@/src/application/services/AnonymousAuthService";

export default async function HomePage({
  searchParams,
}: {
  searchParams: { source?: string; location?: string };
}) {
  const cookieStore = await cookies();
  const sessionToken = cookieStore.get("session_token")?.value;

  // 既存セッションの検証
  if (sessionToken) {
    const authService = new AnonymousAuthService(/* dependencies */);
    const validationResult = await authService.validateSession(sessionToken);

    if (validationResult.isOk()) {
      // セッション有効 → メニュー画面へ
      redirect("/menu");
    }
    // セッション無効の場合は新規作成へ進む
  }

  // 新規セッション作成へリダイレクト
  const params = new URLSearchParams({
    source: searchParams.source || "qr",
    location: searchParams.location || "",
  });

  redirect(`/api/auth/anonymous?${params.toString()}`);
}
```

### 4.2 匿名認証APIエンドポイント

#### POST /api/auth/anonymous

**目的**: 匿名セッションを作成し、Cookieにセット

**リクエスト**:

```typescript
// app/api/auth/anonymous/route.ts
import { NextRequest, NextResponse } from "next/server";
import { AnonymousAuthService } from "@/src/application/services/AnonymousAuthService";

export async function POST(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const userAgent = request.headers.get("user-agent") || undefined;
  const ipAddress =
    request.headers.get("x-forwarded-for") ||
    request.headers.get("x-real-ip") ||
    undefined;

  const authService = new AnonymousAuthService(/* dependencies */);

  const result = await authService.createAnonymousSession({
    source: searchParams.get("source") || "qr",
    location: searchParams.get("location") || undefined,
    userAgent,
    ipAddress,
  });

  if (result.isErr()) {
    return NextResponse.json(
      {
        error: result.error.message,
        code: result.error.code,
      },
      { status: result.error.statusCode }
    );
  }

  const { sessionToken, expiresAt, redirectUrl } = result.value;

  // セッションCookieをセット
  const response = NextResponse.redirect(new URL(redirectUrl, request.url));

  response.cookies.set("session_token", sessionToken, {
    httpOnly: true, // XSS対策
    secure: process.env.NODE_ENV === "production", // HTTPS必須
    sameSite: "lax", // CSRF対策
    expires: expiresAt, // 有効期限
    path: "/", // 全パスで有効
  });

  return response;
}

// GETリクエストも受け付ける（QRコードからの直接アクセス用）
export async function GET(request: NextRequest) {
  return POST(request);
}
```

**レスポンス**:

- 成功: 302 Redirect to /menu（Cookieにセッショントークンをセット）
- 失敗: JSON形式のエラー

```json
// エラー例
{
  "error": "セッションの作成に失敗しました",
  "code": "SESSION_CREATION_FAILED"
}
```

---

### 4.3 セッション検証ミドルウェア

```typescript
// app/middleware.ts
import { NextRequest, NextResponse } from "next/server";
import { AnonymousAuthService } from "@/src/application/services/AnonymousAuthService";

export async function middleware(request: NextRequest) {
  const sessionToken = request.cookies.get("session_token")?.value;

  // 公開パスはスキップ
  const publicPaths = ["/", "/api/auth/anonymous"];
  if (publicPaths.includes(request.nextUrl.pathname)) {
    return NextResponse.next();
  }

  // セッションチェック
  if (!sessionToken) {
    return NextResponse.redirect(new URL("/", request.url));
  }

  const authService = new AnonymousAuthService(/* dependencies */);
  const validationResult = await authService.validateSession(sessionToken);

  if (validationResult.isErr()) {
    // セッション無効 → ルートへリダイレクト
    const response = NextResponse.redirect(new URL("/", request.url));
    response.cookies.delete("session_token");
    return response;
  }

  // セッション有効 → アクセス時刻を更新
  await authService.refreshSession(validationResult.value.id);

  return NextResponse.next();
}

export const config = {
  matcher: [
    /*
     * Match all request paths except:
     * - _next/static (static files)
     * - _next/image (image optimization files)
     * - favicon.ico (favicon file)
     * - public folder
     */
    "/((?!_next/static|_next/image|favicon.ico|public).*)",
  ],
};
```

---

### 4.4 ログアウトAPI

#### POST /api/auth/logout

```typescript
// app/api/auth/logout/route.ts
import { NextRequest, NextResponse } from "next/server";
import { AnonymousAuthService } from "@/src/application/services/AnonymousAuthService";

export async function POST(request: NextRequest) {
  const sessionToken = request.cookies.get("session_token")?.value;

  if (!sessionToken) {
    return NextResponse.json(
      { error: "セッションが見つかりません" },
      { status: 400 }
    );
  }

  const authService = new AnonymousAuthService(/* dependencies */);
  const result = await authService.invalidateSession(sessionToken);

  const response = NextResponse.json({
    success: true,
    message: "ログアウトしました",
  });

  // Cookieを削除
  response.cookies.delete("session_token");

  return response;
}
```

---

## 5. インフラストラクチャ層

### 5.1 UserRepository拡張

```typescript
// infrastructure/persistence/supabase/repositories/SupabaseUserRepository.ts
import { ResultAsync, ok, err } from "neverthrow";
import { createClient } from "@supabase/supabase-js";

export class SupabaseUserRepository implements IUserRepository {
  constructor(private supabase: ReturnType<typeof createClient>) {}

  save(user: User): ResultAsync<User, DomainError> {
    return ResultAsync.fromPromise(
      this.supabase
        .from("users")
        .insert({
          id: user.id,
          session_token: user.sessionToken,
          created_at: user.createdAt.toISOString(),
          last_accessed_at: user.lastAccessedAt.toISOString(),
          metadata: user.metadata || null,
        })
        .select()
        .single(),
      (error) => DomainError.internal(`ユーザー保存エラー: ${error}`)
    ).andThen(({ data, error }) => {
      if (error) {
        return err(DomainError.internal(`保存エラー: ${error.message}`));
      }
      return ok(this.toDomain(data));
    });
  }

  findBySessionToken(token: string): ResultAsync<User | null, DomainError> {
    return ResultAsync.fromPromise(
      this.supabase
        .from("users")
        .select("*")
        .eq("session_token", token)
        .maybeSingle(),
      (error) => DomainError.internal(`検索エラー: ${error}`)
    ).andThen(({ data, error }) => {
      if (error) {
        return err(DomainError.internal(`検索エラー: ${error.message}`));
      }
      return ok(data ? this.toDomain(data) : null);
    });
  }

  updateLastAccessed(id: string): ResultAsync<void, DomainError> {
    return ResultAsync.fromPromise(
      this.supabase
        .from("users")
        .update({ last_accessed_at: new Date().toISOString() })
        .eq("id", id),
      (error) => DomainError.internal(`更新エラー: ${error}`)
    ).map(() => undefined);
  }

  delete(id: string): ResultAsync<void, DomainError> {
    return ResultAsync.fromPromise(
      this.supabase.from("users").delete().eq("id", id),
      (error) => DomainError.internal(`削除エラー: ${error}`)
    ).map(() => undefined);
  }

  private toDomain(data: any): User {
    return {
      id: data.id,
      sessionToken: data.session_token,
      createdAt: new Date(data.created_at),
      lastAccessedAt: new Date(data.last_accessed_at),
      metadata: data.metadata,
    };
  }
}
```

---

## 6. データベーススキーマ拡張

### 6.1 usersテーブルにメタデータカラムを追加

```sql
-- 既存のusersテーブルに追加
ALTER TABLE users ADD COLUMN metadata JSONB;

-- メタデータ用インデックス（sourceでの検索を高速化）
CREATE INDEX idx_users_metadata_source ON users ((metadata->>'source'));

-- セッショントークンの有効期限管理用インデックス
CREATE INDEX idx_users_last_accessed ON users(last_accessed_at);

-- 古いセッションを定期削除する関数
CREATE OR REPLACE FUNCTION delete_expired_sessions()
RETURNS void AS $$
BEGIN
  DELETE FROM users
  WHERE last_accessed_at < NOW() - INTERVAL '24 hours';
END;
$$ LANGUAGE plpgsql;

-- 定期実行（pg_cronなどで設定）
-- SELECT cron.schedule('delete-expired-sessions', '0 * * * *', 'SELECT delete_expired_sessions()');
```

---

## 7. セキュリティ考慮事項

### 7.1 CSRF対策

```typescript
// infrastructure/security/CSRFProtection.ts
import { Result, ok, err } from "neverthrow";
import * as crypto from "crypto";

export class CSRFProtection {
  /**
   * CSRFトークンを生成（セッション作成時）
   */
  generateCSRFToken(sessionToken: string): Result<string, DomainError> {
    try {
      // セッショントークンとランダム値からCSRFトークンを生成
      const randomValue = crypto.randomBytes(32).toString("hex");
      const csrfToken = crypto
        .createHmac("sha256", process.env.CSRF_SECRET || "default-secret")
        .update(`${sessionToken}:${randomValue}`)
        .digest("hex");

      return ok(csrfToken);
    } catch (error) {
      return err(DomainError.internal("CSRFトークン生成エラー"));
    }
  }

  /**
   * CSRFトークンを検証（POST/PUT/DELETE時）
   */
  validateCSRFToken(
    csrfToken: string,
    sessionToken: string
  ): Result<void, DomainError> {
    if (!csrfToken || !sessionToken) {
      return err(DomainError.unauthorized("CSRFトークンが無効です"));
    }

    // トークンの形式検証
    if (!/^[a-f0-9]{64}$/.test(csrfToken)) {
      return err(DomainError.unauthorized("CSRFトークン形式が不正です"));
    }

    // 実際の検証はセッションストアに保存されたトークンと比較
    // （簡略化のため詳細は省略）
    return ok(undefined);
  }
}
```

### 7.2 レート制限（QRコード認証専用）

```typescript
// infrastructure/security/QRAuthRateLimiter.ts
import { ResultAsync, ok, err } from "neverthrow";

export class QRAuthRateLimiter {
  constructor(
    private redis: RedisClient, // または他のKVS
    private config: RateLimitConfig
  ) {}

  /**
   * QRコード認証のレート制限
   * 同一IPから短時間に大量のセッション作成を防ぐ
   */
  checkLimit(ipAddress: string): ResultAsync<void, DomainError> {
    const key = `qr_auth_rate:${ipAddress}`;
    const limit = this.config.maxSessionsPerIP || 5; // 5分間に5セッションまで
    const window = 300; // 5分

    return ResultAsync.fromPromise(this.redis.get(key), (error) =>
      DomainError.internal("レート制限チェックエラー")
    ).andThen((count) => {
      const currentCount = count ? parseInt(count, 10) : 0;

      if (currentCount >= limit) {
        return err(
          new DomainError(
            "セッション作成が制限されています。しばらくお待ちください",
            "RATE_LIMIT_EXCEEDED",
            429
          )
        );
      }

      // カウントをインクリメント
      return ResultAsync.fromPromise(
        this.redis.incr(key).then(() => this.redis.expire(key, window)),
        (error) => DomainError.internal("レート制限更新エラー")
      ).map(() => undefined);
    });
  }
}

interface RateLimitConfig {
  maxSessionsPerIP: number;
  windowSeconds: number;
}
```

### 7.3 セッションハイジャック対策

```typescript
// infrastructure/security/SessionValidator.ts
export class SessionValidator {
  /**
   * セッションのフィンガープリント検証
   * User-Agentが変わった場合はセッション無効化
   */
  validateFingerprint(
    user: User,
    currentUserAgent: string
  ): Result<void, DomainError> {
    const storedUserAgent = user.metadata?.userAgent;

    if (storedUserAgent && storedUserAgent !== currentUserAgent) {
      return err(
        new DomainError(
          "セッションが無効です。再度QRコードをスキャンしてください",
          "SESSION_HIJACK_DETECTED",
          401
        )
      );
    }

    return ok(undefined);
  }
}
```

---

## 8. エラーハンドリング

### 8.1 エラーレスポンス標準化

```typescript
// presentation/utils/errorHandler.ts
import { DomainError } from "@/src/domain/errors/DomainError";

export function handleAuthError(error: DomainError): Response {
  const statusCode = error.statusCode || 500;

  // ログ出力（本番環境では構造化ログ推奨）
  console.error("[Auth Error]", {
    code: error.code,
    message: error.message,
    details: error.details,
    timestamp: new Date().toISOString(),
  });

  // ユーザーフレンドリーなメッセージ
  const userMessage = getUserFriendlyMessage(error.code);

  return new Response(
    JSON.stringify({
      error: userMessage,
      code: error.code,
      statusCode,
    }),
    {
      status: statusCode,
      headers: { "Content-Type": "application/json" },
    }
  );
}

function getUserFriendlyMessage(code: string): string {
  const messages: Record<string, string> = {
    SESSION_CREATION_FAILED:
      "セッションの作成に失敗しました。もう一度QRコードをスキャンしてください",
    SESSION_EXPIRED:
      "セッションの有効期限が切れました。QRコードを再度スキャンしてください",
    INVALID_SESSION_TOKEN:
      "セッションが無効です。QRコードを再度スキャンしてください",
    RATE_LIMIT_EXCEEDED: "アクセスが集中しています。しばらくお待ちください",
  };

  return messages[code] || "エラーが発生しました";
}
```

---

## 9. フロントエンド統合

### 9.1 セッション状態管理（React）

```typescript
// app/hooks/useSession.ts
"use client";

import { useEffect, useState } from "react";

export function useSession() {
  const [isAuthenticated, setIsAuthenticated] = useState<boolean>(false);
  const [isLoading, setIsLoading] = useState<boolean>(true);

  useEffect(() => {
    // セッション確認（Cookieベースなのでサーバーサイドで検証済み）
    // クライアント側では状態のみ管理
    checkSession();
  }, []);

  const checkSession = async () => {
    try {
      const response = await fetch("/api/auth/session", {
        method: "GET",
        credentials: "include",
      });

      setIsAuthenticated(response.ok);
    } catch (error) {
      setIsAuthenticated(false);
    } finally {
      setIsLoading(false);
    }
  };

  const logout = async () => {
    await fetch("/api/auth/logout", {
      method: "POST",
      credentials: "include",
    });
    setIsAuthenticated(false);
    window.location.href = "/";
  };

  return { isAuthenticated, isLoading, logout };
}
```

### 9.2 セッション確認API

```typescript
// app/api/auth/session/route.ts
import { NextRequest, NextResponse } from "next/server";
import { AnonymousAuthService } from "@/src/application/services/AnonymousAuthService";

export async function GET(request: NextRequest) {
  const sessionToken = request.cookies.get("session_token")?.value;

  if (!sessionToken) {
    return NextResponse.json({ authenticated: false }, { status: 401 });
  }

  const authService = new AnonymousAuthService(/* dependencies */);
  const result = await authService.validateSession(sessionToken);

  if (result.isErr()) {
    return NextResponse.json(
      { authenticated: false, error: result.error.message },
      { status: 401 }
    );
  }

  return NextResponse.json({
    authenticated: true,
    userId: result.value.id,
    createdAt: result.value.createdAt,
  });
}
```

---

## 10. QRコード生成仕様

### 10.1 管理画面でのQRコード生成

```typescript
// app/admin/qr-codes/generate/route.ts
import { NextRequest, NextResponse } from "next/server";
import QRCode from "qrcode";

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const location = searchParams.get("location") || "";

  // QRコードURL
  const baseUrl =
    process.env.NEXT_PUBLIC_APP_URL || "https://cafeteria.example.com";
  const qrUrl = location
    ? `${baseUrl}/?source=qr&location=${encodeURIComponent(location)}`
    : `${baseUrl}/?source=qr`;

  try {
    // QRコード生成
    const qrCodeDataUrl = await QRCode.toDataURL(qrUrl, {
      width: 300,
      margin: 2,
      color: {
        dark: "#000000",
        light: "#FFFFFF",
      },
    });

    return NextResponse.json({
      qrCodeUrl: qrUrl,
      qrCodeImage: qrCodeDataUrl,
      location: location || "default",
    });
  } catch (error) {
    return NextResponse.json({ error: "QRコード生成エラー" }, { status: 500 });
  }
}
```

### 10.2 QRコード管理画面

```typescript
// app/admin/qr-codes/page.tsx
'use client';

import { useState } from 'react';
import { Button } from '@/app/components/ui/button';
import { Input } from '@/app/components/ui/input';

export default function QRCodeGeneratorPage() {
  const [location, setLocation] = useState('');
  const [qrCodeImage, setQRCodeImage] = useState<string | null>(null);

  const generateQRCode = async () => {
    const response = await fetch(
      `/admin/qr-codes/generate?location=${encodeURIComponent(location)}`
    );
    const data = await response.json();
    setQRCodeImage(data.qrCodeImage);
  };

  return (
    <div className="p-8">
      <h1 className="text-2xl font-bold mb-4">QRコード生成</h1>

      <div className="space-y-4">
        <div>
          <label className="block mb-2">設置場所（任意）</label>
          <Input
            value={location}
            onChange={(e) => setLocation(e.target.value)}
            placeholder="例: 2階食堂入口"
          />
        </div>

        <Button onClick={generateQRCode}>
          QRコード生成
        </Button>

        {qrCodeImage && (
          <div className="mt-4">
            <img src={qrCodeImage} alt="QR Code" />
            <Button
              onClick={() => {
                const link = document.createElement('a');
                link.download = `qr-code-${location || 'default'}.png`;
                link.href = qrCodeImage;
                link.click();
              }}
              className="mt-2"
            >
              ダウンロード
            </Button>
          </div>
        )}
      </div>
    </div>
  );
}
```

---

## 11. テスト仕様

### 11.1 単体テスト

```typescript
// src/application/services/__tests__/AnonymousAuthService.test.ts
import { describe, it, expect, vi } from "vitest";
import { ok, err } from "neverthrow";
import { AnonymousAuthService } from "../AnonymousAuthService";

describe("AnonymousAuthService", () => {
  describe("createAnonymousSession", () => {
    it("正常系: セッションが作成される", async () => {
      const mockUserRepository = {
        save: vi.fn().mockReturnValue(
          ok({
            id: "test-user-id",
            sessionToken: "test-token",
            createdAt: new Date(),
            lastAccessedAt: new Date(),
          })
        ),
      };

      const service = new AnonymousAuthService(
        mockUserRepository as any,
        {} as any
      );

      const result = await service.createAnonymousSession({
        source: "qr",
        location: "floor2",
      });

      expect(result.isOk()).toBe(true);
      if (result.isOk()) {
        expect(result.value.userId).toBe("test-user-id");
        expect(result.value.sessionToken).toBe("test-token");
      }
    });

    it("異常系: リポジトリエラー", async () => {
      const mockUserRepository = {
        save: vi
          .fn()
          .mockReturnValue(err(DomainError.internal("DB接続エラー"))),
      };

      const service = new AnonymousAuthService(
        mockUserRepository as any,
        {} as any
      );

      const result = await service.createAnonymousSession({
        source: "qr",
      });

      expect(result.isErr()).toBe(true);
      if (result.isErr()) {
        expect(result.error.code).toBe("INTERNAL_ERROR");
      }
    });
  });
});
```

### 11.2 E2Eテスト（Playwright）

```typescript
// e2e/qr-authentication.spec.ts
import { test, expect } from "@playwright/test";

test.describe("QRコード認証フロー", () => {
  test("QRコードアクセスで自動ログイン", async ({ page }) => {
    // QRコードURLにアクセス
    await page.goto("/?source=qr&location=floor2");

    // メニュー画面にリダイレクトされる
    await expect(page).toHaveURL("/menu");

    // セッションCookieが設定されている
    const cookies = await page.context().cookies();
    const sessionCookie = cookies.find((c) => c.name === "session_token");
    expect(sessionCookie).toBeDefined();
    expect(sessionCookie?.httpOnly).toBe(true);
    expect(sessionCookie?.sameSite).toBe("Lax");
  });

  test("セッション有効期限切れで再認証", async ({ page, context }) => {
    // 期限切れのセッションCookieをセット
    await context.addCookies([
      {
        name: "session_token",
        value: "expired-token",
        domain: "localhost",
        path: "/",
        httpOnly: true,
        sameSite: "Lax",
        expires: Math.floor(Date.now() / 1000) - 3600, // 1時間前に期限切れ
      },
    ]);

    // 保護されたページにアクセス
    await page.goto("/menu");

    // ルートにリダイレクトされる
    await expect(page).toHaveURL("/");
  });
});
```

---

## 12. 運用考慮事項

### 12.1 セッションクリーンアップ

```typescript
// scripts/cleanup-expired-sessions.ts
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_KEY!
);

async function cleanupExpiredSessions() {
  const expirationThreshold = new Date();
  expirationThreshold.setHours(expirationThreshold.getHours() - 24);

  const { data, error } = await supabase
    .from("users")
    .delete()
    .lt("last_accessed_at", expirationThreshold.toISOString());

  if (error) {
    console.error("セッションクリーンアップエラー:", error);
    return;
  }

  console.log(`${data?.length || 0}件の期限切れセッションを削除しました`);
}

// 定期実行（cron）: 0 * * * *（毎時0分）
cleanupExpiredSessions();
```

### 12.2 モニタリング指標

```typescript
// infrastructure/monitoring/SessionMetrics.ts
export interface SessionMetrics {
  // セッション作成
  totalSessionsCreated: number;
  sessionsCreatedBySource: Record<string, number>; // 'qr', 'web', etc.
  sessionsCreatedByLocation: Record<string, number>;

  // セッション検証
  sessionValidationAttempts: number;
  sessionValidationFailures: number;

  // エラー
  authErrors: Record<string, number>; // エラーコード別

  // パフォーマンス
  averageSessionCreationTime: number; // ミリ秒
}
```

---

## 13. 環境変数

```bash
# .env.local
# アプリケーションURL（QRコード生成用）
NEXT_PUBLIC_APP_URL=https://cafeteria.example.com

# セッション設定
SESSION_TIMEOUT_HOURS=24
SESSION_COOKIE_SECURE=true  # 本番環境ではtrue

# CSRF対策
CSRF_SECRET=your-csrf-secret-key-here

# Supabase
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_KEY=your-service-key

# レート制限（Redis）
REDIS_URL=redis://localhost:6379
QR_AUTH_RATE_LIMIT=5  # 5分間に5セッションまで
```

---

## 14. まとめ

### 14.1 実装優先順位

1. **Phase 1: 基本認証**
   - AnonymousAuthService実装
   - /api/auth/anonymous エンドポイント
   - セッションCookie管理

2. **Phase 2: セキュリティ強化**
   - CSRFトークン
   - レート制限
   - セッション検証ミドルウェア

3. **Phase 3: QRコード管理**
   - QRコード生成API
   - 管理画面UI

4. **Phase 4: 運用対応**
   - セッションクリーンアップ
   - モニタリング
   - ログ収集

### 14.2 技術スタック

| レイヤー | 技術                  | 用途                                 |
| -------- | --------------------- | ------------------------------------ |
| Frontend | Next.js 16 App Router | サーバーコンポーネント、ミドルウェア |
| Backend  | neverthrow            | 型安全なエラーハンドリング           |
| Database | Supabase (PostgreSQL) | セッション永続化                     |
| Cache    | Redis（推奨）         | レート制限、セッションキャッシュ     |
| QR Code  | qrcode library        | QRコード生成                         |

### 14.3 セキュリティチェックリスト

- [x] HTTPOnly Cookie（XSS対策）
- [x] Secure Cookie（本番環境）
- [x] SameSite=Lax（CSRF対策）
- [x] セッショントークン: 暗号学的に安全な乱数
- [x] レート制限（DDoS対策）
- [x] IPアドレス匿名化（プライバシー保護）
- [x] セッション有効期限管理
- [x] User-Agent検証（セッションハイジャック対策）

---

**最終更新**: 2025-11-17
**バージョン**: 1.0.0
