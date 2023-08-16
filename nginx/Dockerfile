FROM nginx:1.25.2-alpine

# Remove default welcome page
RUN rm /usr/share/nginx/html/index.html

# 1. support running as arbitrary user which belogs to the root group
# 2. users are not allowed to listen on priviliged ports
# 3. comment user directive as master process is run as user in OpenShift anyhow
RUN chmod g+rwx /var/cache/nginx /var/run /var/log/nginx && \
    chgrp -R root /var/cache/nginx && \
    sed -i.bak 's/listen\(.*\)80;/listen 8081;/' /etc/nginx/conf.d/default.conf && \
    sed -i.bak 's/^user/#user/' /etc/nginx/nginx.conf && \
    addgroup nginx root

EXPOSE 8081

USER nginx
