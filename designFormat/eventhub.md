# 实现EventHub

```javascript
class EventHub {
  constructor () {
    this.hub = new Map();
  }
  // 注册回调
  on = (eventName, cb) => {
    if (!this.hub.has(eventName)) {
      this.hub.set(eventName, []);
    }
    let cbs = this.hub.get(eventName);
    cbs.push(cb);
  }
  // 触发事件
  emit = (eventName, params) => {
    if (this.hub.has(eventName)) {
      this.hub.get(eventName).map(cb => cb(params));
    }
  }
  // 取消订阅
  off = (eventName, cb) => {
    if (this.hub.has(eventName)) {
      let cbs = this.hub.get(eventName);
      let index = cbs.findIndex(ele => ele === cb);
      if (index >= 0) cbs.splice(index, 1);
      if (!this.hub.get(eventName).length) this.hub.delete(eventName); 
    } 
  };
} 
```
