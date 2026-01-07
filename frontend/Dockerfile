FROM nginx:alpine
COPY . /usr/share/nginx/html
# Setup reverse proxy so Frontend can talk to Backend on /api
RUN echo 'server { listen 80; location / { root /usr/share/nginx/html; index index.html; } location /api/ { proxy_pass http://backend:5000/api/; } }' > /etc/nginx/conf.d/default.conf
