FROM node:8-alpine

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm install --production

COPY . .

ENV TODO_API_PORT=8082
EXPOSE 8082

# server.js directo en vez de "npm start" (el script usa nodemon, que es devDependency)
CMD ["node", "server.js"]
