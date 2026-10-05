FROM node:20-alpine
# hadolint ignore=DL3018
RUN apk add --no-cache openssl \
 && addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --chown=appuser:appgroup package.json package-lock.json ./
RUN npm ci
COPY --chown=appuser:appgroup . .
RUN npx prisma generate
USER appuser
EXPOSE 4000
CMD ["node", "server.js"]