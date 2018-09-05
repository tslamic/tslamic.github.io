---
layout: post
title: "Abstracting APIs"
subtitle: "How I like to abstract my APIs."
author: "Tadej"
comments: true
---
Assume you're developing an [XKCD](https://xkcd.com/) comic browser on Android. The [API](https://xkcd.com/json.html) offers two endpoints:

-  [`info.0.json`](http://xkcd.com/info.0.json), which retrieves the JSON describing the current comic,
- [`comicNumber/info.0.json`](http://xkcd.com/614/info.0.json) retrieves the JSON describing a specific comic.

There are numerous ways to access the above endpoints: either with libraries such as [Retrofit](http://square.github.io/retrofit/), [Volley](https://github.com/google/volley), [Ion](https://github.com/koush/ion), or manually, using e.g. `HttpURLConnection`and [AsyncTasks](https://developer.android.com/reference/android/os/AsyncTask.html), [Services](https://developer.android.com/guide/components/services.html), [Loaders](https://developer.android.com/guide/components/loaders.html) or other building blocks offered by the SDK.

With constant improvements, best practices and new libraries, it's good to not be tied to a particular implementation. A common approach is to abstract the API in an interface:

```java
interface XkcdApi {
  Single<Comic> getLatestComic();
  Single<Comic> getComic(int comicNumber);
}
```

and provide an implementation for it. Using the de facto standard nowadays, Retrofit, the easiest thing to do is to _tweak_ the interface by adding the required annotations:

```java
interface XkcdApi {
  @GET("info.0.json") 
  Single<Comic> getLatestComic();
  
  @GET("{comicNumber}/info.0.json") 
  Single<Comic> getComic(@Path("comicNumber") int comicNumber);  
}
```

Is this a good idea? One could argue the implementation details have crept into an abstraction, defeating its purpose. If you choose to ignore annotations though, it's no different from the original. But suppose we want to add another method:

```java
Single<Comic> getComic(String searchQuery); 
```

and this method needs to access an endpoint with a different base URL than others. 

If our interface contains Retrofit annotations, we're in trouble. We cannot simply add the new method unless we want to do [extra work](https://github.com/square/retrofit/issues/1404#issuecomment-207408548). We could provide a separate interface, e.g. `XkcdApi` and `XkcdSearchApi`, but we then have two separate instances handling API requests, which might not be ideal.

A neater approach could be to revert back to a plain interface, with required API calls all in the same place:

```java
interface XkcdApi {
  Single<Comic> getLatestComic();
  Single<Comic> getComic(int comicNumber);
  Single<Comic> getComic(String searchQuery); 
}
```

and create two Retrofit instances and a facade class to implement the above:

```java
interface RetrofitXkcdApi {
  String BASE_URL = "http://www.xkcd.com";
  
  @GET("info.0.json") 
  Single<Comic> getLatestComic();
  
  @GET("{comicNumber}/info.0.json") 
  Single<Comic> getComic(@Path("comicNumber") int comicNumber);  
}

interface RetrofitSearchApi {
  String BASE_URL = "https://relevantxkcd.appspot.com";

  @GET("process?action=xkcd")
  Single<Comic> getComic(@Query("query") String searchQuery);
}
```

Although the two Retrofit interfaces look uncomfortably similar to the original one, they shouldn't be treated as such - they're an implementation detail, as we cannot build Retrofit instances without them.

Creating a facade couldn't be simpler:

```java
class XkcdApiImpl implements XkcdApi {
  RetrofitXkcdApi api;
  RetrofitSearchApi search;

  XkcdApiImpl(RetrofitXkcdApi api, RetrofitSearchApi search) {
    this.api = api;
    this.search = search;
  }

  @Override
  public Single<Comic> getLatestComic() {
    return api.getLatestComic();
  }
  
  @Override
  public Single<Comic> getComic(int comicNumber) {
    return api.getComic(comicNumber);
  }

  @Override
  public Single<Comic> getComic(String searchQuery) {
    return search.getComic(searchQuery);
  }
}
```

This offers a unified API, is easy to test and mock, but more importantly, it has the flexibility to implement the `XkcdApi` interface in any way we see fit - we could stick to Retrofit, easily replace it with Volley or use different approaches for each method, with minimal code changes.

It's also incredibly easy to turn `XkcdApiImpl` into a repository. Say this is your [Room](https://developer.android.com/topic/libraries/architecture/room.html) interface:

```java
@Dao  
interface ComicDao {  
  @Query("SELECT * FROM comic WHERE num=:comicNumber LIMIT 1")
  Maybe<Comic> select(int comicNumber);
  void insert(Comic comic);
}
```

Then, with minimal changes, caching comic requests is easy:

```java
class XkcdApiImpl implements XkcdApi {
  RetrofitXkcdApi api;
  RetrofitSearchApi search;
  ComicDao comicDao;

  XkcdApiImpl(RetrofitXkcdApi api, 
              RetrofitSearchApi search,
              ComicDao comicDao) {
    this.api = api;
    this.search = search;
    this.comicDao = comicDao;
  }

  // Other code as before.
  
  @Override
  public Single<Comic> getComic(int comicNumber) {
    return comicDao.select(comicNumber)
                   .switchIfEmpty((SingleSource<Comic>) o -> 
                      api.getComic(comicNumber)
                         .doOnSuccess(c -> comicDao.insert(c)));
  }
```
