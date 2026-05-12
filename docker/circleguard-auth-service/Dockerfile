# ── Stage 1: Build ──────────────────────────────────────────────────────────
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /workspace

# Copy wrapper and root build descriptors first (layer-cached until deps change)
COPY gradlew .
COPY gradle/ gradle/
COPY build.gradle.kts settings.gradle.kts ./
COPY services/circleguard-auth-service/build.gradle.kts \
     services/circleguard-auth-service/build.gradle.kts

RUN chmod +x gradlew && \
    ./gradlew :services:circleguard-auth-service:dependencies --no-daemon -q 2>/dev/null || true

# Copy source and build the fat JAR
COPY services/circleguard-auth-service/src/ services/circleguard-auth-service/src/
RUN ./gradlew :services:circleguard-auth-service:bootJar --no-daemon -x test

# ── Stage 2: Runtime ─────────────────────────────────────────────────────────
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

RUN addgroup -S circleguard && adduser -S circleguard -G circleguard
USER circleguard

COPY --from=builder /workspace/services/circleguard-auth-service/build/libs/*.jar app.jar

EXPOSE 8180
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
