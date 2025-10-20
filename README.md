🎯 MỤC TIÊU

Tạo ứng dụng Telegram Bot có khả năng:

Hiển thị giá vàng (Việt Nam, Spot quốc tế).

Cho phép người dùng đăng ký cảnh báo giá.

Lưu dữ liệu vào PostgreSQL (sẵn sàng cho TimescaleDB).

Có hệ thống config & bảo mật .env.

Tự động tạo bảng DB nếu chưa có.

Có kiến trúc module hóa, dễ mở rộng (multi-market sau này).

Mọi truy vấn DB phải prepared / parameterized.

Tất cả các biến nhạy cảm (API keys, DB URL, Telegram Token) phải đọc từ .env.

🧩 KIẾN TRÚC CẦN TẠO
goldwatcher-bot/
├── src/
│   ├── index.ts                 # entrypoint
│   ├── bot/
│   │   ├── commands/
│   │   │   ├── gia_vang.ts
│   │   │   ├── canh_bao.ts
│   │   │   ├── bieudo.ts
│   │   │   └── help.ts
│   │   ├── middleware/
│   │   │   └── rateLimit.ts
│   │   └── utils/
│   │       ├── format.ts
│   │       └── logger.ts
│   ├── jobs/
│   │   └── fetchPrices.ts
│   ├── services/
│   │   ├── metalsApi.ts
│   │   ├── vnPriceService.ts
│   │   └── fxService.ts
│   ├── db/
│   │   ├── connection.ts
│   │   └── schemaInit.ts
│   └── config/
│       └── env.ts
├── .env.example
├── package.json
├── tsconfig.json
└── README.md

⚙️ CÔNG NGHỆ

Node.js 20+

TypeScript

Telegraf (Telegram Bot framework)

Axios (HTTP client)

pg (PostgreSQL client)

dotenv (môi trường)

node-cron (job định kỳ)

winston (logging)

zod (validate env & input)

bcrypt + crypto (nếu có xác thực user sau này)

🧠 YÊU CẦU CHÍNH VỀ LOGIC
1. Khởi tạo bot

Dùng Telegraf, token từ process.env.TELEGRAM_TOKEN.

Có middleware chống spam (rate limit 3 requests/5s mỗi user).

Log mọi interaction.

2. Lệnh /gia_vang

Gọi services/metalsApi.ts để lấy spot XAU/USD.

Tính giá quy đổi sang VND (từ fxService.ts).

Gọi vnPriceService.ts để lấy giá trong nước (demo từ file JSON tĩnh hoặc API test).

Trả về tin nhắn định dạng đẹp (MarkdownV2).

3. Lệnh /canhbao

Người dùng gửi /canhbao 82000000

Bot lưu rule (user_id, threshold_value, created_at) vào bảng alerts.

Cron job fetchPrices.ts chạy mỗi 5 phút, kiểm tra giá và gửi cảnh báo nếu vượt.

4. Database (PostgreSQL)

Tự động khởi tạo các bảng nếu chưa có:

CREATE TABLE IF NOT EXISTS users (
  id SERIAL PRIMARY KEY,
  telegram_id BIGINT UNIQUE,
  username TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS alerts (
  id SERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id),
  threshold NUMERIC(18,2),
  active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS prices (
  id SERIAL PRIMARY KEY,
  market VARCHAR(10),     -- 'VN' | 'XAU'
  price NUMERIC(18,2),
  currency VARCHAR(5),
  updated_at TIMESTAMP DEFAULT NOW()
);


Sử dụng pg.Pool để kết nối, bật SSL nếu process.env.DB_SSL=true.

5. Security

Không log token hay DB credentials.

.env chứa:

TELEGRAM_TOKEN=xxx
METALS_API_KEY=xxx
DATABASE_URL=postgresql://user:pass@host:5432/goldwatcher
DB_SSL=true


Validate bằng Zod trước khi sử dụng bất kỳ biến môi trường nào.

Dùng prepared statements cho mọi truy vấn (không string interpolation).

Chống DDoS: rate-limit theo userId + Redis cache sau này.

6. Logging

winston ghi log dạng JSON (info/warn/error).

Console log đẹp khi chạy dev, file log khi chạy production.

7. Scheduler

node-cron chạy fetchPrices.ts mỗi 5 phút để cập nhật giá.

Dữ liệu được lưu vào prices.

Khi có rule cảnh báo, gửi tin nhắn Telegram.

🧱 CẤU TRÚC CODE MẪU (rút gọn)
src/db/connection.ts
import { Pool } from 'pg';
import { z } from 'zod';
import dotenv from 'dotenv';
dotenv.config();

const EnvSchema = z.object({
  DATABASE_URL: z.string(),
  DB_SSL: z.string().optional()
});
const env = EnvSchema.parse(process.env);

export const pool = new Pool({
  connectionString: env.DATABASE_URL,
  ssl: env.DB_SSL === 'true' ? { rejectUnauthorized: false } : false
});

src/db/schemaInit.ts
import { pool } from './connection';

export async function initSchema() {
  await pool.query(`
    CREATE TABLE IF NOT EXISTS users (
      id SERIAL PRIMARY KEY,
      telegram_id BIGINT UNIQUE,
      username TEXT,
      created_at TIMESTAMP DEFAULT NOW()
    );
  `);
  await pool.query(`
    CREATE TABLE IF NOT EXISTS alerts (
      id SERIAL PRIMARY KEY,
      user_id INT REFERENCES users(id),
      threshold NUMERIC(18,2),
      active BOOLEAN DEFAULT TRUE,
      created_at TIMESTAMP DEFAULT NOW()
    );
  `);
  await pool.query(`
    CREATE TABLE IF NOT EXISTS prices (
      id SERIAL PRIMARY KEY,
      market VARCHAR(10),
      price NUMERIC(18,2),
      currency VARCHAR(5),
      updated_at TIMESTAMP DEFAULT NOW()
    );
  `);
}

src/bot/index.ts
import { Telegraf } from 'telegraf';
import { initSchema } from '../db/schemaInit';
import { handleGiaVang } from './commands/gia_vang';
import rateLimit from './middleware/rateLimit';
import dotenv from 'dotenv';
dotenv.config();

const bot = new Telegraf(process.env.TELEGRAM_TOKEN!);
bot.use(rateLimit);

bot.start(ctx => ctx.reply('Chào mừng bạn đến với GoldWatcher Bot 💰'));
bot.command('gia_vang', handleGiaVang);

(async () => {
  await initSchema();
  console.log('✅ Database schema initialized');
  await bot.launch();
  console.log('🤖 Bot is running...');
})();

🧾 8. Lưu ý bảo mật vận hành

Triển khai bằng Docker (ẩn .env trong secrets).

Không log token hoặc password.

Dùng helmet nếu mở cổng HTTP public.

Giới hạn quyền DB (user riêng chỉ có CRUD trên schema này).

Backup DB định kỳ (7 ngày / lần).

✅ KẾT QUẢ KHI CODEX CHẠY

Sau khi CodeX thực thi prompt này:

Tạo đầy đủ project Node.js + TypeScript.

Cài đặt các package, tạo .env.example.

Tự động khởi tạo DB schema khi chạy lần đầu.

Bot hoạt động với lệnh /gia_vang (hiển thị giá spot và quy đổi).

Cấu trúc code rõ ràng, tách module, đảm bảo an toàn.