# 11. データベーススキーマ

## 原則：正規化とパフォーマンスのバランス

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
