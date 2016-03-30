Docker image based on [gliderlabs/logspout](https://registry.hub.docker.com/u/gliderlabs/logspout/) with added redis adapter (github.com/brendangibat/logspout-redis-logstash).


## Logspout Options to Process Container Messages
To make logspout ignore a containers logs, set the environmental variable and value of 'LOGSPOUT=ignore' on the container being ignored for it to be ignored.

Logspout with the redis module can also be configured to pass options in to the ELK stack by way of environmental variables at the logspout container level, which can be overrode by the individual containers logging messages, for example:
```
docker run -it -v /var/run/docker.sock:/tmp/docker.sock -e 'OPTIONS={"test":"logspout"}' brendangibat/docker-logspout-redis:latest redis://redis.url:6379
```

Sample container logging to stdout
```
docker run -d -e TEST=LEON -e TEST2=LEON2 -e 'LOGSPOUT_OPTIONS={"test":"leon"}' ubuntu echo 'hello world'
```

In the case above, when the sample container logs any messages, the logspout container forwarding to redis will set an object at the document root of options.test=leon
In this way, individual containers or the logspout container running on a server can set the logging policies and extra information to tag forwarded messages. Such values in the options document set include overrides about what index to use, and if the message should be decoded as JSON data.

Logspout-redis can be configured to at its own container level for what type of logs it expects to process for all of the containers it monitors.

Logspout-logstash sending json logs to custom_index index:

```
docker run -it -v /var/run/docker.sock:/tmp/docker.sock -e 'OPTIONS={"_logging_index":"custom_index","codec":"json"}' brendangibat/docker-logspout-redis:latest redis://redis.url:6379
```
## Individual Container Overrides of Options

Additionally, Logspout-redis looks at the environmental variables of each container for a hash object in the env var 'LOGSPOUT_OPTIONS' and overrides any setting at the logspout-logstash container level of log processing.

Logspout-logstash will process these as json logs to test index

```
docker run -d -e 'LOGSPOUT_OPTIONS={"test":"hello world", "_logging_index":"test", "codec":"json", "type":"test_type"}' ubuntu echo '{"testKeyName": "hello world test"}'
```

The above docker command will send its logs to the test index and be processed as a json object with the root type "test_type" which will merge its document on to the root document ingested in to ELK.


## Logstash Filters

This works well in tandem with logstash filters. By passing in custom options on individual containers or for all messages logged by a running logspout container, you can partition your environments logs in to different indexes or ElasticSearch hosts, let the containers specify their custom format output, change the type per source container or logspout container, and other behaviors as needed.

```

input {
    redis {
        host => 'some_host.url'
        key => 'some_key'
        data_type => 'list'
        type => 'redis'
    }
}


filter {
  if [options][type] {
    mutate {
      replace => {
        "type" => "%{[options][type]}"
      }
    }
  }
  if [data][message][type] {
    mutate {
      replace => {
        "type" => "%{[data][message][type]}"
      }
    }
  }
  if [options][codec] == "json" {
    json {
      source => "message"
    }
  }
}

output {
  if [options][_logging_index] {
      amazon_es {
          hosts => ['es_host.url']
          region => 'aws_region'
          index => '%{[options][_logging_index]}-%{+YYYY.MM.dd}'
      }
  } else {
      amazon_es {
          hosts => ['es_host.url']
          region => 'aws_region'
          index => 'logging-%{+YYYY.MM.dd}'
      }
  }
}
```
