from node:24

workdir /app

copy . .

run npm install --legacy-peer-deps

expose 5173

CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0", "--port", "5173"]
