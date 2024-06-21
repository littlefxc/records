---
{}
---


## 什么是领域事件

![[领域事件.png]]

## 事件命名和基本属性

![[领域事件命名和基本属性.png]]

### 示例

```java
public abstract class DomainEvent extends ApplicationEvent {  
  
    private String eventId;  
    private LocalDateTime occurTime;  
  
    public String getEventName() {  
        return (String) this.source;  
    }  
  
    public DomainEvent(Object source) {  
        super(source);  
        eventId = UUID.randomUUID().toString();  
        occurTime = LocalDateTime.now();  
    }  
  
    public abstract String key();  
  
    public String getEventId() {  
        return eventId;  
    }  
  
    public void setEventId(String eventId) {  
        this.eventId = eventId;  
    }  
  
    public LocalDateTime getOccurTime() {  
        return occurTime;  
    }  
  
    public void setOccurTime(LocalDateTime occurTime) {  
        this.occurTime = occurTime;  
    }  
}
```

## 发布和订阅方式

![[领域事件_发布和订阅方式.png]]
## 事件存储

![[领域事件_事件存储.png]]
## 事件处理的要求

![[领域事件_事件处理的要求.png]]

## 领域事件和大数据分析

![[领域事件_领域事件和大数据分析.png]]