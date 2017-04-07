---
layout: post
title: "Parsing text into typed objects using RegEx named groups"
date: 2017-4-6
comments: true
sharing: true
categories: util
---

A little while ago, I had to build a desktop app that received input from devices which I had no control of. Although each implementation had similar requirements, the devices connected to the app were slightly different and so were the inputs received from them. I've looked for other similar implementations and found that some people implemented device specific parsers. Furthermore, they used configuration to select the correct parser. I think there is at least one issue with this approach: any change to the known inputs would require the application to change as well. Maybe there is a better way...

The trick I want to demonstrate relies on named group capturing from Regular Expression and a little fiddling with reflection.

> Note: this technique works great for most scenarios but if you are looking for a super high performance solution, this is not for you.

Alright! Let's see some code.

<pre class="brush:csharp">
public static TData ParseToObject&lt;TData&gt;(this string data, string regularExpression) where TData : new()
{
    if (string.IsNullOrEmpty(data)) throw new ArgumentNullException(nameof(data));
    if (string.IsNullOrEmpty(regularExpression)) throw new ArgumentNullException(nameof(regularExpression));

    // create an instance of the TData class
    var result = new TData();

    // find all public properties of the TData class
    var properties = result.GetType().GetProperties(BindingFlags.Public | BindingFlags.Instance);

    // Regex match and get the captured groups
    var groups = Regex.Match(data, regularExpression).Groups;

    foreach (var propertyToSet in properties)
    {
        // If there is a property called RawInput, set that with the original text value. 
        // This can be quite handy for troubleshooting.
        if (propertyToSet.Name.ToUpper(CultureInfo.InvariantCulture) == "RAWINPUT")
        {
            propertyToSet.SetValue(result, data, null);
            continue;
        }

        // find group for property
        var group = groups[propertyToSet.Name];
        var groupValue = group.Success ? group.Value : null;

        // skip setting property if group has no value, doesn't exit or property is not value type
        if ((string.IsNullOrEmpty(groupValue) || !propertyToSet.PropertyType.IsValueType) 
        propertyToSet.PropertyType != typeof(string)) continue;

        // convert value and set property
        var value = propertyToSet.PropertyType.IsEnum
            ? Enum.Parse(propertyToSet.PropertyType, groupValue)
            : Convert.ChangeType(groupValue, propertyToSet.PropertyType, CultureInfo.InvariantCulture);

        propertyToSet.SetValue(result, value, null);
    }

    return result;
}
</pre>

So with above code you can, for instance, parse a string that represents Product and Price into a typed object:

<pre class="brush:csharp">
public class ProductPrice
{
    public string ProductCode { get; set; }
    public decimal Price { get; set; }
}

// 123456 is the product and 22.2 is the price
// I added the . just to show how easy it is to 
// match but not capture a piece of text
"123456 . 22.2".ParseTo&lt;ProductPrice&gt;(
    "(?&lt;ProductCode&gt;\\d{6}) \\. (?&lt;Price&gt;\\d{2}\\.\\d)");

// the result of the above is an instance of ProductPrice 
// class with ProductCode = 123456 and Price = 22.2
</pre>

I find this most useful when you don't really have control of the input, maybe it is a device input or a legacy system that just drop you some info. Regardless, this can save a bunch of time when the input changes or a new device needs to be supported.

You can find samples [at my github repo](https://github.com/jlucaspains/BlogSamples).

Happy coding!