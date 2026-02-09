FROM alpine:3.19
RUN apk add --no-cache keepalived curl
COPY keepalived.conf /etc/keepalived/keepalived.conf
CMD ["keepalived", "--dont-fork", "--log-console", "-D"]
