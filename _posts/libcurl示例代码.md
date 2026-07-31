---
title: libcurl示例代码
date: 2024-11-29
tags: 
  - C
  - C/C++
---
~~~c
#include <stdio.h>
#include <stdlib.h>
#include <curl/curl.h>

typedef struct response Response;
struct response {
    char* data;
    size_t size;
};

size_t write_callback(char *ptr, size_t size, size_t nmemb, void *userdata) {
    size_t total_size = size * nmemb;

    Response* response = (Response*)userdata;
    char* p = realloc(response->data, response->size + total_size + 1);
    if (p == NULL)
        return CURL_WRITEFUNC_ERROR;
    else 
        response->data = p;
    
    memcpy(&(response->data[response->size]), ptr, total_size);
    response->size += total_size;
    response->data[response->size] = '\0';

    return total_size;
}

int main() {
    CURL* curl;
    CURLcode code;

    Response response;
    response.data = (byte*)malloc(1);
    response.size = 0;

    curl = curl_easy_init();
    if (curl == NULL) {
        fprintf(stderr, "Curl Init Failed\n");
        return -1;
    }

    curl_easy_setopt(curl, CURLOPT_URL, "http://www.httpbin.org/get?hello=world");
    curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, write_callback);
    curl_easy_setopt(curl, CURLOPT_WRITEDATA, &response);

    code = curl_easy_perform(curl);
    if (code != CURLE_OK) {
        fprintf(stderr, "Curl Perform Error: %s", curl_easy_strerror(code));
        return -1;
    }
    printf("%sResponse-Size: %d\n", response.data, response.size);
    free(response.data);

    char url[] = "Hello World %%#*";
    char* encoded_url = curl_easy_escape(curl, url, 0);
    if (encoded_url == NULL) {
        printf("URL Encode Fail\n");
    }
    printf("url: %s\n", url);
    printf("encoded_url: %s", encoded_url);

    curl_free(encoded_url);
    curl_easy_cleanup(curl);
    return 0;
}
~~~

