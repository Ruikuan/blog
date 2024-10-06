# Access `ConnectionItems` in `HttpContext`

Sometimes you need to share some data connection-wide, but base on the handling of request, in such case, you can just access `ConnectionContext.Items` to store and fetch the shared data.
It probably only works on Kestrel.

```cs
IDictionary<object, object> connectionItems = httpContext.Features.Get<IConnectionItemsFeature>().Items;
connectionItems.Add("key", "value");
```

