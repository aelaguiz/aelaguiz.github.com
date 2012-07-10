--- 
permalink: /yapsy-pickle-nope/index.html
categories: []

title: YAPSY + Pickle = Nope
layout: post
published: true
---
I modified my application to use YAPSY (<a href="http://yapsy.sourceforge.net/)">http://yapsy.sourceforge.net/)</a> for the dynamic loading of my &ldquo;domain specific&rdquo; code
and everything was beautiful when running locally. Once I turned on parallel executing using pp I started getting these really annoying errors:

{% highlight python %}
File "/usr/lib/python2.7/pickle.py", line 755, in save_global (obj, module, name))
pickle.PicklingError: Can't pickle : it's not found as __builtin__. MyClass
{% endhighlight %}


<p>   
I ended up tracking the error down down by instrumenting pickle.py (dirty but works):
   </p>

{% highlight python %}
def save_global(self, obj, name=None, pack=struct.pack):
	write = self.write
        memo = self.memo
 
        if name is None:
		name = obj.__name__
		module = getattr(obj, 'module', None)

	print 'Module is', module

	if module is None:
		module = whichmodule(obj, name)

        try:
		print 'Importing',module
		__import__(module)
		mod = sys.modules[module]
		print 'mod is', mod
 
		klass = getattr(mod, name)
		print 'klass', klass
        except (ImportError, KeyError, AttributeError):
		raise PicklingError( "Can't pickle %r: it's not found as %s.%s" %  (obj, module, name))
{% endhighlight %}

The "Module is" print would always print \_\_builtin\_\_, which of course made the error message start to make sense:

{% highlight python %}
  pickle.PicklingError: Can't pickle : it's not found as __builtin__.MyClass
{% endhighlight %}

Basically pickle isn't finding the class in the \_\_builtin\_\_ module. It should not of course be in the \_\_builtin\_\_ module. Looking at
  the YAPSY loader I see why:

{% highlight python %}
   try:
    candidateMainFile = open(candidate_filepath+'.py','r')
    exec(candidateMainFile,candidate_globals)
{% endhighlight %}


I'm guessing that since the loading is occuring by running an exec on the plugin source, the code is getting getting loaded into the \_\_builtin\_\_ module. 
Anyhow, I'm removing YAPSY actually since for my purposes I really don't need an actual plugin framework and \_\_import\_\_(mod) works just fine.
