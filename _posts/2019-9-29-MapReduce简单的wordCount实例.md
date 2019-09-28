### 使用MapReduce的前提条件 ###

   待处理的数据集可以分解成许多小的数据集，而且每一个小数据集都可以完全并行地进行处理。在wordCount程序任务中，不同的单词之间的频数不存在相关性，彼此独立，可以吧不同的单词分发给不同的机器进行处理。所以，我们可以使用MapReduce实现词频统计。

   利用Java编写MapReduce程序需要以下几个步骤。

1. 编写Map处理逻辑
2. 编写Reduce处理逻辑
3. 编写main方法
4. 对编译好的代码打包以及运行程序

### 编写Map处理逻辑 ###

   首先，我们要搞清楚Map的输入和输出对象。输入时，Map采用Hadoop默认的<key, value>输入方式，即使用文件的行号作为key，每一行的内容作为value；Map输出采用<key, 1>输出，即key代表输出的单词，1代表输出了一次。所以，最后在java代码中，我们确定输入类型为<Object, Text>, 输出类型为<Text, IntWritable>。为了实现具体的分析操作，我们需要继承Mapper类并重写map函数。

```java
public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable> {
        private static final IntWritable one = new IntWritable(1);
        private Text word = new Text();

        public TokenizerMapper() {
        }

        //重写map方法
        public void map(Object key, Text value, Mapper<Object, Text, Text, IntWritable>.Context context) throws IOException, InterruptedException {
            //将每一个单词写入itr
            StringTokenizer itr = new StringTokenizer(value.toString());

            while(itr.hasMoreTokens()) {
                this.word.set(itr.nextToken());
                //以<key,1>的格式输出
                context.write(this.word, one);
            }

        }
    }
```

   在运行代码结束后，我们得到经过map逻辑之后的处理结果为<"am", 1>, <"China", 1>等等。

   Map部分得到中间结果后，接下来会进入到shuffle阶段，这个阶段Hadoop会自动将Map的输出结果进行分区、排序、合并。但如果我们阅读过一些资料后，会发现，shuffle之后如果提供Combiner函数对中间结果进行合并后再发送给Reduce，会大大减少网络传输数据量。举个例子

   没有Combiner函数之前：<"Hello",<1,1,1> >

   有Combiner函数之后：<"Hello",3>

   Combiner函数其实类似于Reduce逻辑过程，我们可以在main函数中直接调用，具体请看后面WordCount完整代码。

### 编写Reduce处理逻辑 ###

   Reduce处理逻辑。基本与Map过程类似。只是IntWritable在shuffle阶段处理过后，变成了Iteraber迭代器容器。在Reduce过程中，我们必须遍历这个容器，对该容器中的数字进行累加，得到单词总数。具体代码如下：

```java
public static class IntSumReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
        private IntWritable result = new IntWritable();

        public IntSumReducer() {
        }

        public void reduce(Text key, Iterable<IntWritable> values, Reducer<Text, IntWritable, Text, IntWritable>.Context context) throws IOException, InterruptedException {
            int sum = 0;

            IntWritable val;
            //将Iterable中的数字迭代累加
            for(Iterator i$ = values.iterator(); i$.hasNext(); sum += val.get()) {
                val = (IntWritable)i$.next();
            }

            this.result.set(sum);
            context.write(key, this.result);
        }
    }
```

### 编写main函数并将三部分放入WordCount类中 ###

   在主函数中，我们需要在主函数中通过Job类设置Hadoop程序运行时的环境变量，我们通过Configuration获得程序运行时的参数情况，并将他们储存在String[] otherArgs中。然后，我们通过类Job设置环境参数。接下来，我将所有逻辑放入WordCount中，不想全部敲一遍代码的同学可以直接复制，代码如下：

```java
import java.io.IOException;
import java.util.Iterator;
import java.util.StringTokenizer;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.GenericOptionsParser;

public class WordCount {
    public WordCount() {
    }

    public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();
        String[] otherArgs = (new GenericOptionsParser(conf, args)).getRemainingArgs();
        if(otherArgs.length < 2) {
            System.err.println("Usage: wordcount <in> [<in>...] <out>");
            System.exit(2);
        }

        Job job = Job.getInstance(conf, "word count");
        job.setJarByClass(WordCount.class);
        job.setMapperClass(WordCount.TokenizerMapper.class);
        //Combiner函数直接调用的时Reduce逻辑过程
        job.setCombinerClass(WordCount.IntSumReducer.class);
        job.setReducerClass(WordCount.IntSumReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        for(int i = 0; i < otherArgs.length - 1; ++i) {
            FileInputFormat.addInputPath(job, new Path(otherArgs[i]));
        }

        FileOutputFormat.setOutputPath(job, new Path(otherArgs[otherArgs.length - 1]));
        System.exit(job.waitForCompletion(true)?0:1);
    }

    public static class IntSumReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
        private IntWritable result = new IntWritable();

        public IntSumReducer() {
        }

        public void reduce(Text key, Iterable<IntWritable> values, Reducer<Text, IntWritable, Text, IntWritable>.Context context) throws IOException, InterruptedException {
            int sum = 0;

            IntWritable val;
            for(Iterator i$ = values.iterator(); i$.hasNext(); sum += val.get()) {
                val = (IntWritable)i$.next();
            }

            this.result.set(sum);
            context.write(key, this.result);
        }
    }

    public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable> {
        private static final IntWritable one = new IntWritable(1);
        private Text word = new Text();

        public TokenizerMapper() {
        }

        public void map(Object key, Text value, Mapper<Object, Text, Text, IntWritable>.Context context) throws IOException, InterruptedException {
            StringTokenizer itr = new StringTokenizer(value.toString());

            while(itr.hasMoreTokens()) {
                this.word.set(itr.nextToken());
                context.write(this.word, one);
            }

        }
    }
}

```

### 代码打包以及程序运行 ###

   首先是关于代码打包，具体有两种常用的方法，一是直接在idea中使用直接打包，二是通过书上P150页的方法进行打包。

  再有就是关于程序运行，书上的命令是这样的

```
./bin/hadoop jar wordCount.jar WordCount input output
```

它这里的input和output文件夹是在/usr/root/文件夹下的，如果你直接在根目录/下创建的input文件夹，直接在input和output前面加/就好了。

  运行jar包除了书上的方法，还有一种就是直接运行shell程序，但是如果直接运行shell程序没有自己单独一步一步的写命令更加灵活，所以我更推荐直接在终端命令行敲命令。

  最后，这篇文章是我基于书上对于mapreduce讲解的一些小扩展，大家在做实验的时候可以参考一下。
