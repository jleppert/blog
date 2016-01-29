---
layout: post
title: The Prüfer Sequence
description: "Space efficient random trees"
modified: 2016-01-26
tags: [code, npm]
image:
  feature:
---

The [Prüfer Sequence](http://en.wikipedia.org/wiki/Pr%C3%BCfer_sequence) is a cool way of representing a tree of nodes with a sequence of labels. I wrote a javascript version to create arbitrary tree structures for testing node streams for a recent project. Here's an implementation for converting a sequence to an array of connected edges:

{% highlight js %}
function prufer(a) {
  var tree = [];
  var T = Array.apply(null, Array(a.length + 2)).map(function(_, i) { return i; });
  var deg = Array.apply(null, Array(1*T.length)).map(function() { return 1; });
  a.map(function(i) { deg[i]++; });

  for(var i = 0; i < a.length; i++) {
    for(var j = 0; j < T.length; j++) {
      if(deg[T[j]] === 1) {
        tree.push([a[i], T[j]]);
        deg[a[i]]--;
        deg[T[j]]--;
        break;
      }
    }
  }

  var last = T.filter(function(x) { return deg[x] === 1; });
  tree.push(last);

  return tree;
}
{% endhighlight %}

<p data-height="231" data-theme-id="0" data-slug-hash="WbNeYg" data-default-tab="result" data-user="jleppert" class='codepen'>See the Pen <a href='http://codepen.io/jleppert/pen/WbNeYg/'>WbNeYg</a> by Johnathan Leppert (<a href='http://codepen.io/jleppert'>@jleppert</a>) on <a href='http://codepen.io'>CodePen</a>.</p>
<script async src="//assets.codepen.io/assets/embed/ei.js"></script>

Here's the code on [github](https://github.com/jleppert/prufer) and [npm module](https://www.npmjs.com/package/prufer).
