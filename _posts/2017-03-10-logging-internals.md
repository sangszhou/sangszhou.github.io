---
layout: post
title:  "logback 的实现原理"
date:   "2016-03-10 17:50:00"
categories: java
keywords: java
---

分析 Logback

### Logger

```java
public final class Logger implements org.slf4j.Logger, LocationAwareLogger, AppenderAttachable<ILoggingEvent>, Serializable {
    private String name;
    // The assigned levelInt of this logger. Can be null.
    transient private Level level;
    // The effective levelInt is the assigned levelInt and if null, a levelInt is
    // inherited form a parent.
    transient private int effectiveLevelInt;
      /**
       * The parent of this category. All categories have at least one ancestor
       * which is the root category.
       */
    transient private Logger parent;
      /**
       * The children of this logger. A logger may have zero or more children.
       */
    transient private List<Logger> childrenList;
    transient private AppenderAttachableImpl<ILoggingEvent> aai;
    // default to true
    transient private boolean additive = true;
      
    final transient LoggerContext loggerContext;
    
    // 真正的 info, warn, error 过程，托管给 appender
    private void buildLoggingEventAndAppend(final String localFQCN,
      final Marker marker, final Level level, final String msg,
      final Object[] params, final Throwable t) {
        LoggingEvent le = new LoggingEvent(localFQCN, this, level, msg, t, params);
        le.setMarker(marker);
        callAppenders(le);
  }

// 这是 append 继承的代码实现，appender 会调用父亲的实现，除非显式的调用 additive 方法
  public void callAppenders(ILoggingEvent event) {
    int writes = 0;
    for (Logger l = this; l != null; l = l.parent) {
      writes += l.appendLoopOnAppenders(event);
      if (!l.additive) {
        break;
      }
    }
    // No appenders in hierarchy
    if (writes == 0) {
      loggerContext.noAppenderDefinedWarning(this);
    }
  }

   
  private int appendLoopOnAppenders(ILoggingEvent event) {
    if (aai != null) {
      return aai.appendLoopOnAppenders(event);
    } else {
      return 0;
    }
  }
  
  // appender 是一个 list, 可以动态的添加，后面的就是 appender 接口的操作了
  public int appendLoopOnAppenders(E e) {
    int size = 0;
      for (Appender<E> appender : appenderList) {
        appender.doAppend(e);
        size++;
      }
    return size;
  }

}
```

### appender

```java
public interface Appender<E> extends LifeCycle, ContextAware, FilterAttachable<E> {
  String getName();

  void doAppend(E event) throws LogbackException;

  void setName(String name);
}
```

以 Outputstream 为例

```java
public class OutputStreamAppender<E> extends UnsynchronizedAppenderBase<E> {
  protected Encoder<E> encoder;
  protected LogbackLock lock = new LogbackLock();
  private OutputStream outputStream;
    
  // 使用 encoder 编码过程
  protected void writeOut(E event) throws IOException {
    this.encoder.doEncode(event);
  }

  // 使用同步保证线程安全，防止出现脏数据 
  protected void subAppend(E event) {
    if (!isStarted()) {
      return;
    }
    try {
      // this step avoids
      if (event instanceof DeferredProcessingAware) {
        ((DeferredProcessingAware) event).prepareForDeferredProcessing();
      }
      // the synchronization prevents the OutputStream from being closed while we
      // are writing. It also prevents multiple threads from entering the same
      // converter. Converters assume that they are in a synchronized block.
      synchronized (lock) {
        writeOut(event);
      }
    } catch (IOException ioe) {
      // as soon as an exception occurs, move to non-started state
      // and add a single ErrorStatus to the SM.
      this.started = false;
      addStatus(new ErrorStatus("IO failure in appender", this, ioe));
    }
  }
}
```

### Encoder & Layout

encoder 和 layout 是一个东西，只不过因为历史原因而共存的吧

```java
public class LayoutKafkaMessageEncoder<E> extends KafkaMessageEncoderBase<E> {

    public LayoutKafkaMessageEncoder() {}

    public LayoutKafkaMessageEncoder(Layout<E> layout, Charset charset) {
        this.layout = layout;
        this.charset = charset;
    }

    private Layout<E> layout;
    private Charset charset;
    private static final Charset UTF8 = Charset.forName("UTF-8");

    @Override
    public void start() {
        if (charset == null) {
            addInfo("No charset specified for PatternLayoutKafkaEncoder. Using default UTF8 encoding.");
            charset = UTF8;
        }
        super.start();
    }

    @Override
    public String doEncode(E event) {
        final String message = layout.doLayout(event);
        return message;
    }

    public void setLayout(Layout<E> layout) {
        this.layout = layout;
    }

    public void setCharset(Charset charset) {
        this.charset = charset;
    }

    public Layout<E> getLayout() {
        return layout;
    }

    public Charset getCharset() {
        return charset;
    }

}

public class DefaultLayout extends LayoutBase<ILoggingEvent> {
    MarioContext marioContext;
    Optional<String> hostname = NetworkUtils.getIpAddr();

    public DefaultLayout(MarioContext marioContext) {
        this.marioContext = marioContext;
    }

    public String doLayout(ILoggingEvent event) {
        StringBuffer sbuf = new StringBuffer(128);
        if(this.hostname.isPresent()) {
            sbuf.append((String)this.hostname.get() + " ");
        }


        sbuf.append(DateTimeUtils.utcCurrent("yyyy-MM-dd HH:mm:ss"));
        sbuf.append(" ");
        sbuf.append(event.getLevel());
        sbuf.append(" [");
        sbuf.append(event.getThreadName());
        sbuf.append("] ");
        sbuf.append(event.getLoggerName());
        sbuf.append(" - ");
        sbuf.append(event.getFormattedMessage());
        return sbuf.toString();
    }
}
```

### LoggerContext

```java
public class LoggerContext extends ContextBase implements ILoggerFactory, LifeCycle {

  final Logger root;
  private int size;
  private int noAppenderWarning = 0;
  final private List<LoggerContextListener> loggerContextListenerList = new ArrayList<LoggerContextListener>();
  
  private Map<String, Logger> loggerCache;
  
  // getLogger 的位置就在这里，好像并没有从配置文件读取并初始化的过程
  public final Logger getLogger(final String name) {
    if (name == null) {
      throw new IllegalArgumentException("name argument cannot be null");
    }

    // if we are asking for the root logger, then let us return it without
    // wasting time
    if (Logger.ROOT_LOGGER_NAME.equalsIgnoreCase(name)) {
      return root;
    }

    int i = 0;
    Logger logger = root;

    // check if the desired logger exists, if it does, return it
    // without further ado.
    Logger childLogger = (Logger) loggerCache.get(name);
    // if we have the child, then let us return it without wasting time
    if (childLogger != null) {
      return childLogger;
    }

    // if the desired logger does not exist, them create all the loggers
    // in between as well (if they don't already exist)
    String childName;
    while (true) {
      int h = LoggerNameUtil.getSeparatorIndexOf(name, i);
      if (h == -1) {
        childName = name;
      } else {
        childName = name.substring(0, h);
      }
      // move i left of the last point
      i = h + 1;
      synchronized (logger) {
        childLogger = logger.getChildByName(childName);
        if (childLogger == null) {
          childLogger = logger.createChildByName(childName);
          loggerCache.put(childName, childLogger);
          incSize();
        }
      }
      logger = childLogger;
      if (h == -1) {
        return childLogger;
      }
    }
  }
}

//Logger 的函数
class Logger{
  Logger createChildByName(final String childName) {
    int i_index = LoggerNameUtil.getSeparatorIndexOf(childName, this.name.length() + 1);
    if (i_index != -1) {
      throw new IllegalArgumentException("For logger [" + this.name
          + "] child name [" + childName
          + " passed as parameter, may not include '.' after index"
          + (this.name.length() + 1));
    }

    if (childrenList == null) {
      childrenList = new ArrayList<Logger>(DEFAULT_CHILD_ARRAY_SIZE);
    }
    Logger childLogger;
    childLogger = new Logger(childName, this, this.loggerContext);
    childrenList.add(childLogger);
    childLogger.effectiveLevelInt = this.effectiveLevelInt;
    return childLogger;
  }
  // 不知道在哪被调用的
  public synchronized void addAppender(Appender<ILoggingEvent> newAppender) {
    if (aai == null) {
      aai = new AppenderAttachableImpl<ILoggingEvent>();
    }
    aai.addAppender(newAppender);
  }

}

```

### 一个完整的例子

```java
public class KafkaAppender extends AppenderBase<ILoggingEvent> {

    KafkaWriter kafkaWriter;
    KafkaMessageEncoder<ILoggingEvent> kafkaMessageEncoder;
    // @todo make it configurable
    String topic;

    public KafkaAppender(KafkaWriter kafkaWriter, KafkaMessageEncoder messageEncoder, String topicName) {
        this.kafkaWriter = kafkaWriter;
        this.kafkaMessageEncoder = messageEncoder;
        topic = topicName;
    }

    @Override
    protected void append(ILoggingEvent eventObject) {
        String encoded = kafkaMessageEncoder.doEncode(eventObject);
        KeyedMessage<String, String> message = new KeyedMessage<>(topic, encoded);
        kafkaWriter.putMessages(message);
    }

    @Override
    public void start() {
        super.start();
    }

    @Override
    public void stop() {
        kafkaWriter.close();
        super.stop();
    }
}
```

**启动**

```java
    public void addKafkaAppender(MarioContext context) {

        if(!toKafkaEnable) return;

        if (brokers == null || brokers.length == 0)
            throw new IllegalArgumentException("brokers is not defined in config");

        ArrayList<String> brokerList = new ArrayList<>();

        for (String broker: brokers) {
            brokerList.add(broker);
        }

        kafkaWriter = new KafkaWriter<>(new KafkaWriterConfig(brokerList));
        //String hello = "hello";
        //kafkaWriter.putMessages(new KeyedMessage(topicName, hello));

        LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();

        Layout mylayout = new DefaultLayout(context);
        mylayout.start();

        LayoutKafkaMessageEncoder<ILoggingEvent> layoutKafkaMessageEncoder = new LayoutKafkaMessageEncoder<>();
        layoutKafkaMessageEncoder.setLayout(mylayout);
        layoutKafkaMessageEncoder.setContext(lc);
        layoutKafkaMessageEncoder.start();

        KafkaAppender kafkaAppender = new KafkaAppender(kafkaWriter, layoutKafkaMessageEncoder, topicName);
        kafkaAppender.setName("kafkaAppender");
        kafkaAppender.setContext(lc);
        kafkaAppender.start();

        Logger rootLogger = (Logger) LoggerFactory.getLogger(Logger.ROOT_LOGGER_NAME);
        rootLogger.addAppender(kafkaAppender);
    }

```