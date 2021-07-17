策略模式
```js
form.onsubmit = function(){
  if(form.username.value === ''){
    alert('用户名不能为空')
    return;
  }
  if(form.password.value.length < 6){
    alert('密码长度不能小于6位')
    return;
  }
  if(!/(^1[3|5|8][0-9]{9}$)/.test(form.phone.value)){
    alert('手机号格式不正确')
    return;
  }
}
```
不管是`if...else...`还是`switch...case...`逻辑多了都很难维护

关键在于**把校验策略解偶成可以单独注册，然后建一个可以执行校验的类**，解决上面的问题

注册校验策略的对象
```js
const strategies = {
  empty(value, errMsg){
    if(value.length === 0){
      return errMsg;
    }
  },
  minLength(value, len, errMsg){
    if(value.length < len){
      return errMsg;
    }
  },
  isMobile(value, errMsg){
    if(!/(^1[3|5|8][0-9]{9}$)/.test(value)){
      return errMsg;
    }
  }
}
```
校验类
```js
class Validator {
  constructor(){
    this.rules = [];
  }
  add(elem, rule, err){
    const args_arr = rule.split(":");
    this.rules.push(()=>{
      const handler = args_arr.shift();
      args_arr.unshift(elem.value);
      args_arr.push(err);
      return strategies[handler].apply(elem, args_arr)
    })
  }
  start(){
    let errmsg = []
    for(let i = 0; i < this.rules.length; i++ ){
      const err = this.rules[i]();
      if(err){
          errmsg.push(err)
      }
    }
    return errmsg.join(",");
  }
}
```
缺点比较少，主要是提高代码复杂度（一个`if...else...`能解决的事情拆成了复杂结构）。
