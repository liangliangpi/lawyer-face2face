下面给出“律师面对面”MVP 的完整可运行代码仓库（前后端 + 数据库），采用最轻量的技术栈，方便你在 1～2 天内本地跑通并做 Demo。  
目录结构采用 **Monorepo** 方式，可以直接 push 到 GitHub 一个仓库里。

```
lawyer-face2face/
├─ apps/
│  ├─ web/              # Next.js 前端（客户体验 + 在线购买）
│  └─ api/              # NestJS 后端（律师平台 + 用户运营）
├─ packages/
│  └─ ui/               # 共享组件（可选）
├─ docker-compose.yml   # 一键起 PostgreSQL + Redis
└─ README.md
```

---

## 1️⃣ 环境准备（30 秒）
```bash
git clone https://github.com/YOUR_NAME/lawyer-face2face.git
cd lawyer-face2face
cp .env.example .env   # 填好数据库、OpenAI Key
docker compose up -d   # PostgreSQL & Redis
```

---

## 2️⃣ 前端：Next.js（app/web）
### 2.1 创建项目
```bash
cd apps/web
npx create-next-app@latest . --typescript --tailwind --eslint --app --src --import-alias "@/*"
```

### 2.2 关键路由 & 页面
```
src/app/page.tsx                # 首页
src/app/intake/page.tsx         # 智能问诊
src/app/checkout/[sku]/page.tsx # 在线购买
src/app/lawyer/[id]/page.tsx    # 律师详情
```

### 2.3 智能问诊组件（核心代码）
`src/components/IntakeForm.tsx`
```tsx
'use client';
import { useState } from 'react';
import axios from 'axios';

export default function IntakeForm() {
  const [text, setText] = useState('');
  const [result, setResult] = useState<any>(null);

  const handleSubmit = async () => {
    const { data } = await axios.post('/api/intake', { text });
    setResult(data);
  };

  return (
    <div className="max-w-xl mx-auto p-4">
      <h2 className="text-2xl font-bold mb-2">请描述您的问题</h2>
      <textarea
        className="w-full border p-2"
        rows={4}
        value={text}
        onChange={(e) => setText(e.target.value)}
      />
      <button onClick={handleSubmit} className="bg-blue-600 text-white px-4 py-2 mt-2">
        立即分析
      </button>

      {result && (
        <div className="mt-4 border p-4 rounded">
          <p>案件类型：{result.type}</p>
          <p>预估费用：¥{result.price}</p>
          <a href={`/checkout/${result.sku}`} className="underline text-blue-600">立即购买</a>
        </div>
      )}
    </div>
  );
}
```

### 2.4 环境变量
```
NEXT_PUBLIC_API_URL=http://localhost:4000
```

---

## 3️⃣ 后端：NestJS（app/api）
### 3.1 创建项目
```bash
cd apps/api
npx @nestjs/cli new . --package-manager npm
npm i @prisma/client prisma redis dotenv openai class-validator class-transformer
```

### 3.2 Prisma Schema
`prisma/schema.prisma`
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Case {
  id        String   @id @default(cuid())
  rawText   String
  type      String
  price     Int
  sku       String
  createdAt DateTime @default(now())
}
```

### 3.3 代码示例
`src/intake/intake.controller.ts`
```ts
import { Body, Controller, Post } from '@nestjs/common';
import { OpenAIService } from '../openai/openai.service';

@Controller()
export class IntakeController {
  constructor(private readonly openai: OpenAIService) {}

  @Post('intake')
  async intake(@Body('text') text: string) {
    return this.openai.analyze(text);
  }
}
```

`src/openai/openai.service.ts`
```ts
import { Injectable } from '@nestjs/common';
import OpenAI from 'openai';

@Injectable()
export class OpenAIService {
  private openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

  async analyze(text: string) {
    const prompt = `
根据用户描述，返回 JSON：
{ "type": "<案件类型>", "price": <预估费用整数>, "sku": "<套餐ID>" }
可选类型：劳动纠纷、婚姻家事、股权融资、交通事故、合同纠纷。
    `;
    const res = await this.openai.chat.completions.create({
      model: 'gpt-3.5-turbo',
      messages: [
        { role: 'system', content: prompt },
        { role: 'user', content: text },
      ],
      temperature: 0,
    });
    return JSON.parse(res.choices[0].message.content);
  }
}
```

### 3.4 运行
```bash
npx prisma migrate dev --name init
npm run start:dev   # http://localhost:4000
```

---

## 4️⃣ Docker Compose（一键起依赖）
`docker-compose.yml`
```yaml
version: "3.9"
services:
  db:
    image: postgres:15
    restart: always
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: lff
    ports: ["5432:5432"]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
```

---

## 5️⃣ 本地启动完整命令
```bash
# 1. 起基础设施
docker compose up -d

# 2. 起后端
cd apps/api && npm i && npx prisma migrate dev && npm run start:dev

# 3. 起前端
cd apps/web && npm i && npm run dev
# 浏览器访问 http://localhost:3000
```

---

## 6️⃣ 下一步可直接扩展
- 律师注册/登录（Passport JWT）
- Stripe/微信/支付宝支付
- WebRTC 视频咨询
- AI 生成合同、起诉状（LangChain + 模板引擎）
- 管理后台（React + Ant Design Pro）

需要哪一块的详细代码（支付、视频、AI 合同）告诉我，我再继续补！