# ── Stage 1: Build ──────────────────────────────────────────────────────────
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /workspace

COPY gradlew .
COPY gradle/ gradle/
COPY build.gradle.kts settings.gradle.kts ./
COPY services/circleguard-notification-service/build.gradle.kts \
     services/circleguard-notification-service/build.gradle.kts

RUN chmod +x gradlew && \
    ./gradlew :services:circleguard-notification-service:dependencies --no-daemon -q 2>/dev/null || true

COPY services/circleguard-notification-service/src/ services/circleguard-notification-service/src/
RUN ./gradlew :services:circleguard-notification-service:bootJar --no-daemon -x test

# ── Stage 2: Runtime ─────────────────────────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

RUN addgroup -S circleguard && adduser -S circleguard -G circleguard
USER circleguard

COPY --from=builder /workspace/services/circleguard-notification-service/build/libs/*.jar app.jar

EXPOSE 8082
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
