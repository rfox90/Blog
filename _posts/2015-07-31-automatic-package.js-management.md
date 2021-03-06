---
layout: post
title: "Automatic Package.js Management"
date: 2015-07-31 00:36:03
author: Richard Fox
summary:
categories:
- blog
img: 
thumb:
tags:
- meteor 
comments: true
---

When building a large meteor application a common approach is to modularize the application into packages. This is great as it allows developers much finer control over their file load order which is often causes problems with an application that doesn't use packages. 

We however lose the ability to refactor and dynamically manage the source of an application/package. The Package.js file has to be constantly changed and modified to keep everything loaded and in the correct order.

This killed some of the Meteor magic that first brought me to developing on the platform.

There is some talk of [fixing this in the future](https://trello.com/c/mHK2dpr5/68-new-way-of-defining-packages-apps-and-controlling-file-load-order) but for now I wanted to solve it.

##Globs to the Rescue
![oh my glob](https://raw.githubusercontent.com/isaacs/node-glob/master/oh-my-glob.gif)

Something that instantly came to mind were [globs](https://en.wikipedia.org/wiki/Glob_%28programming%29) especially coming from projects that used other build systems such as [gulp](http://gulpjs.com/) and [grunt](http://gruntjs.com/).

Globs allow you to specify patterns similar to Regular Expressions that match sets of files in a system. 

For example `client/**` would match everything in the client folder of an application or package.

The goal here is to use globs and package.js to hand some of the load order processing back to the computer and instead focus on building the application.

To use this I restructured my application into a local core packages. This way all of Package.js's features are available to the application and I can control build order. The application is only comprised of one core package currently but splitting it up and continueing to use this technique should be easy.


##Globs in Package.js

I thought using globs in a Package.js might be a bit tricky but thankfully due to [Npm.require](http://docs.meteor.com/#/full/Npm-require) it's quite simple.

Anywhere in your Package.js you can require the [glob module from npm](https://www.npmjs.com/package/glob)

{% highlight js %}
var globApi = Npm.require('glob');
{% endhighlight %}

and then you can use any of the module's api.

###glob.sync
I decided to use [glob.sync](https://www.npmjs.com/package/glob#glob-sync-pattern-options) as my method of choice from glob's api as it preserves the Meteor feel by not relying on callbacks. 

It returns an array of strings as results where each element is a file or folder matched by a glob. This is perfect as it can be plugged into [api.addFiles](http://docs.meteor.com/#/full/pack_addFiles).

So to using the above example of everything in the client folder we can include that with:

{% highlight js %}
api.addFiles(glob.sync('client/**'),['client']);
{% endhighlight %}
   
###Excluding Folders
The above glob unfortunately includes folders. You can pass options to the glob.sync method to ignore folders, but this didn't seem to work in meteor.

So given the tree:

- client
  - folder
     - file.js

The resulting array from the glob would be:

{% highlight js %}
['client/folder','client/folder/file.js']
{% endhighlight %}


To avoid this we need to edit our glob a little bit. It supports a regular expression like syntax so:

{% highlight js %}
'client/{**/*,*}.+(js|html)'
{% endhighlight %}

Will include any files with the .js or .html extension(I'm not using coffee script for now) anywhere within the client folder and will exclude folders.

###CWD and Package.js

Glob defaults its searching to the CWD for the application and for the meteor build process this is unfortunately not anywhere near the Package's directory. 

Thankfully due to the package being within the application we can get to its director quite easily if we know its name. You can then pass that in an options object to glob:

{% highlight js %}
var path = Npm.require('path');
var packageName = 'core';
options.cwd = path.join(process.cwd() + '/packages/'+packageName+'/');
{% endhighlight %}

This won't work for external packages that are not in the packages directory but i'm looking at ways to support this. 


###Putting it all Together

To wrap these concepts together and provide a nicer way of using it I wrote the following function:
{% highlight js %}
function registerGlob(glob,target) {
	if(target === 'both') {
		target = ['client','server'];
	} else {
		var targetArray = [];
		targetArray[0] = target;
		target = targetArray;
	}
	var options = {};		
	options.cwd = path.join(process.cwd() + '/packages/test/');
	var globResults = globApi.sync(glob,options);
	api.addFiles(globResults,target);
}
{% endhighlight %}
Which i scoped inside of `Package.onUse` so that api is defined.

I also disliked the way api.addFiles requires the architectures to be specified in an array e.g. `['client','server']`. It allows fine control over where your package files go but for this glob approach I wanted something easier to use.
So by using registerGlob I can just specify the location as a single string:
{% highlight js %}
registerGlob('common/config/{**/*,*}.+(js|html)','both');
registerGlob('server/config/{**/*,*}.+(js|html)','server');
registerGlob('client/config/{**/*,*}.+(js|html)','client');
{% endhighlight %}


I've posted a complete example Package.js which uses this to a [github gist](https://gist.github.com/rfox90/fa1a7c7a85f6ebc8bda1).

The approach seems to work well in development but I would probably move it to a staticly generated file for a production environment.

Hopefully we'll see something easier to use coming in future meteor versions.

Let me know what you think!


