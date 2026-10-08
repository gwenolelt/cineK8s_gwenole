# -- Etape 1 : build --
# Image de build (Maven + JDK 21)
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app

# Copier le pom.xml SEUL, puis telecharger les dependances (mise en cache)
COPY pom.xml .
RUN mvn -q -B dependency:go-offline

# Copier les sources, puis compiler SANS lancer les tests
COPY src ./src
RUN mvn -q -B package -DskipTests

# -- Etape 2 : image finale (JRE seulement, non-root) --
# Image d'execution (JRE 21, variante alpine)
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Creer un utilisateur systeme "spring" d'UID 10001, puis basculer dessus
RUN addgroup -S spring && adduser -S -u 10001 -G spring spring
USER spring

# Recuperer le jar produit a l'etape "build" et le nommer app.jar
COPY --from=build /app/target/*.jar app.jar

# Port d'ecoute documente (cf. server.port dans application.yaml)
EXPOSE 8080

# Lancer le jar, avec une JVM qui dimensionne son heap a 75 % de la memoire du conteneur
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
