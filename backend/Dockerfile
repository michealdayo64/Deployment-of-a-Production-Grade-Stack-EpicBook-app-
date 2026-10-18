# ---------- Stage 1: install production dependencies ----------
FROM node:20-alpine AS deps

# Set the working directory inside the container
WORKDIR /app

# Copy package.json and package-lock.json to the working directory
COPY package*.json ./

# Install dependencies
RUN npm ci --omit=dev

# ---------- Stage 2: runtime image ----------
FROM node:20-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app
RUN apk add --no-cache curl
COPY --from=deps /app/node_modules ./node_modules
COPY . .
USER node
EXPOSE 3001
CMD ["node", "server.js"]
