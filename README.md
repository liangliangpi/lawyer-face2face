# Lawyer Face2Face MVP

一键启动：

```bash
docker compose up -d
cd apps/api && npm i && npx prisma migrate dev && npm run start:dev
cd ../web && npm i && npm run dev