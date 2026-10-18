# Stage 1- Builder stage
FROM node:24 AS builder

WORKDIR /app

COPY . .

RUN npm install --legacy-peer-deps

RUN npm run build

# Stage 2- Runner stage
FROM node:24-alpine AS runner

WORKDIR /app

COPY package.json .
COPY vite.config.js .

RUN npm install -g vite

COPY --from=builder /app/dist ./dist
Copy --from=builder /app/node_modules ./node_modules

CMD ["npm", "run", "preview", "--", "--host", "0.0.0.0", "--port", "4173"]
