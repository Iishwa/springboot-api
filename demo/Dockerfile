COPY demo/pom.xml .
RUN mvn dependency:go-offline -B

# Copy source code and build JAR
COPY demo/src ./src
RUN mvn clean package -DskipTests

# ---- Stage 2: Runtime ----
FROM eclipse-temurin:17-jre
WORKDIR /app

# Add a non-root user for security
RUN useradd -m springuser
USER springuser

# Copy JAR from build stage
COPY --from=build /app/target/demo-0.0.1-SNAPSHOT.jar app.jar

# Expose application port
EXPOSE 8080

# Optional healthcheck (remove if you don’t use Actuator)
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# Run the app
ENTRYPOINT ["java","-jar","app.jar"]
