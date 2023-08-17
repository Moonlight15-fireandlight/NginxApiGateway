# Desafios Nginx
## Desafio2

- Creamos un cluster kubernetes de 3 nodos usando el servicio de Amazon EKS. Se utilizo un archivo en terraform () que genera automaticamente el cluster en una region, vpc, subredes, etc.. predeterminados. Usted puede modificarlos para levantar un cluster con los requisitos que necesite para su desarollo.

- Luego, desplegamos los pods y servicios de las aplicaciones que se observan en el diagrama () y se encuentran en el enlace https://github.com/fchmainy/nginx-aks-demo. Para esto utilizaremos los manifiestos de ``all_attributes.yaml``, ``frontend-namespace-via-apigw.yaml`` y ``generator-direct.yaml``. 

Verifique que los pods desplegados se observen de igual forma: 
(Colocar imagen de los pods adjectivs .. en modod running)


- Desplegamos NGINX+ utilizando el manifiesto ``kustomization.yaml`` dentro de la carpeta nginx-ic. Verfique que los pods de nginx esten correctamente desplegados como se observa en la siguiente imagen.

- En el despliege de nginx como API Gateway se debe configurar una estructura de archivos que se encuentra en el enlace https://www.nginx.com/c/deploy-nginx-as-an-api-gateway/ y que se muestra de la siguiente forma:
(Colocar imagen de nginx como API Gateway)

Como referencia para cada uno los archivos que se necesita desplegar se utilizo el trabajo desarollado en https://galvarado.com.mx/post/desplegar-un-api-gateway-con-nginx/. A continuación, mostraemos los archivos correspondientes que se utilizaron para levantar el API Gateway.

Primero, creamos el archivo de configuracion principal del nginx ``nginx.conf`` donde agregamos las configuracion del gateway con el comando include. 

```
user  nginx;
worker_processes  auto;
error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    #tcp_nopush     on;
    keepalive_timeout  65;
    include /etc/nginx/api_gw.conf; # Configuracion del Api gateway
}
```
Luego, creamos el archivo de configuracion ``api_gw.conf``:

```
log_format jwt '$remote_addr - $remote_user [$time_local] "$request" '
               '$status $body_bytes_sent "$http_referer" "$http_user_agent" '
               '$jwt_header_alg $jwt_claim_uid $jwt_claim_url';

limit_req_zone $remote_addr zone=perip:1m rate=1r/s;

server {
  include api_conf.d/*.conf;
  proxy_intercept_errors on;  
  include api_errors.conf; 
  default_type application/json;
}
```
En el archivo api_conf.d se encuentra lo relacionado al procesamiento de trafico del API gateway que se muestra en la grafica del problema. 

```
location / {

  location /name {
    proxy_pass http://generator.rest-api;
    proxy_set_header Host $host;

    limit_req zone=perip nodelay;
    limit_req_status 429; 
  }

  location = /api/v1 {
    proxy_pass http://generator.rest-api;
    proxy_set_header Host $host;

    limit_req zone=perip nodelay;
    limit_req_status 429; 
  }

  location /api/v1/colors {
    proxy_pass http://colors.rest-api;
    proxy_set_header Host $host;

    limit_req zone=perip nodelay;
    limit_req_status 429;
  }

  return 404;
}
```
Cada una de los "location" explica una directiva de ruta que mediante el proxy_pass redirige el trafico hacia la aplicaciones desplegadas, la direccion root dirige el trafico hacia la aplicacion generator. 

- En la validacion de peticiones mediante JWT existe una referencia propia de nginx https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-jwt-authentication/. Creamos un JSON Web Key File en el archivo api_secre.jwk.

```
{"keys":
    [{
        "k":"ZmFudGFzdGljand0",
        "kty":"oct",
        "kid":"0001"
    },
    {
        "k":"bmdpbnhjaDAy",
        "kty":"oct",
        "kid":"0002"
    }]
}
```
Esta llave generada esta basada en ejemplo que se encuentra en la documentacion oficial de nginx. Si desea saber mas sobre el formato de la llave, visite el siguiente enlace https://www.rfc-editor.org/rfc/rfc7517. Una vez creado la llave, luego declaramos que se genere una peticion para cuando queramos ingresar o hacer un get sobre algunos de los pod. Esto se logra especificando en cada location o ruta del archivo apigateway (api_gw.conf).

```
location /api/v1/adjectives {
  auth_jwt "Adjectives API";
  auth_jwt_key_file api_secret.jwk; 
  proxy_pass http://adjectives.rest-api;
}
location /api/v1/locations {
  auth_jwt "Locations API";
  auth_jwt_key_file api_secret.jwk;
  proxy_pass http://locations.rest-api;
}
```

Se ha agregado las peticiones por token solo en los casos de las aplicaciones adjectives y locations. Entonces, al momento de que el usuario desee ingresar a estas rutas necesitará colocar un JWT token para acceder a la data.

- Para generar un rate limit en las solicitudes, se necesita agregar el comando ``limit_req_zone $remote_addr zone=perip:1m rate=1r/s`` en el archivo del apigateway. Y de igual forma con el JWT agregar a las locaciones de cada pod, los siguientes comandos: 

```
limit_req zone=perip nodelay;
limit_req_status 429;
```

- Nginx te permite modificar o escoger que respuesta, a parte del default de nginx, se desea obtener ante peticiones no autenticadas. Esto se establece con los siguientes comandos ubicados que se encuentran en el archivo api_erros.conf.

```
error_page 400 = @400;
location @400 { return 400 '{"status":400,"message":"Bad request"}\n'; }

error_page 401 = @401;
location @401 { return 401 '{"status":401,"message":"Authorization Required"}\n'; }

error_page 429 = @429;
location @429 { return 429 '{"status":429,"message":"Too Many Requests"}\n'; }

error_page 404 = @404;
location @404 { return 404 '{"status":404,"message":"Resource not found"}\n'; }
```
Luego esto sea agrega en archivo prinicpal del gateway mediante un include con la direccion del archivo. Existes mas respuestas que usted puede agregar al archivo api_erros.conf, reivse la documentacion oficial en la url que se mostro en lineas anteriores. 

- Finalmente, una vez realizada todas las configuraciones necesarias dentro del archivo apigateway procedemos, a construir una imagen en docker con las configuraciones realizadas. Para esto es necesario crear un archivo dockerfile que cree una imagen del archivo con todas las configuraciones realizadas correspondientes al apigateway. 

```
FROM 974979602211.dkr.ecr.us-east-1.amazonaws.com/nginxplus:v1
RUN rm /etc/nginx/nginx.conf /etc/nginx/conf.d/default.conf
COPY conf/* /etc/nginx/

RUN mkdir api_conf.d
COPY conf/api_conf.d/att_api.conf /etc/nginx/api_conf.d/att_api.conf
```




