---
title: springboot 使用Gzip或Snappy压缩数据
status: Done
转载: https://blog.csdn.net/zzhongcy/article/details/108487879
Tags:
  - redis
---

## Gzip 实现

```java
public class RedisSerializerGzip extends JdkSerializationRedisSerializer {
 
    @Override
    public Object deserialize(byte[] bytes) {
        return super.deserialize(decompress(bytes));
    }
 
    @Override
    public byte[] serialize(Object object) {
        return compress(super.serialize(object));
    }
 
 
    private byte[] compress(byte[] content) {
        byte[] ret = null;
        ByteArrayOutputStream byteArrayOutputStream = null;
        try {
            byteArrayOutputStream = new ByteArrayOutputStream();
            GZIPOutputStream gzipOutputStream= new GZIPOutputStream(byteArrayOutputStream);
            gzipOutputStream.write(content);
            //gzipOutputStream.flush();     //只调用flush不会刷新，压缩类型的流需要执行close或者finish才会完成
            stream.close();   //内部调用finish
 
            ret = byteArrayOutputStream.toByteArray();
            byteArrayOutputStream.flush();
            byteArrayOutputStream.close();
        } catch (IOException e) {
            throw new SerializationException("Unable to compress data", e);
        }
        return ret;
    }
 
    private byte[] decompress(byte[] contentBytes) {
        byte[] ret = null;
        ByteArrayOutputStream out = null;
        try {
            out = new ByteArrayOutputStream();
            GZIPInputStream stream = new GZIPInputStream(new ByteArrayInputStream(contentBytes));
            IOUtils.copy(stream, out);
            stream.close();
 
            ret = out.toByteArray();
            out.flush();
            out.close();
        } catch (IOException e) {
            throw new SerializationException("Unable to decompress data", e);
        }
        return ret;
    }
 
}
```

## Snappy 实现

```java
import org.springframework.data.redis.serializer.JdkSerializationRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializer;
import org.springframework.data.redis.serializer.SerializationException;
import org.springframework.util.SerializationUtils;
import org.xerial.snappy.Snappy;
import java.io.Serializable;
 
public class RedisSerializerSnappy extends JdkSerializationRedisSerializer {
 
    private RedisSerializer<Object> innerSerializer;
 
    public RedisSerializerSnappy(RedisSerializer<Object> innerSerializer) {
        this.innerSerializer = innerSerializer;
    }
 
    @Override
    public Object deserialize(byte[] bytes) {
        return super.deserialize(decompress(bytes));
    }
 
    @Override
    public byte[] serialize(Object object) {
        return compress(super.serialize(object));
    }
 
    private byte[] compress(byte[] content) {
        try {
            byte[] bytes = innerSerializer != null ? innerSerializer.serialize(content)
                    : SerializationUtils.serialize((Serializable) content);
            return Snappy.compress(bytes);
 
        } catch (Exception e) {
            throw new SerializationException(e.getMessage(), e);
        }
    }
 
    private byte[] decompress(byte[] contentBytes) {
        try {
            byte[] bos = Snappy.uncompress(contentBytes);
            return (byte[]) (innerSerializer != null ? innerSerializer.deserialize(bos) : SerializationUtils.deserialize(bos));
        } catch (Exception e) {
            throw new SerializationException(e.getMessage(), e);
        }
    }
}
```


## 参考文章

[用Gzip数据压缩方式优化redis大对象缓存 - woshare - 博客园 (cnblogs.com)](https://www.cnblogs.com/woshare/p/15955460.html)
[Sprintboot redis 采用gzip和Snappy compress压缩数据_springboot redis 数据压缩-CSDN博客](https://blog.csdn.net/zzhongcy/article/details/108487879)