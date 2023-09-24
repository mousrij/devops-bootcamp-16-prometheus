FROM openjdk:8-jdk-alpine
EXPOSE 8080 8081
RUN mkdir /opt/app
COPY build/libs/bootcamp-java-mysql-monitoring-1.0.jar /opt/app
WORKDIR /opt/app
CMD ["java", "-jar", "bootcamp-java-mysql-monitoring-1.0.jar"]
