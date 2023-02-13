![image](https://user-images.githubusercontent.com/1689101/213045711-7fb9fa4f-77f4-423d-8fae-ed4f6c824e0f.png)

# Krakend Tutorial
KrakenD API - example 

## Pre requirements:
To test this project run the following commands, you require docker installed in your computer. 


## Build docker 
  
```bash
  // Change ENV to enviroment (PROD | STG | DEV)
  docker build -f docker/Dockerfile . -t krakend-example:latest --build-arg ENV=dev
```

## Run docker 

```bash
  docker run -d -it -p 80:8080 --name=krakend-example krakend-example
```

## Run krakend locally with docker-compose (watch active)

```bash
  cd docker && docker-compose up -d krakend
```
## Examples

### 1. Health check to krakend
```json
// conf/krakend.json
{
  "endpoint": "/health",
  "input_headers": ["*"],
  "backend": [
    {
      "url_pattern": "/__health",
      "host": ["http://localhost:8080"]
    }
  ]
},
```

### 2. Health check to krakend with flexible config 
**https://www.krakend.io/docs/configuration/flexible-config/**

##### configuration required when building docker
```dockerfile
RUN FC_ENABLE=1 \
  FC_OUT=/tmp/krakend.json \
  FC_PARTIALS="/etc/krakend/partials" \
  FC_SETTINGS="/etc/krakend/settings/$ENV" \
  FC_TEMPLATES="/etc/krakend/templates" \
  krakend check -d -t -c krakend.json
``` 


##### config file services

```json
// conf/settings/dev|stg|prod/services.json
{
  "localhost": "http://localhost:8080",
  "json_placeholder": "https://jsonplaceholder.typicode.com"
}
```

##### config endpoint krakend.json

```json
// conf/krakend.json
{
  "endpoint": "/health",
  "input_headers": ["*"],
  "backend": [
    {
      "url_pattern": "/__health",
      "host": ["{{ .services.localhost }}"]
    }
  ]
},
```

### 3. Proxy
##### Proxy from ```/api/posts/``` to https://jsonplaceholder.typicode.com/posts, return a collection
```json
{
  "@comments": "Proxy from /api/posts/ to https://jsonplaceholder.typicode.com/posts",
  "endpoint": "/api/posts/",
  "method": "GET",
  "output_encoding": "json-collection",
  "backend": [
    {
      "url_pattern": "/posts",
      "encoding": "safejson",
      "method": "GET",
      "disable_host_sanitize": false,
      "host": ["{{ .services.json_placeholder }}"], 
      "is_collection": true
    }
  ],
  "extra_config": {}
}
```

### 3. Proxy with query params
##### Proxy from ```/api/posts/{id}``` to https://jsonplaceholder.typicode.com/posts/{id}, return a object
```json
{
      "@comments":"Proxy with query params",
      "endpoint": "/api/posts/{id}",
      "method": "GET",
      "output_encoding": "json",
      "backend": [
        {
          "url_pattern": "/posts/{id}",
          "encoding": "json",
          "method": "GET",
          "disable_host_sanitize": false,
          "host": [
            "https://jsonplaceholder.typicode.com/"
          ]
        }
      ],
      "input_query_strings":[
        "id"
      ]
    }
```

### 4. Response manipulation - merge two response backend
Doc: **https://www.krakend.io/docs/endpoints/response-manipulation/**

```json
{
  "@comments":"This endpoint merge two responses post + comment.",
  "endpoint": "/api/merge/{postId}/{commentId}",
  "method": "GET",
  "output_encoding": "json",
  "backend": [
    {
      "url_pattern": "/comments/{commentId}",
      "encoding": "json",
      "method": "GET",
      "host": [
        "https://jsonplaceholder.typicode.com/"
      ],
      "group": "comments"
    },
    {
      "url_pattern": "/posts/{postId}",
      "encoding": "json",
      "method": "GET",
      "host": [
        "https://jsonplaceholder.typicode.com/"
      ]
    }
  ]
}
```

### 5. Response manipulation - extract properties of type object 
Doc: **https://www.krakend.io/docs/endpoints/response-manipulation/**

```json
{
  "@comments":"This endpoint get the address response attribute data.",
  "endpoint": "/api/users/target",
  "method": "GET",
  "output_encoding": "json",
  "backend": [
    {
      "url_pattern": "/users/2",
      "encoding": "json",
      "method": "GET",
      "host": [
        "https://jsonplaceholder.typicode.com/"
      ],
      "target": "address"
    }
  ]
}
```

### 6. Request post 
```json
{
      "@comments":"This endpoint create post.",
      "endpoint": "/api/posts",
      "method": "POST",
      "output_encoding": "json",
      "backend": [
        {
          "url_pattern": "/posts",
          "encoding": "json",
          "method": "POST",
          "host": [
            "{{ .services.json_placeholder }}"
          ]
        }
      ]
    },
```

### 7. Request get to post 

```json
    {
      "@comments":"Proxy endpoint get to post.",
      "endpoint": "/api/posts/create",
      "method": "GET",
      "output_encoding": "json",
      "backend": [
        {
          "url_pattern": "/posts",
          "encoding": "json",
          "method": "POST",
          "host": [
            "{{ .services.json_placeholder }}"
          ]
        }
      ]
    }
```

example curl: 
```curl
curl --location --request GET 'http://localhost:8080/api/posts/create?title=primer post&body=hola&userId=1' \
--header 'Content-Type: application/json' \
--data-raw '{
    "title": "foo",
    "body": "bar",
    "userId": 1
}'
```

### 5. Setup rate limit and cache
Doc: 
- **https://www.krakend.io/docs/enterprise/service-settings/service-rate-limit/#service-rate-limiting-max_rate**
- **https://www.krakend.io/docs/backends/caching/#cache-configuration**

```json
{
  "@comments":"This endpoint use internal cache and set rate limit",
  "endpoint": "/api/users/{userId}/cache",
  "method": "GET",
  "output_encoding": "json",
  "backend": [
    {
      "url_pattern": "/users/{userId}",
      "encoding": "json",
      "method": "GET",
      "host": [
        "{{ .services.json_placeholder }}"
      ],
      "extra_config":{
        "qos/http-cache":{
          "shared":true
        }
      }
    }
  ],
  "extra_config":{
    "qos/ratelimit/router": {
      "max_rate": 2,
      "client_max_rate": 1,
      "strategy": "ip"
    }
  }
}
```