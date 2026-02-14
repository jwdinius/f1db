FROM eclipse-temurin:21-jdk-jammy AS devcontainer

SHELL ["/bin/bash", "-c"]
ENV DEBIAN_FRONTEND=noninteractive

# 1. Install Docker CLI
RUN apt-get update && apt-get install -y ca-certificates curl git gnupg lsb-release sudo && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
    chmod a+r /etc/apt/keyrings/docker.gpg && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
    tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    apt-get update && apt-get install -y docker-ce-cli

# setup user env at the end
# -m option creates a fake writable home folder
RUN adduser --disabled-password --gecos '' f1db
RUN adduser f1db sudo
RUN echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

# 2. Create the docker group if it doesn't exist and add your user
# Replace 'databaseuser' with the name of the user you are using
RUN groupadd docker || true && \
    usermod -aG docker f1db

FROM eclipse-temurin:21-jdk-jammy AS builder
WORKDIR /app

# 1. Copy only the Gradle wrapper and configuration files
COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts settings.gradle.kts gradle.properties ./

# 2. Download dependencies. 
# We run 'dependencies' or a dry-run build to populate the cache.
# The '|| true' ensures the build continues even if this specific task doesn't exist.
RUN chmod +x gradlew
RUN ./gradlew build -x test --no-daemon > /dev/null 2>&1 || true

# 3. NOW copy the source code. 
# Only changes to /src will trigger the layers below this point.
COPY build-logic build-logic
COPY src src

ENV GRADLE_USER_HOME=/app/.gradle-cache

# Now the cache mount points to that specific path
RUN --mount=type=cache,target=/app/.gradle-cache \
    ./gradlew build -x test

# --- Stage 2: Run ---
FROM eclipse-temurin:21-jre-jammy
ARG VERSION
WORKDIR /app

# Create a non-root user for security
RUN useradd -m f1db
USER f1db

# Copy the jar from the build stage
# Note: Replace 'app.jar' with your actual jar name found in build/libs/
COPY --from=builder /app/build/libs/f1db-$VERSION.jar f1db.jar

# Expose the database port (e.g., 5432, 8080, etc. - adjust as needed)
EXPOSE 8080

# Run the application
ENTRYPOINT ["java", "-jar", "f1db.jar"]