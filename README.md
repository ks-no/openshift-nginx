# NGINX image for Openshift

based on the official nginx image with the following modifications to be able to run on openshift:
(see https://torstenwalter.de/openshift/nginx/2017/08/04/nginx-on-openshift.html)

- listens on port 8081
- directories /var/cache/nginx, /var/run and /var/log/nginx are writeable for root group
- removes the user directive from /etc/nginx/nginx.conf

# Using the image

New releases are published to Github packages under the name `docker.pkg.github.com/ks-no/openshift-nginx/fiks-nginx-openshift:VERSION`
