FROM 974979602211.dkr.ecr.us-east-1.amazonaws.com/nginxplus:v1
RUN rm /etc/nginx/nginx.conf /etc/nginx/conf.d/default.conf
COPY conf/* /etc/nginx/

RUN mkdir api_conf.d
COPY conf/api_conf.d/att_api.conf /etc/nginx/api_conf.d/att_api.conf

