---
title: spring gateway 二次url encode 问题
tags:
  - spring cloud
categories:
  - spring cloud
toc: false
date: 2020-04-09 16:08:24
---

[spring gateway issues](https://github.com/spring-cloud/spring-cloud-gateway/issues/1394)

spring gateway 转发业务系统请求时遇到了url encoded again的问题，将work around 放到filter order 最后一个

work around：
```java
public class FixUrlEncodingFilter implements GlobalFilter, Ordered {
    public final Map<String,String> blackList= new HashMap<String,String>(){{
        put("\\[", "%5b");
        put("]", "%5d");
    }};
    @Override
    public Mono<Void> filter(final ServerWebExchange exchange, final GatewayFilterChain chain) {
        URI uri = exchange.getRequest().getURI();
        boolean encoded = containsEncodedParts(uri);
        if (!encoded) {
            uri = alreadyEncodedUri(exchange);
        }
        return chain.filter(exchange.mutate().request(exchange.getRequest().mutate().uri(uri).build()).build());
    }

    private URI alreadyEncodedUri(final ServerWebExchange exchange) {
        final MultiValueMap<String, String> escapedParameters = queryParamsEscaped(exchange);
        final URI original = exchange.getRequest().getURI();
        UriComponentsBuilder uriBuilder = UriComponentsBuilder.newInstance()
                .scheme(original.getScheme())
                .host(original.getHost())
                .port(original.getPort())
                .path(original.getPath());

        if (escapedParameters != null && escapedParameters.size() > 0) {
            uriBuilder = uriBuilder.queryParams(escapedParameters);
        }
        return uriBuilder.build(true).toUri();
    }

    private MultiValueMap<String, String> queryParamsEscaped(final ServerWebExchange exchange) {
        final MultiValueMap<String, String> parameters = exchange.getRequest().getQueryParams();
        final MultiValueMap<String, String> escapedParameters = new LinkedMultiValueMap<>();
        if (parameters != null) {
            parameters.entrySet().forEach(entry -> {
                final List<String> escapedValues = new ArrayList<>();
                final String escapedKey = escape(entry.getKey());
                entry.getValue().forEach(value -> {
                    escapedValues.add(escape(value));
                });
                escapedParameters.put(escapedKey, escapedValues);
            });
        }
        return escapedParameters;
    }

    public String escape(final String value) {
        if (value == null || value.isEmpty()) {
            return "";
        }
        try {
            final String[] temp = {URLEncoder.encode(value, StandardCharsets.UTF_8.toString())};
            blackList.forEach((k,v)->{
                temp[0] = temp[0].replaceAll(k, v);
            });
            return temp[0];
        } catch (final UnsupportedEncodingException ex) {
            return value;
        }
    }

    @Override
    public int getOrder() {
        return 10000;
    }
}
```