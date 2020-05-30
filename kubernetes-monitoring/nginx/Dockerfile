FROM nginx:stable-alpine

RUN addgroup -g 1000 app \
    && adduser -u 1000 -G app -s /bin/sh -D app

COPY nginx.conf /etc/nginx/nginx.conf

USER 1000

EXPOSE 8000