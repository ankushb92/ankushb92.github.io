---
layout: post
title: "Simple async cache in C#"
tags: [ programming, C#, concurrency ]
full-width: true
---

In this post I want to talk about building an async cache in C# that fulfils the following requirements:
1. The cache handles read requests to a database
2. Assume that there are no writes to the database, so no need to worry about cache invalidation
3. Multi-threaded: Multiple callers are calling from multiple threads concurrently
4. Non-blocking: No sleeps, no locks
5. Avoid making requests to the database while there's an identical outstanding request

### Code outline
{% highlight csharp linenos %}
interface IDataFetcher
{
    // Performs the read from the database
    Task<string> ReadDataAsync(string key);
}

class Cache
{
    IDataFetcher fetcher;

    async Task<string> GetDataAsync(string key, bool force)
    {
        // to do
        // if key is not present in cache, read from fetcher
        // if force is set, read from fetcher (even if key is present in cache)
    }
}
{% endhighlight %}

### Solution

{% highlight csharp linenos %}
using System.Collections.Concurrent;
using System.Threading.Tasks;

interface IDataFetcher
{
    // Performs the read from the database
    Task<string> ReadDataAsync(string key);
}

class Cache
{
    IDataFetcher fetcher;

    static readonly ConcurrentDictionary<string, string> dataCache = new ConcurrentDictionary<string, string>();
    static readonly ConcurrentDictionary<string, Task<string>> taskCache = new ConcurrentDictionary<string, Task<string>>();

    async Task<string> GetDataAsync(string key, bool force)
    {
        // When force is 'false' AND dataCache contains the key
        if (force == false && dataCache.ContainsKey(key)) {
            dataCache.TryGetValue(key, out string cachedValue);
            return cachedValue;
        }
        // When force is 'true' OR dataCache does not contain the key
        else {
            if (taskCache.ContainsKey(key) == false) {
                taskCache.TryAdd(key, fetcher.ReadDataAsync(key));
                // TryAdd() may return 'false' but that's okay because that means the key already exists
            }
            taskCache.TryGetValue(key, out Task<string> cachedTask);

            // Await on cachedTask until we have a fresh value
            string freshvalue = await cachedTask;

            // Update caches
            dataCache.TryAdd(key, freshvalue);
            taskCache.TryRemove(key, out Task<string> _);

            return freshValue;
        }
    }
}
{% endhighlight %}

#### Notes on the implementation

* I created two caches: `dataCache` and `taskCache`. `dataCache` caches results from `ReadDataAsync()`. `taskCache` caches the last outstanding request (`Task`) made to `ReadDataAsync()` (this fulfils requirement #5 defined above).<br/>
* The usage of `ConcurrentDictionary` is to ensure that `GetDataAsync` can be called from multiple threads safely. (fulfils requirement #3).<br/>
* The cache is non-blocking (requirement #4).
