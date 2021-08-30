FROM nginx:latest

RUN mkdir /app

COPY homework.html /app

RUN touch /var/run/nginx.pid

RUN chown -R 1001 /app /etc/nginx /var/cache/nginx /var/log/nginx /var/run/nginx.pid

USER 1001

COPY default.conf /etc/nginx/conf.d/default.conf

CMD ["nginx", "-g", "daemon off;"]

EXPOSE 8000