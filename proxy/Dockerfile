# Proxy image inherited from the previous team.

FROM nginx:stable-alpine-slim

RUN apk add --no-cache curl

COPY nginx.conf /etc/nginx/nginx.conf

USER nginx

EXPOSE 8080

CMD ["nginx", "-g", "daemon off;"]
