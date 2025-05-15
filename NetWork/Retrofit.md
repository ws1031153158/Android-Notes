# ResponseBody
## .string()
HTTP 响应体本质上是单向流(OkHttp的ResponseBody是基于一次性的流)，一旦读取完毕，流就会被关闭  
第一次调用string()之后，流已经关闭，第二次调用时就会抛出异常或者返回空值  
此外，response.body().bytes() 也只可调用一次  
response.body().charStream() 可以调用多次
