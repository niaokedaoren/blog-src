---
title: Sequence File的优点
date: 2016-12-12 21:54:02
categories:
- hadoop
tags:
- mapreduce
- sequencefile
---

### 缘起
最近遇到个问题，我需要处理上游生成的很多数据文件，文件统一存放在hdfs上，所以方便用mr处理，但是单个文件的size差别很大，有几m的，也有几g对。我的整个处理流程分两步，第一步预处理文件，因为上游任务会陆续生产文件，所以单独起mr分别预处理；第二步，把预处理的文件全部merge在一起，预处理的文件我都是采用gz压缩，所以每个文件会启一个map，导致两个问题，1）大文件会拖慢整个job的节奏，2）小文件处理启动的map数量过多。

### sequence file
解决方案，就是预处理结果采用[sequencefile](https://wiki.apache.org/hadoop/SequenceFile)格式存储，因为sequencefile格式的文件支持大文件split成多个map执行，多个小文件合并成一个map任务执行。用sequencefile，需要注意一个问题，就是下游使用这些文件的map任务的key类型，要和上游输出的key类型一致，例如我们例子中的LongWritable类型；另外，如果mr的sequencefile输出要导入到hive中使用，那么hive就会自动忽略key字段，一般的解决方案，就是输出一个new Text("")的key，然后需要的字段学到value中。

### 代码
1. 预处理mr代码，需要setOutputFormatClass为SequenceFileOutputFormat.class，另外注意这里设置OutputKeyClass为LongWritable，下游任务的map key需要保持一致。
```
public class CookAsPb extends Configured implements Tool {
    public int run(String[] strings) throws Exception {
        Configuration conf = getConf();
        conf.set("mapreduce.map.output.compress", "true");
        ...
        Job job = Job.getInstance(conf);
        job.setJarByClass(getClass());
        job.setMapperClass(CookAsPbMapper.class);
        job.setOutputKeyClass(LongWritable.class);
        job.setOutputValueClass(Text.class);
        job.setOutputFormatClass(SequenceFileOutputFormat.class);
        FileOutputFormat.setCompressOutput(job, true);
        FileOutputFormat.setOutputCompressorClass(job, GzipCodec.class);
       ...
        return ret;
    }

    public static void main(String[] args) throws Exception {
        int ret = ToolRunner.run(new Configuration(), new CookAsPb(), args);
        System.exit(ret);
    }
}
```

2. mereg任务代码，setInputFormatClass为SequenceFileOutputFormat.class，mapper代码的input key为LongWritable。
```
public class MergePbs extends Configured implements Tool {
    public int run(String[] strings) throws Exception {
        ...
        job.setMapperClass(MergePbsMapper.class);
        job.setInputFormatClass(SequenceFileInputFormat.class);
        ...
    }
}

public class MergePbsMapper extends Mapper<LongWritable, Text, Text, Text> {
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        ...
    }
}
```
