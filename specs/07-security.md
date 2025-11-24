# 8. セキュリティ仕様 (Firebase Auth)

## Firebase SDK 使用方針

### 原則：Admin SDKは最低限の特権操作のみに使用

| 操作                 | 使用SDK    | 理由                                 |
| -------------------- | ---------- | ------------------------------------ |
| IDトークン検証       | Admin SDK  | サーバーサイドでのトークン検証に必要 |
| カスタムクレーム付与 | Admin SDK  | 管理者権限付与はAdmin SDKでのみ可能  |
| Firestore CRUD       | Client SDK | セキュリティルールで保護、Admin不要  |
| Storage アップロード | Client SDK | セキュリティルールで保護、Admin不要  |

**Admin SDKを使用する場所（限定）：**

- `AdminAuthService` - 管理者認証のみ

**Client SDKを使用する場所：**

- `FirestoreOrderRepository` - 注文データ操作
- `FirestoreMenuRepository` - メニューデータ操作
- `FirebaseStorageService` - 画像アップロード

## 認証方式

### 1. ユーザー認証: Firebase Anonymous Auth

- **匿名認証**: 初回アクセス時はFirebaseの匿名認証を使用し、UIDを発行します。
- **プロファイル永続化**: 2回目以降の注文時、入力された個人情報（部署、氏名など）をFirestoreの `users/{uid}` ドキュメントに保存し、次回以降の入力を省略可能にします。
- **セッション管理**: Firebase Authのトークン（ID Token）を使用します。

### 2. 管理者認証: Custom Claims

- **認証フロー**:
  1. 管理者用ログイン画面でパスワードを入力。
  2. サーバーサイド（Cloud Functions または Next.js API）でパスワードハッシュを検証。
  3. 検証成功時、対象のUID（または管理者用UID）に対して `admin: true` のカスタムクレームを付与。
  4. クライアントサイドでトークンをリフレッシュし、管理者権限を取得。

## 実装イメージ

### 管理者ログイン (Server Action / API)

```typescript
import { auth } from "firebase-admin";
import { ResultAsync, ok, err } from "neverthrow";

export class AdminAuthService {
  // パスワード検証とカスタムクレーム付与
  async loginAdmin(
    idToken: string,
    passwordInput: string
  ): Promise<Result<void, DomainError>> {
    // 1. IDトークンの検証
    const decodedToken = await auth().verifyIdToken(idToken);
    const uid = decodedToken.uid;

    // 2. パスワード検証 (環境変数のハッシュと比較)
    // 実際にはソルト付きハッシュなどで検証
    if (!this.verifyPassword(passwordInput)) {
      return err(DomainError.unauthorized("パスワードが間違っています"));
    }

    // 3. カスタムクレーム付与
    await auth().setCustomUserClaims(uid, { admin: true });

    return ok(undefined);
  }

  private verifyPassword(input: string): boolean {
    // 環境変数のADMIN_PASSWORD_HASHと比較
    return (
      createHash("sha256").update(input).digest("hex") ===
      process.env.ADMIN_PASSWORD_HASH
    );
  }
}
```

### クライアントサイドでの管理者判定

```typescript
// フック例
const useAdmin = () => {
  const { user } = useAuth(); // Firebase Auth hook
  const [isAdmin, setIsAdmin] = useState(false);

  useEffect(() => {
    if (user) {
      user.getIdTokenResult().then((idTokenResult) => {
        setIsAdmin(!!idTokenResult.claims.admin);
      });
    }
  }, [user]);

  return isAdmin;
};
```

## セキュリティ対策

### Firestore セキュリティルール

`specs/10-database-schema.md` 参照。`request.auth.token.admin == true` で管理者権限を判定します。

### Cloud Run ロギング

アプリケーションのログは標準出力（stdout/stderr）に出力することで、Cloud Loggingに自動的に収集されます。

```typescript
// utils/logger.ts
export const logger = {
  info: (message: string, meta?: any) => {
    console.log(JSON.stringify({ severity: "INFO", message, ...meta }));
  },
  error: (message: string, error?: any) => {
    console.error(
      JSON.stringify({
        severity: "ERROR",
        message,
        error: error?.message,
        stack: error?.stack,
      })
    );
  },
};
```
