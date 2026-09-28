# Imagen Debian (no alpine) porque node-sass 4.x descarga binario precompilado
# para node 8; en alpine tocaria compilarlo desde fuente.
FROM node:8

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm install

COPY . .

ENV PORT=8080
EXPOSE 8080

# "npm start" levanta el dev-server, que ademas hace de proxy hacia
# auth-api y todos-api. El proxy es obligatorio: el codigo Vue llama a
# /login y /todos como rutas relativas al mismo origen.
CMD ["npm", "start"]
