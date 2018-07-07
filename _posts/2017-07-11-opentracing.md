---
layout: post
tags : [分布式跟踪]
title: opentracing

---

OpenTracing: 分布式追踪系统标准

## 概念

#### trace

代表了一个事务或者流程在（分布式）系统中的执行过程, 是多个span组成的一个有向无环图

#### span

具有**开始时间**和**执行时长**的逻辑运行单元

span 之间的关系

* ChildOf
* FollowsFrom: 父级节点不以任何方式依赖他们子节点的执行结果

span 的属性

* operationName

* 开始时间

* 执行时长

* logs 
  每个span可以进行多次Logs操作，每一次Logs操作，都需要一个带时间戳的时间名称，以及可选的任意大小的存储结构
  对象数组, 对象包含timestamp

* tags
  对象数组, 每个对象中是k和v, 键值对. tags没有时间戳
  Span的tag不会进行传输，因为他们不会被子级的span继承

* SpanContext



jaeger-client中Span的构造器:
```javascript
function Span(tracer, operationName, spanContext, startTime, references) {
    _classCallCheck(this, Span);

    this._tracer = tracer;
    this._operationName = operationName;
    this._spanContext = spanContext;
    this._startTime = startTime;
    this._logger = tracer._logger;
    this._references = references;
    this._logs = [];
    this._tags = [];
}
```

#### SpanContext

SpanContext代表跨越进程边界，传递到下级span的状态, 包含trace_id, 自己的span_id, parentId等

jaeger-client中SpanContext的构造器:

```javascript
function SpanContext(traceId, spanId, parentId, traceIdStr, spanIdStr, parentIdStr) {
    .......
    _classCallCheck(this, SpanContext);

    this._traceId = traceId;
    this._spanId = spanId;
    this._parentId = parentId;
    this._traceIdStr = traceIdStr;
    this._spanIdStr = spanIdStr;
    this._parentIdStr = parentIdStr;
    this._flags = flags;
    this._baggage = baggage;
    this._debugId = debugId;
    this._samplingFinalized = samplingFinalized;
}
```

#### Baggage

Baggage是存储在SpanContext中的一个键值对(SpanContext)集合, 它会在一条追踪链路上的所有span内全局传输，包含这些span对应的SpanContexts

Baggage vs. Span Tags

* Baggage在全局范围内, 跨进程传输数据; tag不会进行传输, 不被继承
* span的tag可以用来记录业务相关的数据，并存储于追踪系统中; Baggage中的非业务数据不一定需要存储


#### logs

每一次日志记录都会包含一个时间戳, 以及多个key:value阈.

logs 的json表示形式:

```javascript
logs: [
    {
        timestamp: 1500795177508447,
        fields: [
            {
                key: "event",
                type: "string",
                value: "HTTP request received"
            },
            {
                key: "level",
                type: "string",
                value: "info"
            },
            {
                key: "method",
                type: "string",
                value: "GET"
            },
            {
                key: "url",
                type: "string",
                value: "/customer?customer=123"
            }
        ]
    },
    {
        timestamp: 1500795177508469,
        fields: [
            {
                key: "event",
                type: "string",
                value: "Loading customer"
            },
            {
                key: "customer_id",
                type: "string",
                value: "123"
            },
            {
                key: "level",
                type: "string",
                value: "info"
            }
        ]
    }
]
```
---

## Inject and Extract

Inject: SpanContexts -> Carrier(跨进程通讯数据, 如http header)

Extract: 从Carrier提前出SpanContexts

---

## 平台无关API语义

#### Span Interface

* Get the Span's SpanContext
* Finish
* Set a key:value tag on the Span
* Add a new log event

* Set a Baggage item

    `span.setBaggageItem('where', 'zhongguo');`

    在http header中通过新增header 传播: `'uberctx-where': 'zhongguo'`


---
Node.js 中的一个span:
```javascript
{ traceIdLow: <Buffer b8 eb 79 de 91 d5 87 1d>,
traceIdHigh: <Buffer 00 00 00 00 00 00 00 00>,
spanId: <Buffer b8 eb 79 de 91 d5 87 1d>,
parentSpanId: <Buffer 00 00 00 00 00 00 00 00>,
operationName: 'getUsersAsync',
references: [],
flags: 1,
startTime: <Buffer 00 05 54 05 69 28 e2 50>,
duration: <Buffer 00 00 00 00 00 3a 0f 48>,
tags:
[ { key: 'sampler.type',
vType: 'STRING',
vStr: 'const',
vDouble: 0,
vBool: false,
vLong: <Buffer 00 00 00 00 00 00 00 00>,
vBinary: <Buffer 00 00 00 00 00 00 00 00> },
{ key: 'sampler.param',
vType: 'BOOL',
vStr: '',
       vDouble: 0,
       vBool: true,
       vLong: <Buffer 00 00 00 00 00 00 00 00>,
       vBinary: <Buffer 00 00 00 00 00 00 00 00> },
     { key: 'page',
       vType: 'DOUBLE',
       vStr: '',
       vDouble: 20,
       vBool: false,
       vLong: <Buffer 00 00 00 00 00 00 00 00>,
       vBinary: <Buffer 00 00 00 00 00 00 00 00> },
     { key: 'pageSize',
       vType: 'DOUBLE',
       vStr: '',
       vDouble: 78,
       vBool: false,
       vLong: <Buffer 00 00 00 00 00 00 00 00>,
       vBinary: <Buffer 00 00 00 00 00 00 00 00> } ],
  logs: [ { timestamp: <Buffer 00 05 54 05 69 29 20 d0>, fields: [Array] } ] }
```

