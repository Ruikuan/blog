# 使用 Perfview 收集 events 时如何带上 callstack？

在使用 Perview 来按照 Providers 收集 events 时，在 Provier 后面加上 `:@StacksEnabled=true` 即可收集到对应的 callstack。

```
@StacksEnabled=true
```

![perview collect callstack](../Content/perfview_callstack.png)
