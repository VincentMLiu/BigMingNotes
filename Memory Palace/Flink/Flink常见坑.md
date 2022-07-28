# Flink on yarn长任务报错AMRMClientImpl does not update AMRM token properly
Invalid AMRMToken from appattempt

## 说明
Yarn 2.7.0版本之前的bug
详情可参考如下链接：
https://issues.apache.org/jira/browse/YARN-3103

https://www.cnblogs.com/029zz010buct/p/9953041.html

正确代码：
```
private void updateAMRMToken(org.apache.hadoop.yarn.api.records.Token token)
    throws IOException
  {
    org.apache.hadoop.security.token.Token<AMRMTokenIdentifier> amrmToken = new org.apache.hadoop.security.token.Token(token.getIdentifier().array(), token.getPassword().array(), new Text(token.getKind()), new Text(token.getService()));
    
    UserGroupInformation currentUGI = UserGroupInformation.getCurrentUser();
    currentUGI.addToken(amrmToken);
    amrmToken.setService(ClientRMProxy.getAMRMTokenService(getConfig()));
  }
```
在构建完了token之后，才会更新服务。


错误代码：
```
private void updateAMRMToken(org.apache.hadoop.yarn.api.records.Token token)
    throws IOException
  {
    org.apache.hadoop.security.token.Token<AMRMTokenIdentifier> amrmToken = new org.apache.hadoop.security.token.Token(token.getIdentifier().array(), token.getPassword().array(), new Text(token.getKind()), new Text(token.getService()));
    
    amrmToken.setService(ClientRMProxy.getAMRMTokenService(getConfig()));
    UserGroupInformation currentUGI = UserGroupInformation.getCurrentUser();
    if (UserGroupInformation.isSecurityEnabled()) {
      currentUGI = UserGroupInformation.getLoginUser();
    }
    currentUGI.addToken(amrmToken);
  }

```
在增加token之前，先设置了service，导致上下文发生变化，就导致了多个不同的token的产生，后续如果选择了特定的token，就会报错。

## 解决方案
依赖或环境中将Yarn版本升级至2.7.0以上