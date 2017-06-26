---
layout: page
title : "About"
weight: 5
---
{% include JB/setup %}

I work as a senior Android engineer at [58](http://about.58.com/home/), focused mainly on mobile development.

All stuff here express myself which has nothing to do with my employer;  

## Contact

Email：<me@avenwu.net>  
Github:[https://github.com/avenwu/](https://github.com/avenwu/)

## Projects

I'v also made some projects in spare time as you can see listed below; Feel free to star/fork/follow.

Java NIO: [http://java-nio.avenwu.net](http://java-nio.avenwu.net)  
YoYo GitHub: [http://avenwu.net/yoyo](http://avenwu.net/yoyo)  
Support: [https://github.com/avenwu/support](https://github.com/avenwu/support)  
Cnblogs: [http://avenwu.net/cnblogs](http://avenwu.net/cnblogs)  

---

<script src="http://code.jquery.com/jquery-2.1.4.js"></script>
<script src="http://unslider.com/unslider/dist/js/unslider-min.js"></script>

<link rel="stylesheet" href="http://unslider.com/css/reset.css">
<link rel="stylesheet" href="{{ site.baseurl}}/css/slider.css">
<link rel="stylesheet" href="http://unslider.com/unslider/dist/css/unslider.css">
<link rel="stylesheet" href="http://unslider.com/unslider/dist/css/unslider-dots.css">
<div class="demos">
	<div class="demo">
		<ul >
			<li>
				<h2 id="java-nio">Java NIO(2016)</h2>
				<img src="{{ site.baseurl }}/assets/images/java-nio-720.png" onclick="javascript:openUrl('http://java-nio.avenwu.net')"/><br /><br />
				<p>
				My first translation work about Java NIO. It cost me nearly two months to finish all the stuff. The book is now hosted on <a href="http://java-nio.avenwu.net">GitBook</a> where you can choose to read online or download the PDF file.</p>
			</li>
			<li>
				<h2 id="yoyo-github2016">YoYo Github（2016~）</h2>
				<img src="{{ site.baseurl }}/assets/images/yoyo-1024-500.png" onclick="javascript:openUrl('http://avenwu.net/yoyo')"/><br /><br />
				<p>
				This project has just been started recently in April 2016, YoYo is desired to be a well-designed Github application with material style from Google；</p>
			</li>
			<li>
				<h2 id="support2015">Support（2015~）</h2>
				<img src="{{ site.baseurl }}/assets/images/support-1024-500.png" onclick="javascript:openUrl('https://github.com/avenwu/support')"/><br /><br />
				<p>
				Custom Android support library, include some useful utils and widget.（#￣▽￣#）</p>
			</li>
			<li>
				<h2 id="cnblogs-2014">Cnblogs (2014~)</h2>
				<img src="{{ site.baseurl }}/assets/images/cnblogs-1024-500.png" onclick="javascript:openUrl('http://avenwu.net/cnblogs')"/><br /><br />
				<p>
				Another cool application for Cnblogs, it’s been developed since 2014 and has been stable with most used features;  <br />
				There must be bugs（╯－_－）╯╧╧, any bug reports will be appreciated(￣ . ￣);</p>
			</li>
		</ul>
	</div>

	<script>$('.demo').unslider({
		autoplay: true, 
		arrows: false,
		infinite: true
	});
	function openUrl(url) {
	  var win = window.open(url, '_blank');
	  win.focus();
	}
	</script>

</div>
