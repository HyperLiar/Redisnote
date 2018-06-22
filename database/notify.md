# Notify
## 1. keyspaceEventsStringToFlags
```
int keyspaceEventsStringToFlags(char *classes)

*p = classes
while((c = *p++) != '\0')
按c的值 flags |= NOTIFY_XXX
即识别classes对应的flags
不识别的场合返回-1
```
## 2. keyspaceEventsFlagsToString
```
sds keyspaceEventsFlagsToString(int flags)

flags &与运算，为真的场合 res= sdscatlen(res, "", 1)
```
## 3. notifyKeyspaceEvent
```
void notifyKeyspaceEvent(int type, char *event, robj *key, int dbid)

event: c string 代表事件名称
key: redis object 代表key名称
dbid: key存在的dbid

如果有module对事件有相应需要通知module系统

pubsubPublishMessage(chanobj, eventobj)
pubsubPublishMessage(chanobj, key)
```
