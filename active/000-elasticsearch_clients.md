- Start Date: (2024-02-02)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Discussion Issue: (GitHub issue this was discussed in before the RFC, if any)
- Implementation PR(s): (leave this empty)

# ELASTICSEARCH CLIENTS<RFC title>

## Summary

> Currently, the configuration of GMS for ES clients only supports single node configuration.We hope to support multiple ES client node configurations.

## Basic example

> Add configuration for `elasticsearch.addrs` to configure Elasticsearch clients, and maintain compatibility with the original configuration when implementing the code.Modify the code file RestHighLevelClientFactory:
>
> ```
> ...
> @Value("${elasticsearch.addrs}")
> private String addrs;
> ...
> @Nonnull
>   private RestClientBuilder createBuilder(String scheme) {
>     final RestClientBuilder builder;
>     if(StringUtils.isEmpty(addrs)) {
>       builder = RestClient.builder(new HttpHost(host, port, scheme));
>     } else {
>       List<HttpHost> hosts = Arrays.stream(addrs.split(",")).map(m->{
>         String[] arrayStr = m.split(":");
>         if(arrayStr.length==1) {
>           return new HttpHost(arrayStr[0], 9200, scheme);
>         } else if(arrayStr.length==2) {
>           return new HttpHost(arrayStr[0], Integer.parseInt(arrayStr[1]), scheme);
>         }
>         return null;
>       }).filter(f->f!=null).collect(Collectors.toList());
>       builder = RestClient.builder(hosts.toArray(new HttpHost[hosts.size()]));
>     }
> 
> 
>     if (!StringUtils.isEmpty(pathPrefix)) {
>       builder.setPathPrefix(pathPrefix);
>     }
> 
>     builder.setRequestConfigCallback(
>         requestConfigBuilder -> requestConfigBuilder.setConnectionRequestTimeout(connectionRequestTimeout));
> 
>     return builder;
>   }
> ```
>
> 

## Motivation

> When deploying an ES cluster, it is hoped that the datahub can connect to multiple ES clients, ensuring that the datahub can still function even after certain ES nodes are down.

## Detailed design

> - Add configuration elasticsearch.addrs and environment variable ELASTICSEARCH_ ADDRS, default to null. If addrs is not configured, the original ELASTICSEARCH_HOST and ELASTICSEARCH_PORT will be used, otherwise addrs is the main one, and the configuration format is "ip1: port1, ip2: port2"
>
> - Modify class com. linkedin. gms. factory. common. RestHighLevelClientFactory with the following code:
>
>   ```
>   ...
>   @Value("${elasticsearch.addrs}")
>   private String addrs;
>   ...
>   @Nonnull
>   private RestClientBuilder createBuilder(String scheme) {
>    final RestClientBuilder builder;
>    if(StringUtils.isEmpty(addrs)) {
>      builder = RestClient.builder(new HttpHost(host, port, scheme));
>    } else {
>      List<HttpHost> hosts = Arrays.stream(addrs.split(",")).map(m->{
>        String[] arrayStr = m.split(":");
>        if(arrayStr.length==1) {
>          return new HttpHost(arrayStr[0], 9200, scheme);
>        } else if(arrayStr.length==2) {
>          return new HttpHost(arrayStr[0], Integer.parseInt(arrayStr[1]), scheme);
>        }
>        return null;
>      }).filter(f->f!=null).collect(Collectors.toList());
>      builder = RestClient.builder(hosts.toArray(new HttpHost[hosts.size()]));
>    }
>   
>   
>    if (!StringUtils.isEmpty(pathPrefix)) {
>      builder.setPathPrefix(pathPrefix);
>    }
>   
>    builder.setRequestConfigCallback(
>        requestConfigBuilder -> requestConfigBuilder.setConnectionRequestTimeout(connectionRequestTimeout));
>   
>    return builder;
>   }
>   ```
>
>   
>
> - Modify file docker/datahub-gms/start.sh with the following code:
>
>   ```
>   ......
>   if [[ $ELASTICSEARCH_USE_SSL == true ]]; then
>     ELASTICSEARCH_PROTOCOL=https
>   else
>     ELASTICSEARCH_PROTOCOL=http
>   fi
>   
>   if [ -z "$ELASTICSEARCH_ADDRS" ]; then
>       ELASTICSEARCH_URL="$ELASTICSEARCH_PROTOCOL://$ELASTICSEARCH_HOST:$ELASTICSEARCH_PORT"
>   else
>       ELASTICSEARCH_TEMP=$(echo $ELASTICSEARCH_ADDRS | awk -F',' '{print $1}')
>       ELASTICSEARCH_URL="$ELASTICSEARCH_PROTOCOL://$ELASTICSEARCH_TEMP"
>   fi
>   
>   WAIT_FOR_EBEAN=""
>   if [[ $SKIP_EBEAN_CHECK != true ]]; then
>     if [[ $ENTITY_SERVICE_IMPL == ebean ]] || [[ -z $ENTITY_SERVICE_IMPL ]]; then
>       WAIT_FOR_EBEAN=" -wait tcp://$EBEAN_DATASOURCE_HOST "
>     fi
>   fi
>   ......
>   if [[ $SKIP_ELASTICSEARCH_CHECK != true ]]; then
>     exec dockerize \
>       -wait $ELASTICSEARCH_URL -wait-http-header "$ELASTICSEARCH_AUTH_HEADER" \
>       $COMMON
>   else
>     exec dockerize $COMMON
>   fi
>   ```
>
> - Modify file docker/datahub-mae-consumer/start.sh with the following code:
>
>   ```
>   ......
>   
>   if [[ ${ELASTICSEARCH_USE_SSL:-false} == true ]]; then
>       ELASTICSEARCH_PROTOCOL=https
>   else
>       ELASTICSEARCH_PROTOCOL=http
>   fi
>   
>   if [ -z "$ELASTICSEARCH_ADDRS" ]; then
>       ELASTICSEARCH_URL="$ELASTICSEARCH_PROTOCOL://$ELASTICSEARCH_HOST:$ELASTICSEARCH_PORT"
>   else
>       ELASTICSEARCH_TEMP=$(echo $ELASTICSEARCH_ADDRS | awk -F',' '{print $1}')
>       ELASTICSEARCH_URL="$ELASTICSEARCH_PROTOCOL://$ELASTICSEARCH_TEMP"
>   fi
>   
>   dockerize_args=("-timeout" "240s")
>   if [[ ${SKIP_KAFKA_CHECK:-false} != true ]]; then
>     IFS=',' read -ra KAFKAS <<< "$KAFKA_BOOTSTRAP_SERVER"
>     for i in "${KAFKAS[@]}"; do
>       dockerize_args+=("-wait" "tcp://$i")
>     done
>   fi
>   if [[ ${SKIP_ELASTICSEARCH_CHECK:-false} != true ]]; then
>     dockerize_args+=("-wait" "$ELASTICSEARCH_URL" "-wait-http-header" "$ELASTICSEARCH_AUTH_HEADER")
>   fi
>   ......
>   
>   ```
>
> - Modify file docker/elasticsearch-setup/Dockerfile with the following code:
>
>   ```
>   ......
>   FROM ${APP_ENV}-install AS final
>   CMD if [ "$ELASTICSEARCH_USE_SSL" == "true" ]; then ELASTICSEARCH_PROTOCOL=https; else ELASTICSEARCH_PROTOCOL=http; fi \
>       && if [[ -n "$ELASTICSEARCH_USERNAME" ]]; then ELASTICSEARCH_HTTP_HEADERS="Authorization: Basic $(echo -ne "$ELASTICSEARCH_USERNAME:$ELASTICSEARCH_PASSWORD" | base64)"; else ELASTICSEARCH_HTTP_HEADERS="Accept: */*"; fi \
>       && if [ -z "$ELASTICSEARCH_ADDRS" ]; then \
>           ELASTICSEARCH_URL="$ELASTICSEARCH_PROTOCOL://$ELASTICSEARCH_HOST:$ELASTICSEARCH_PORT"; \
>       else \
>           ELASTICSEARCH_TEMP=$(echo $ELASTICSEARCH_ADDRS | awk -F',' '{print $1}'); \
>           ELASTICSEARCH_URL="$ELASTICSEARCH_PROTOCOL://$ELASTICSEARCH_TEMP"; \
>       fi \
>       && if [[ "$SKIP_ELASTICSEARCH_CHECK" != "true" ]]; then \
>           dockerize -wait $ELASTICSEARCH_URL -wait-http-header "${ELASTICSEARCH_HTTP_HEADERS}" -timeout 120s /create-indices.sh; \
>       else /create-indices.sh; fi
>   ```
>
> - Modify file docker/elasticsearch-setup/create-indices.sh with the following code:
>
>   ```
>   ......
>   if [[ $ELASTICSEARCH_USE_SSL == true ]]; then
>       ELASTICSEARCH_PROTOCOL=https
>   else
>       ELASTICSEARCH_PROTOCOL=http
>   fi
>   echo -e "going to use protocol: $ELASTICSEARCH_PROTOCOL"
>   
>   # Elasticsearch URL to be suffixed with a resource address
>   if [ -z "$ELASTICSEARCH_ADDRS" ]; then
>       ELASTICSEARCH_URL="$ELASTICSEARCH_PROTOCOL://$ELASTICSEARCH_HOST:$ELASTICSEARCH_PORT"
>   else
>       ELASTICSEARCH_TEMP=$(echo $ELASTICSEARCH_ADDRS | awk -F',' '{print $1}')
>       ELASTICSEARCH_URL="$ELASTICSEARCH_PROTOCOL://$ELASTICSEARCH_TEMP"
>   fi
>   
>   # set auth header if none is given
>   if [[ -z $ELASTICSEARCH_AUTH_HEADER ]]; then
>     if [[ ! -z $ELASTICSEARCH_USERNAME ]]; then
>       # no auth header given, but username is defined -> use it to create the auth header
>       AUTH_TOKEN=$(echo -ne "$ELASTICSEARCH_USERNAME:$ELASTICSEARCH_PASSWORD" | base64 --wrap 0)
>       ELASTICSEARCH_AUTH_HEADER="Authorization:Basic $AUTH_TOKEN"
>       echo -e "going to use elastic headers based on username and password"
>     else
>       # no auth header or username given -> use default auth header
>       ELASTICSEARCH_AUTH_HEADER="Accept: */*"
>       echo -e "going to use default elastic headers"
>     fi
>   fi
>   ......
>   ```
>
>   

## How we teach this

> We should update user guides to educate users for:
>
> - ​	use env ELASTICSEARCH_ADDRS if you want to config muti es clients
