---
layout: post
title: "Date input placeholder hack"
date: 2021-02-05
comments: false
sharing: true
categories:
  - util
  - web
description: >-
  The input type date is a very nice control introduced in html5. The problem is that it is very limited at this point and features such as placeholder is not available. Thus, let's hack it!
---

The input type date is a very nice control introduced in HTML5. The problem is that it is very limited at this point and features such as placeholder may not available. Regardless, you may use it successfully if you are willing to make some css adjustments. In this post I will show how to add a placeholder to a date editor using minimal javascript and css. For the demo below I'm using vanilla css and vuejs just because it is so easy to use.

Since the placeholder attribute is not supported yet, we can use the :before pseudo element of the input. However, we only want to show the placeholder only if the input is empty and not focused. We can control the empty/not empty via a simple binding that checks the date data property as shown below.

```js
var mySimpleApp = new Vue({
  el: '#mainapp',
  data: {
  	date: ""
  },
  template: `<input type="date" v-model="date" placeholder="Type a date" :class="{'empty': !date, 'notempty': date}" />`
})
```

Most of the magic happens in CSS though:

```css
/* HACK: When the element is empty, we want to hide the masked date input. We achieve that by changing the input color to transparent */
input[type="date"].empty {
  color: transparent;
}
/* When the input is not empty or focused, we want the color back */
input[type="date"].notempty,
input[type="date"].empty:focus {
  color: #495057;
}
/* When the input is not empty or focused, we want the pseude :before element to have no content */
input[type="date"].notempty:before,
input[type="date"].empty:focus:before {
  content: "";
}
/* When the input is empty and not focused (implict assumption here), we want the pseude :before element to have the content of the placeholder attribute. We also want to put the color back in the input */
input[type="date"].empty:before {
  content: attr(placeholder);
  color: #6c757d;
}
```

That's about it. You can see a working demo in https://jsfiddle.net/cfbpso5v/3

Cheers, Lucas
