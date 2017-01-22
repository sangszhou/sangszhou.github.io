---
layout: post
title: Metric Service
categories: [shell]
keywords: linux, shell, script
---

### Metric data overview

Metric data refers to the collection of data that reflects the working status of the system. Usually, developers use this kind of data to know how well their program works.

There are many types of metric data worth recording, the first one is system metrics such as cpu, memory and disk usage. 
If your system is running on JVM, you can also have jvm metrics recorded as well. Those kind of metric data gives developer the overview of the working condition of the whole system. If the cpu or memory load is high, you might need more powerful hardware or make your system distributed. For JVM based system, there's a lot useful tools out there to help understand how well the program runs, developers can get JVM metrics data by taking advantage of those tools. So, the first kind of metric data gives your a overview of the working condition of system and can be used to identifying problems. The second kind of metric data is more detail than the first one described above. It aims to help developers understand how their program works in the granularity of method call. How many times a method is called, how long it takes to finish the method call, is there any method call throws exceptions. Those kind of metric data help developers investigate the problems by finding the bottleneck of your program and optimize it accordingly.

In summary, Monitoring is a critical part in building a large and complex system, and the more complex your system gets, the harder it is to understand its performance and troubleshoot problems. Besides, modern systems don't run in standalone mode, instead they always interact with other systems which introducing the need for cluster monitoring. There's a lot components in iCam yet the metric services still missing from most of them. Mario metric services solve the problem by building the infrastructure to store and analysis the metric data and providing an easy to use, non-intrusive way to log the metric data.

### How to use mario metric service

Mario ships with a built in metric service which enable both backbone and adoption developers to record metrics without writing extra code or building the infrastructure to store and visualize the data. When you are writing an adoption on mario, you get metric service for free, below is an example shows how to use it.

**Get started:**

a simple fetcher of mario adoption.

```java
public class DummyFetcher extends Fetcher {
    
    public static Random r = new Random();

    @InjectService
    LoggingService logger;

    @InjectService
    HttpClientService httpClientService;

    RestTemplate restTemplate;

    @Override
    public void init() {
        HttpComponentsClientHttpRequestFactory clientHttpRequestFactory = 
                new HttpComponentsClientHttpRequestFactory(HttpClientBuilder.create().build());
        restTemplate = httpClientService.createRestTemplate(clientHttpRequestFactory);
    }


    public void nextEvents(EventCollection ec, int limit) {

        logger.log("Generate 2 random events");

        Event ev1 = Event.create();

        ev1.setIntValue("primary", r.nextInt());
        ev1.setIntValue("secondary", r.nextInt());
        ec.putEvent(ev1);

        logger.log("First event is: " + ev1.toJson().get("primary") + " " + ev1.toJson().get("secondary"));

        Event ev2 = Event.create();

        ev2.setIntValue("primary", r.nextInt());
        ev2.setIntValue("secondary", r.nextInt());
        ec.putEvent(ev2);

        logger.log("Second event is: " + ev2.toJson().get("primary") + " " + ev2.toJson().get("secondary"));

        ResponseEntity<String> result = null;

        Random rand = new Random();

        try {
            int prob = rand.nextInt(100);
            if(prob > 70) {
                result = restTemplate.getForEntity("http://icam-dev-soa-01:2015/user/xinszhou", String.class);
            } else if(prob > 30){
                result = restTemplate.getForEntity("http://icam-dev-soa-01:2015/user/xinszh223ou2", String.class);
            } else {
                result = restTemplate.getForEntity("http://icam-dev-soa-01:2015/us3243er/xinszh223ou2", String.class);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

This might be the simplest implementation of fetcher. Every time `nextEvents` is called, it just emit two random integer and 
put them into EventCollection, send a http request to `user info api`. In this method, there is no 
single line of metric related code yet the most important metric data you may interest has already collected for you.

Now, can go to kibana dashboard and visualize the metric data

![](/images/posts/designpattern/metric_data_visualization.png)

The top two figures shows the dump and fetch event number aggregated by adoption name, it record the number of method call of `nextEvent`. The bottom left figure record the status code responded from `user info api`, and bottom right figure shows the event number aggregated by metric name.
 
Those figures are just simple show cases of what you can do with those data, you can plot your metric data according to your 
need. The predefined metrics we are recording now is dump event, fetch event and http request.

```java
public class HttpMetric extends Span {
    String adoption;
    String path;
    String method;
    int statusCode;
    boolean success;
    String exception;
}

public class FetchDumpMetric extends Span {
    String adoption;
    int eventsCount;
}

public abstract class Span implements Metric {

    long start;
    long end;
    long timeConsumed;
    String type = "span";
    String name;
}

public abstract class Event implements Metric {
    long time;
    String type = "event";
    String name;
}
```

Span and Event are two most common usage of metric data. Span usually represent an action with begin and end timing, while
event represent a discrete action itself which don't have time spent attribute.

## How adoption/backbone developers use metric service
 
Metric service will record the metric data of api calls and fetch/dump events, if your need for metric data is one of those, then you don't need to do anything. If your requirement is a little bit complex and need more metric data in your fetch/dump task then you might need to do some extra work to get your metric data. 

One scenario is that you want to record the metric of specific method call. Suppose, you have a 
method called `parseFile` and you want to record the filename, file type and how long it takes to parse the file. With those data you can investigate why certain method call is suspicious long.

```java
// define the metric data
class ParseFileMetric extends Span {
    String fileName;
    int fileSize;
};

// add the metric info you interested to metricService
public void parseFile(String fileName) {
    ParseFileMetric metric = new ParseFileMetric();
    metric.setStart(System.currentTimeMillis());
    
    metric.setName("fetch-event");
    metric.setFileName(fileName);
    
    // parse the file
    
    metric.setEnd(System.currentTimeMillis());
    metricService.add(metric);
};
```

To use MetricService, you must have access to marioComponent where you can get the marioContext from.

The second scenario is that you want to access third party storage or services which can not take advantage of RestTemplate. 
We suppose this scenario is rare, so we didn't weave the metric code with those third party libraries as we do for RestTemplate. But if you do have such need, you can weave the metric code just as we talked about above or contact backbone developer and let them do that for you.

When you need to access API outside mario, use RestTemplate created by HttpClientService, don't create you own one or use 
other http clients. After the metric data has been recorded, go to kibana dashboard and plot the figure you are interested in.

### How to configure metric service

There are two configurable switches used to control the status of metric service which are all enabled by default. 
The first one is `metric.enable` defined in `mario.conf` and second one is  `metric.enableAdoptionMetric` defined in profile.conf. `metric.enable` is controlled by backbone developers, once set to false, no metric data will be recorded.
`metric.enableAdoptionMetric` is controlled by adoption developer, adoption developer can turned it on or off based on the their own requirements.

### The architecture of mario metric service

![pic](/images/posts/designpattern/mario-metic-service-architecture.png)

Metric service get the business semantics information from mario core, such as the adoption name one metric data belong to. 

Metric service write metric data to kafka instead of elasticsearch for kafka has a higher throughput and more stable. Elasticsearch is no-sql database, so we don't need to define schema before writing data to it which means when you want to create a new metric type you could just inherit metric interface and put it to metric service.  

### FAQ

**1.** How to archive old data?

The archive strategy has not been decided yet, for now we don't delete any metric data. We gonna see how many data is produced 
after mario go to production and choose archive strategy accordingly.

**2.** How to know the metric service is running good ?

todo

**3.** Can I use annotation based metric recording instead of programmatically do that?

No, you cannot do that for now. Annotation based metric recording is a easy way to record metrics but that will need AOP which bring a lot of complexity to mario. So, annotation based metric recording is not a priority for backbone team.

**4.** How can I get the system metric data of mario

In production, mario runs as a java program on storm. So, you can leverage the existing monitor tools of storm and jvm to get the metric data you need. Mario metric service aim to help record the detailed metrics data which has business semantics. 