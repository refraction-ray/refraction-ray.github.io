---
layout: post
title: "Timestamps from Filebeat to Elasticsearch"
date: 2019-08-08
excerpt: "Filebeat modules configuration with and without logstash in Elastic stack"
tags: [linux]
comments: true
---

* toc
{:toc}

## Introduction

[Elastic stack](https://www.elastic.co/cn/what-is/elk-stack) (previously known as ELK stack) is a set of very poweful tools for log collection, searching and analysis. In this short post, I will discuss about the specific configurations enabling modules in filebeat and with special focus on the possible timestamp mismatch issues in this setup. The detailed instuctions and what happens behind the scenes are also presented.

## Filebeat

Beat is the last introduced tool out of the four main ingredients of Elastic stack: elasticsearch, kibana, logstash and beat. Therefore, beat is actually the reason why ELK stacks are also refered as Elastic stack now. Beat is very lightweighted data collector which is responsible to collect data at the edge and send data to the next stage (either logstash or elasticsearch(ES) in usual setups). Beats contain several variants, such as heartbeat metricbeat and filebeat, which are designed for differet types of data. Here we will only focus on filebeat, the most commonly used beat, which parses the log files and ship them to some centralized service (logstash or ES).

Installation of filebeat follows the same pattern with the other tools in Elastic stack. Please refer to the official doc on how to install it. On Ubuntu, with previously added elastic.co PPA,  a single line `apt install filebeat` is enough. Make sure the software versions for all the tools in Elastic stack are the same. All the following discussions are based on 6.8.0.

There are two common architectures for elastic stack. The first one is [filebeat]->logstash->[elasticsearch]->kibana; while the second solution can remove logstash as [filebeat]->[elasticsearch]->kibana. (The list symobol [ ] indicates the corresponding ingredients can be deployed on more than one nodes.) Honestly speaking, filebeat itself is now powerful enough and can handle parsing and reformatting log data very well. Besides, filebeat works much better when the output is directly linked to ES (as we will show below). So unless some special needs or you really know what you are doing, I recommend the second architecture as ELK (EBK actually) stacks for newbies. For a brief histroy of logstash and filebeat, you can read [this post](https://logz.io/blog/filebeat-vs-logstash/), though it is a bit outdated, since ingest nodes and their integration with filebeat are not mentioned.

## Configurations

The basic configurations is very simple,  you only need to change /etc/filebeat/filebeat.yml. In the basic case,  you can only tune the output section, either set output.elasticsearch or set output.logstash based on your choice of elastic stack architecture. Before starting filebeat, you need to run the following command in shell:

```bash
$ sudo filebeat setup -e # if filebeat is configured with output ES 
# OR
$ sudo filebeat setup -e -E output.logstash.enabled=false -E output.elasticsearch.hosts=["eshost:9200"] # if filebeat is configured with output logstash
```

By this command, filebeat will do the following tasks: setup index template in ES, setup dashboards in kibana (and possible some extra tasks like machine learning in kibana but we don't need to care about them for now). If you have configured filebeat with output logstash, then you have to overwrite the configuration by `-E` flag at setup time. This is because filebeat has to connect to ES irrespective of its output. Otherwise, filebeat setup cannot write index template into ES.

If you now start filebeat service, you can check logs in kibana at webfront, and you can find that all logs message are in the `message` key, and no further processing and paring on logs are applied. Namely, there is no keys as `system.syslog.program` or `system.syslog.message`. Squashing all logs into one key is not what we want, it is hard for us to search and analyse logs. But this is actually all that filebeat can do: just collecting log datas from log files line by line, and add some metadata of beat, say `beat.host`, `beat.version`, then send the item based on output configuration of filebeat. No reformatting happens, that's why filebeat is super lightweighted.

Since filebeat itself cannot further process logs and split one line of log into multiple meaningful keys, we need ask help for other tools. Logstash of course can do that, this is what logstash is designed for. If you have logstash between filebeat and elasticsearch (i.e. the first architecture of ELK stack), you can add some filter rules (most with grok plugin to reformat) into /etc/logstash/conf.d (see some useful examples in the [doc](https://www.elastic.co/guide/en/logstash/current/config-examples.html)). In this approach, the workflow is: filebeat collects data, logstash reformats data and ES saves data.

ES itself can reformat log data. With (not so) newly introduced ingest nodes and pipelines feature, ES can easily handle log parsing. Intrinsically, we add some middlewares at the input of ES; every input log, if pipeline is claimed, will go through the corresponding pipeline and get reformatted. See ingest pipelines [here](https://www.elastic.co/cn/blog/new-way-to-ingest-part-1) for more details. In this post, we will take this approach. Instead of using logstash filter (which could be somehow complicated to customize by oneself), we utilize pipelines of ES to reformat the log collected by filebeat.

But isn't it also involved to configure and customize pipelines on ES side? Luckily, filebeat can do this automatically. Firstly we enable some modules shipped with filebeat, such as system, nginx or apache by `sudo filebeat modules enable system nginx apache2` (note module names are space separated in this command). If you have enabled some modules in filebeat (which is equivalent to change corresponding filename to yml extension in /etc/filebeat/modules dir), filebeat can create corresponding pipeline by filebeat setup automatically.

If you want to change the configuration for filebeat modules, we need to follow a filebeat reconfigure/reload workflow. `service filebeat restart` won't work as expected. Instead, you need to first stop filebeat service on all the machines. And then most importantly, delete all pipelines previously configured by filebeat modules: `curl -XDELETE 'http://localhost:9200/_ingest/pipeline/filebeat*'`. Now you can start filebeat service again with new configurations in /etc/filebeat/modules.d. The point here is that, the old pipelines on ES won't be overwritten by default unless explicit delete. And since log reformatting happens all on pipelines, you must overwrite them to make new filebeat modules configuration take effects, see [this post](https://discuss.elastic.co/t/filebeat-system-module-auth-log-timezone-conversion-problem/140192/7) for the actions in this paragraph. 


## When logstash is in between

It seems to be simple if we deploy filebeat directly connected to ES. However, if we choose the first architecture with logstash between ES and filebeat, things can get rather involved. Some of them might be feature and some of them might be bug or lack of feature.

First thing to note, ES pipelines for modules of filebeat cannot be automatically set. Instead, it requires another shell command to explicitly set pipeline as

```bash
sudo filebeat setup --pipelines --modules nginx,mysql,iptables,apache2,system  -E output.logstash.enabled=false -E output.elasticsearch.hosts=['eshost:9200']  -M system.auth.var.convert_timezone=true -M system.syslog.var.convert_timezone=true -M nginx.error.var.convert_timezone=true
```

Note the module list here is comma separated and without extra space. Note `-M` here beyond `-E`, they represent configuration overwirtes in modules configs. We will discuss why we need `-M` in this command in the next section.

Similar thing applies to filebeat reload workflow, after deleting the old pipelines, one should run filebeat setup with explicit pipeline args again.

And to make sure the correct pipeline is applied on ES for input log lines, we need to configure logstash output as the following:

```bash
output {
  if [@metadata][pipeline] {
    elasticsearch {
      hosts => ["eshost:9200"]
      manage_template => false
      index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
      pipeline => "%{[@metadata][pipeline]}"
    }
  } else {
    elasticsearch {
      hosts => ["eshost:9200"]
      manage_template => false
      index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
    }
  }
}
```
## Timestamp mismatch

Since I am not living in London, time related issue is always a nightmare to me. And after enabling filebeat modules and restart filebeat service following the reloading workflow, by opening the browser and checking on kibana, you may find no logs or at least no syslogs. But wait, by changing the time range on kibana from the last 15 mins to this week, logs are there and … in the future. Strictly speaking, in the eight hours later  which reminds me of my timezone (Asia/Beijing +8). 

It is obviously some time mismatch issues related to timezone settings, but there are several questions immediately: 1. Why some logs have correct timestamps, but some logs not (including syslog and auth.log)? 2. Why the incorrect timestamp is +8 from now? If it is a timezone issue, it seems to me that the incorrect timestamp should be -8.

The answer to the first question: logs have different forms. For nginx/acess.log, one line looks like

```
166.111.*.* - abc [08/Aug/2019:08:03:03 +0800] "GET / HTTP/1.1" 200 2058 "http://*/app/kibana" 
```

Note +0800 in the log, namely the log itself already contains information about time zone. In other words, anyone can confidently convert the log timestamps into UTC time. But for syslog, one line looks like

```
Aug  8 10:39:07 master dnsmasq[2224]: query[A] 3.ubuntu.pool.ntp.org from *
```

In this log, there is no timezone information. And by the log file itself, we cannot judge whether the time here is UTC or related to some other timezones. Since rsyslog can actually be configured to use a different timezone from local machine ([ref](https://stackoverflow.com/questions/39094180/change-timezone-to-utc-in-rsyslog-configuration)), system module from filebeat thus cannot determine the timezone of such log files. By the default configuration of system.yml in /etc/filebeat/modules.d, filebeat setup doesn't set any timezone convertion on pipelines for ES. In other words, the literal time string will be treated as UTC time by default.

The answer to the second question: if the literal time is written into ES, why it happens in the future? This turns out to be something related with kibana. On ES side, `@timestamp` key are always UTC times and thus timezone agnostic. The reason that we feel the timestamps are just right as local time in kibana is because the default setting of kibana is rendering the timestamp with timezone defined by the broswer, aka. timezone in  local OS. One can change this in kibana management: kibana advanced settings. Therefore, since the timestamp for syslog is literally the same with the one written in the log, and it is treated as UTC time, with kibana running on machines with +8 timezone, these logs happen in the future.

Solutions for timestamp mismatch: just tell filebeat modules that the log file timestamp is consistent with timezone on local machine and hence timezone convertion is needed within pipelines in ES. This option is in /etc/filebeat/modules/system.yml, as `var.convert_timezone: true`. By setting this for corresponding modules, and reload filebeat (recall the reload workflow), the timestamps in kibana should be right this time.

But if you have configured logstash in between, no, you are still not set. This might be a [bug](https://discuss.elastic.co/t/filebeat-pipeline-setup-for-system-module-ignores-var-convert-timezone-true/181456/2). var configuration on system.yml will be omitted when filebeat is connected to logstash. When filebeat setup the pipelines, the timezone info doesn't enter in. One can check this by `curl -XGET 'http://eshost:9200/_ingest/pipeline/filebeat-6.8.0-system*|grep timezone`, and nothing returned, indicating timezone key is not written into pipelines on ES even the var is configured in filebeat modules. The workaround, is the `-M` flag we have shown in the last section. By overwirting module configurations on command line, the resulted new pipelines on ES are correct with timezone info.

Note not only syslog and auth.log watched by system module have timestamp issues, nginx/error.log also has this subtlety. One line from nginx.error looks like

```
2019/08/08 08:03:02 [error] 2121#2121 blah blah
```

It is surprising that access log from nginx contains timezone information but error log doesn't. Anyway, remember adding convert_timezone option to nginx.yml under error section in filebeat modules configuration, too.

## Summary

One tip and two lessons.

The tip: When you configure ELK stacks and find no logs shown on kibana by default, try increasing the time range (even including the future), you might find the misconfig is just a timestamp mismatch issue.

Lesson 1: It is much harder to configure the ELK stack with logstash in between as we have seen in this post again and again. There is actually no need to try logstash for a basic log collection infrastructure. Filebeat is good enough.

Lesson 2: Time issue is alway a huge pain, not only in Elastic stack but also in software development in general. Let alone daylight saving time, business days...