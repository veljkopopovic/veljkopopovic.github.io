---
layout: post
title: "Grunting from command line"
date: 2015-01-28 22:06:00 +0100
comments: false
---


# Grunting around

Have you ever tried [Grunt](http://gruntjs.com/)? Oh, you should give it a try, it's awesome. Once you catch up with basics, how to configure tasks and run grunt file, it will become very usefull tool. Simple, configurable with large community, lot of plugins (they call them tasks) ... Basically, one can do almost anything with grunt.

Let me consider very simple task: you want to develop some JavaScript web component. So, once you're satisfied how code looks, run jshint, concat all files into one in given order and do minify concated file. Let's make it simple, we could complicate it with much more tasks, but this, simple three-task program will suffice. Oh, I said program? Yes, indeed it is ... So, what do you do? Since you know about grunt, you will write a simple grunt file, something like this:


```javascript
///TODO
```


As you can see from code, we used several grunt plugins ///TODO:list them, so all we have to do is to create a package.json and ```npm install --save``` those files and here we go, run grunt and that's it. At this point it is natural to ask: Why do I have to create package.json, I am not developing node module? Answer is simple: with no package.json you will be unable to persist which packages should be used and there will be no way to pass this project to any other developer. Other developer will have to do ```npm install``` before running ```grunt```. Still seems bit itchy if you ask me. Let me remind you: we are trying to produce a web component, not a node module. Ok, so you decide to let it be: you do create package.json with requirements and you do instruct other team members to do npm install before grunt. Later on, you get another project, new web component do develop. What you can do: copy Gruntfile.js and reconfigure it and, yes, you have to, ```npm init && npm install --save same_requirements```. After a while, you end up with hundred web components and hundred installs of quite same npm modules and hundred copies of quite same grunfile with bit different settings, most often with different file lists you want to concat and so on. And you decide do add something like concat separator (for example, put the original file name in concated file before everything, in comment).
Conclusion: it's not as reusable as we would like ...

At this point, most developers will take one of these paths:  let it be, and do update each and every grunt file you have or simply give up grunt and try to find something bit better. One way or another, one problem remains: your job becomes less automated then expected.

There is a third way: why don't we reduce problem to a minimal set of data required by grunt tasks (configurations) on one side and grunt task and grunt requirements on the other. A bit of juggling with nodejs, grunt, one new tool project and you will end up as happy as possible and will keep on grunting around.


### Grunt programs
