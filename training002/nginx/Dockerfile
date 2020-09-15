# Dockerfile for nginx-container

FROM nginx:latest

RUN rm -f /etc/nginx/conf.d/default.conf
COPY ./files/tomcat.conf /etc/nginx/conf.d/