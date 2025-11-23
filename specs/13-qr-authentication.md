# 13. QRコード認証仕様 (Firebase版)

## 概要

Firebase Authenticationの匿名認証（Anonymous Auth）を利用して、QRコードスキャンからシームレスに注文を開始できるフローを実現します。

## 認証フロー

1. **QRコードスキャン**: ユーザーが店舗等のQRコード（URL）をスキャン。
   - URL例: `https://cafeteria.example.com/?source=qr&location=floor1`
2. **LP/メニュー画面表示**: Next.jsアプリが起動。
3. **自動匿名ログイン**:
   - クライアントサイドで `signInAnonymously(auth)` を実行。
   - 既にログイン済み（永続化されたセッションがある）場合はスキップ。
4. **ユーザー情報取得**:
   - ログイン成功後、Firestoreの `users/{uid}` を参照。
   - 過去にプロフィール情報（部署・氏名）が保存されていれば、注文フォームに自動入力。

## 実装詳細

### クライアントサイド認証 (Hooks)

```typescript
// hooks/useAnonymousAuth.ts
import { useEffect, useState } from "react";
import { getAuth, signInAnonymously, onAuthStateChanged, User } from "firebase/auth";
import { doc, getDoc, getFirestore } from "firebase/firestore";

export function useAnonymousAuth() {
  const [user, setUser] = useState<User | null>(null);
  const [profile, setProfile] = useState<UserProfile | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const auth = getAuth();
    
    // 認証状態の監視
    const unsubscribe = onAuthStateChanged(auth, async (currentUser) => {
      if (currentUser) {
        setUser(currentUser);
        // プロファイル取得
        const db = getFirestore();
        const docRef = doc(db, "users", currentUser.uid);
        const docSnap = await getDoc(docRef);
        if (docSnap.exists()) {
          setProfile(docSnap.data() as UserProfile);
        }
      } else {
        // 未ログインなら匿名ログイン実行
        try {
          await signInAnonymously(auth);
        } catch (error) {
          console.error("Anonymous auth failed", error);
        }
      }
      setLoading(false);
    });

    return () => unsubscribe();
  }, []);

  return { user, profile, loading };
}
```

### ユーザープロファイルの保存 (注文時)

注文確定時に、入力されたユーザー情報をFirestoreに保存し、次回以降の入力を省略可能にします。

```typescript
// application/use-cases/CreateOrderUseCase.ts の一部（イメージ）

// 注文保存後...
if (shouldSaveProfile) {
  await firestore().collection("users").doc(userId).set({
    department: input.department,
    name: input.name,
    gender: input.gender,
    ageGroup: input.ageGroup,
    lastAccessedAt: firestore.FieldValue.serverTimestamp()
  }, { merge: true });
}
```

## QRコード生成

管理画面から、場所情報（`location`パラメータ）を含んだURLのQRコードを生成します。
Firebase Dynamic Linksは非推奨のため、単純なURLパラメータを使用します。

URL: `https://<app-id>.web.app/?source=qr&location=cafeteria_entrance`
