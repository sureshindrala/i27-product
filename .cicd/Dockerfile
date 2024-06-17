# Need to create a Dockerfile for java based applications
# Dockerfile for Eureka

FROM openjdk:18.0.2.1-jdk
ARG JAR_SOURCE
ENV JAR_SOURCE ${JAR_SOURCE}
ARG JAR_DEST
ENV JAR_DEST ${JAR_DEST}
RUN mkdir -p /opt/sureshindrala1/
WORKDIR /opt/sureshindrala1/
COPY ["${JAR_SOURCE}", "/opt/i27/i27cart-product.jar"]
RUN chmod 777 /opt/sureshindrala1/
EXPOSE 8132
CMD ["java", "-jar", "/opt/i27/i27cart-product.jar"]