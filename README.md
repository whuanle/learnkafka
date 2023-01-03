

## C#

在 C# 中使用 Kafka ，最佳的客户端库是 confluent-kafka-dotnet，其底层使用了一个 C 语言编写的库 librdkafka。

直接搜索 Confluent.Kafka 即可引用。



### 生产者

```bash
using Confluent.Kafka;
using System.Net;

public class Program
{
    static void Main()
    {
        
        var config = new ProducerConfig
        {
            BootstrapServers = "host1:9092",
            ...
        };

        using (var producer = new ProducerBuilder<Null, string>(config).Build())
        {
            ...
        }
    }
}
```



如果要将消息推送到 Kafka，

```
var result = await producer.ProduceAsync("weblog", new Message<Null, string> { Value="a log message" });
```



```
using Confluent.Kafka;
using System.Net;

public class Program
{
    static async Task Main()
    {
        var config = new ProducerConfig
        {
            BootstrapServers = "192.168.3.156:9092"
        };

        using (var producer = new ProducerBuilder<Null, string>(config).Build())
        {
            var result = await producer.ProduceAsync("weblog", new Message<Null, string> { Value = "a log message" });
        }
    }
}
```

![image-20221217105932107](images/image-20221217105932107.png)

![image-20221217105953883](images/image-20221217105953883.png)



推送消息之后

![image-20221217110035589](images/image-20221217110035589.png)

### 批量生产



```csharp
using Confluent.Kafka;
using System.Net;

public class Program
{
    static async Task Main()
    {
        var config = new ProducerConfig
        {
            BootstrapServers = "192.168.3.156:9092"
        };

        using (var producer = new ProducerBuilder<Null, string>(config).Build())
        {
            for (int i = 0; i < 10; ++i)
            {
                producer.Produce("my-topic", new Message<Null, string> { Value = i.ToString() }, handler);
            }
        }
        // 帮忙程序自动退出
        Console.ReadKey();
    }

    public static void handler(DeliveryReport<Null, string> r)
    {
        Console.WriteLine(!r.Error.IsError
            ? $"Delivered message to {r.TopicPartitionOffset}"
            : $"Delivery Error: {r.Error.Reason}");
    }
}
```



Produce 方法也是异步的，因为它从不阻塞。消息传递信息可以通过后台线程上的(可选的)传递报告处理程序实现带外可用。Product 方法更直接地映射到底层的 library dkafka production API，因此比 ProduceAsync 的开销要小一些。但是 ProduceAsync 仍然非常高性能——在典型的硬件上每秒能够产生数十万条消息。



![image-20221217111011471](images/image-20221217111011471.png)



```
using Confluent.Kafka;
using System.Net;

public class Program
{
    static async Task Main()
    {
        var config = new ProducerConfig
        {
            BootstrapServers = "192.168.3.156:9092"
        };

        using (var producer = new ProducerBuilder<Null, string>(config).Build())
        {
            for (int i = 0; i < 10; ++i)
            {
                producer.Produce("my-topic", new Message<Null, string> { Value = i.ToString() }, handler);
            }
            producer.Flush(TimeSpan.FromSeconds(10));
        }
        // 帮忙程序自动退出
        Console.ReadKey();
    }

    public static void handler(DeliveryReport<Null, string> r)
    {
        Console.WriteLine(!r.Error.IsError
            ? $"Delivered message to {r.TopicPartitionOffset}"
            : $"Delivery Error: {r.Error.Reason}");
    }
}
```





调用Flush此方法可使所有缓冲记录立即可用于发送（即使linger.ms大于0）并在与这些记录关联的请求完成时发生阻塞。 

如果将 Kafka 服务停止，客户端肯定是不能推送消息的，启动程序，会发现 producer.Flush(TimeSpan.FromSeconds(10)); 会等待 10s。此时 handler 也不会起效。

![image-20221217111733131](images/image-20221217111733131.png)



那么，我们应该怎么样获取当前批量提交的结果呢？



```
            var result = producer.Flush(TimeSpan.FromSeconds(10));
            Console.WriteLine(result);
```

会返回失败的数量

![image-20221217112413365](images/image-20221217112413365.png)



有个疑问，当超过 10s 是，没有成功发送的消息还会再发送吗？



一般我们都会很关心失败重试等。



### 使用 Tasks.WhenAll

除了框架本身的批量提交，我们也可以利用 Tasks.WhenAll 来实现批量提交获取返回结果。

只是框架本身的是使用底层的接口，

Produce 方法也是异步的，因为它从不阻塞。消息传递信息可以通过后台线程上的(可选的)传递报告处理程序实现带外可用。Product 方法更直接地映射到底层的 library dkafka production API，因此比 ProduceAsync 的开销要小一些。但是 ProduceAsync 仍然非常高性能——在典型的硬件上每秒能够产生数十万条消息。

```csharp
        using (var producer = new ProducerBuilder<Null, string>(config).Build())
        {
            List<Task> tasks = new();
            for (int i = 0; i < 10; ++i)
            {
                var task = producer.ProduceAsync("my-topic", new Message<Null, string> { Value = i.ToString() });
                tasks.Add(task);
            }
            await Task.WhenAll(tasks.ToArray());
        }
```



```csharp
using Confluent.Kafka;
using System.Net;
using System.Security.Cryptography;
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;
using BenchmarkDotNet.Jobs;

public class Program
{
    static void Main()
    {
        var summary = BenchmarkRunner.Run<KafkaProduce>();
    }
}

[SimpleJob(RuntimeMoniker.Net70)]
[SimpleJob(RuntimeMoniker.NativeAot70)]
[RPlotExporter]
public class KafkaProduce
{
    // 每批消息数量
    [Params(1000, 10000,100000)]
    public int N;

    private ProducerConfig _config;
    
    
    [GlobalSetup]
    public void Setup()
    {
        _config = new ProducerConfig
        {
            BootstrapServers = "192.168.3.156:9092"
        };
    }

    [Benchmark]
    public async Task UseAsync()
    {
        using (var producer = new ProducerBuilder<Null, string>(_config).Build())
        {
            List<Task> tasks = new();
            for (int i = 0; i < N; ++i)
            {
                var task = producer.ProduceAsync("ben1-topic", new Message<Null, string> { Value = i.ToString() });
                tasks.Add(task);
            }
            await Task.WhenAll(tasks);
        }
    }

    [Benchmark]
    public void UseLibrd()
    {
        using (var producer = new ProducerBuilder<Null, string>(_config).Build())
        {
            for (int i = 0; i < N; ++i)
            {
                producer.Produce("ben2-topic", new Message<Null, string> { Value = i.ToString() }, null);
            }
            producer.Flush(TimeSpan.FromSeconds(60));
        }
    }
}
```



```
正在 Ping 192.168.3.156 具有 32 字节的数据:
来自 192.168.3.156 的回复: 字节=32 时间=1ms TTL=64
来自 192.168.3.156 的回复: 字节=32 时间=2ms TTL=64
来自 192.168.3.156 的回复: 字节=32 时间=2ms TTL=64
来自 192.168.3.156 的回复: 字节=32 时间=1ms TTL=64
```

| Method   | Job           | Runtime       | N      |     Mean |   Error |   StdDev |       Gen0 |      Gen1 |      Gen2 |    Allocated |
| -------- | ------------- | ------------- | ------ | -------: | ------: | -------: | ---------: | --------: | --------: | -----------: |
| UseAsync | .NET 7.0      | .NET 7.0      | 1000   | 125.1 ms | 2.21 ms |  2.17 ms |          - |         - |         - |   1055.43 KB |
| UseLibrd | .NET 7.0      | .NET 7.0      | 1000   | 124.7 ms | 2.26 ms |  2.12 ms |          - |         - |         - |    359.18 KB |
| UseAsync | NativeAOT 7.0 | NativeAOT 7.0 | 1000   | 124.8 ms | 1.83 ms |  1.62 ms |          - |         - |         - |   1055.43 KB |
| UseLibrd | NativeAOT 7.0 | NativeAOT 7.0 | 1000   | 125.1 ms | 1.76 ms |  1.64 ms |          - |         - |         - |    359.18 KB |
| UseAsync | .NET 7.0      | .NET 7.0      | 10000  | 143.9 ms | 3.70 ms | 10.86 ms |  1250.0000 |  750.0000 |  250.0000 |  10577.22 KB |
| UseLibrd | .NET 7.0      | .NET 7.0      | 10000  | 140.6 ms | 2.74 ms |  4.80 ms |   250.0000 |         - |         - |   3523.29 KB |
| UseAsync | NativeAOT 7.0 | NativeAOT 7.0 | 10000  | 145.7 ms | 3.25 ms |  9.59 ms |  1250.0000 |  750.0000 |  250.0000 |  10577.22 KB |
| UseLibrd | NativeAOT 7.0 | NativeAOT 7.0 | 10000  | 140.6 ms | 2.78 ms |  5.56 ms |   250.0000 |         - |         - |   3523.29 KB |
| UseAsync | .NET 7.0      | .NET 7.0      | 100000 | 407.3 ms | 7.17 ms |  9.58 ms | 13000.0000 | 7000.0000 | 2000.0000 | 105185.91 KB |
| UseLibrd | .NET 7.0      | .NET 7.0      | 100000 | 259.7 ms | 5.72 ms | 16.78 ms |  4000.0000 |         - |         - |  35164.82 KB |
| UseAsync | NativeAOT 7.0 | NativeAOT 7.0 | 100000 | 419.8 ms | 8.31 ms | 13.19 ms | 14000.0000 | 8000.0000 | 2000.0000 |  105194.3 KB |
| UseLibrd | NativeAOT 7.0 | NativeAOT 7.0 | 100000 | 255.3 ms | 6.31 ms | 18.62 ms |  4000.0000 |         - |         - |  35164.72 KB |

![image-20221217161812034](images/image-20221217161812034.png)



### 消费

```csharp
using System.Collections.Generic;
using Confluent.Kafka;

...

var config = new ConsumerConfig
{
    BootstrapServers = "host1:9092,host2:9092",
    GroupId = "foo",
    AutoOffsetReset = AutoOffsetReset.Earliest
};

using (var consumer = new ConsumerBuilder<Ignore, string>(config).Build())
{
    ...
}
```



```
using Confluent.Kafka;
using System.Net;

public class Program
{
    static void Main()
    {
        var config = new ConsumerConfig
        {
            BootstrapServers = "192.168.3.156:9092",
            GroupId = "test1",
            AutoOffsetReset = AutoOffsetReset.Earliest
        };

        CancellationTokenSource source = new CancellationTokenSource();
        using (var consumer = new ConsumerBuilder<Ignore, string>(config).Build())
        {
            consumer.Subscribe("my-topic");

            while (!source.IsCancellationRequested)
            {
                var consumeResult = consumer.Consume(source.Token);
                Console.WriteLine(consumeResult.Message.Value);
            }

            consumer.Close();
        }
    }
}
```

