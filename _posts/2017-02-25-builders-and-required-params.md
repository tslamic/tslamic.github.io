---
layout: post
title: "Builders and required params"
description: "How to handle builders with required parameters"
tags: [Android, Java, Builder, Design Patterns]
---
Ever since Effective Java 2 came out, the builder pattern became a defacto standard for instantiating objects with multiple parameters.

{% highlight java %}
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
  .calories(100)
  .fat(0)
  .sodium(35)
  .carbohydrate(27)
  .build();
{% endhighlight %}

Required parameters are usually passed in via a constructor, as in the example above, but having too many might defeat the purpose of a builder and spread confusion all around. Can you tell what the two arguments above denote?

A way to avoid this is to provide a no-params constructor:

{% highlight java %}
NutritionFacts cocaCola = new NutritionFacts.Builder() // No required params
  .servingSize(240)
  .servings(8)
  .calories(100)
  .fat(0)
  .sodium(35)
  .carbohydrate(27)
  .build();
{% endhighlight %}

and validate the data whenever a `build()` method is invoked:

{% highlight java %}
public NutritionFacts build() {
  if (servingSize <= 0) {
    throw new IllegalStateException("serving size required");
  }
  if (servings <= 0) {
    throw new IllegalStateException("number of servings required");
  }
  return new NutritionFacts(this);
}
{% endhighlight %}

The above works, but it would be nice to provide a fluid API where it is impossible to invoke `build()` before the required parameters are provided.

Let's describe the builder with a simple interface:

{% highlight java %}
interface Builder {
  Builder servingSize(int size);
  Builder servings(int servings);
  Builder calories(int calories);
  Builder fat(int fat);
  Builder sodium(int sodium);
  Builder carbohydrate(int carbohydrate);
  NutritionFacts build();
}
{% endhighlight %}

Recall the first two methods, `servingSize(int)` and `servings(int)` are required. Could we force the user to start the builder chain with them? 

Suppose we declare two additional interfaces:

{% highlight java %}
// Represents the first required param. Note the return type.
interface ServingSizeBuilder { 
  ServingBuilder servingSize(int size);
}

// Again, note the return type.
interface ServingBuilder {
  Builder serving(int size);
}
{% endhighlight %}

Starting with a `ServingSizeBuilder` instance, you're now forced to start the chain with the two required params and are unable to invoke `build()` before:

{% highlight java %}
servingSizeBuilder.servingSize(someInt)
    .serving(anotherInt)
    // all other methods specified in a Builder.
{% endhighlight %}

We should remove the two required setters methods from the `Builder` interface, as they are described in the two we defined above. We end up with:

{% highlight java %}
interface Builder {
  interface ServingSizeBuilder { 
    ServingBuilder servingSize(int size);
  }
  interface ServingBuilder {
    Builder serving(int size);
  }
  // Note, no servingSize(int size) or serving(int size)
  Builder calories(int calories);
  Builder fat(int fat);
  Builder sodium(int sodium);
  Builder carbohydrate(int carbohydrate);
  NutritionFacts build();
}
{% endhighlight %}

This can be fully appreciated when implemented:

{% highlight java %}
class NutritionFactsBuilder implements Builder,
        Builder.ServingSizeBuilder, Builder.ServingBuilder {
  private int servingSize = 0;
  private int servings = 0;
  private int calories = 0;
  private int fat = 0;
  private int carbohydrate = 0;
  private int sodium = 0;

  // To start the builder chain with required parameters, 
  // we need to provide the following static factory method
  // returning ServingSizeBuilder.
  public static ServingSizeBuilder builder() {
    return new NutritionFactsBuilder();
  }

  private NutritionFactsBuilder() {}

  @Override public ServingBuilder servingSize(int val) {
    servingSize = val;
    return this;
  }

  @Override public Builder serving(int val) {
    servings = val;
    return this;
  }

  @Override public Builder calories(int val) {
    calories = val;
    return this;
  }

  @Override public Builder fat(int val) {
    fat = val;
    return this;
  }

  @Override public Builder carbohydrate(int val) {
    carbohydrate = val;
    return this;
  }

  @Override public Builder sodium(int val) {
    sodium = val;
    return this;
  }

  @Override public NutritionFacts build() {
    return new NutritionFacts(this);
  }
}
{% endhighlight %}

There it is! A builder where required params must be set before `build()` method can be invoked.