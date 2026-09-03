FROM nginx:alpine

ARG APP_VERSION=dev
ENV APP_VERSION=${APP_VERSION}

RUN rm /etc/nginx/conf.d/default.conf

COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
