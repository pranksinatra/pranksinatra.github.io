---
title: WordPress Component Design with Timber & Twig
date: 2017-05-08 20:21:00 -04:00
description: "(under construction)"
---

At Goshen College, we use the [Timber framework](https://www.upstatement.com/timber/) for Wordpress themes. Timber relies on [Twig](https://twig.sensiolabs.org/) for templating, which is {% raw %}{{&nbsp;similar.in&nbsp;\|&nbsp;syntax&nbsp;}}{% endraw %} to other templating engines like Handlebars (JavaScript) or Jinja (Python).

In my experience, **learning Twig isn't the hard part.** The Timber docs do a good job of explaining how to pass a PHP array to a Timber template and use the many nifty Wordpress shortcuts that Timber provides. 

But good template design is hard. I don't claim to be an expert by any means, but just want to share some templates I've iterated on many times and today make developing performant Wordpress sites significantly easier.

## Basic Component Structure
You'll want to put your CSS first, your HTML second, and your JavaScript last. This way your HTML doesn't show up before it's styled and you don't need to wait for the DOMContentLoaded event (that's $(document).ready() in jQuery land) for your JavaScript to be able to reference your HTML.

I generally use a random id for each component too, just to make them easier to reference in JS.

```php
function my_shortcode($atts) 
{
	// shortcode logic...

	return Timber::compile('component.twig', [
		'content' => '...'
		'pathToCSS' => '...'
		'pathToJS' => '...'
	]);
}
```

{% raw %}
```twig
<link href="{{ pathToCSS }}" rel="stylesheet" />

{% set id = 'component--' ~ random() %}
<figure id="{{ id }}" class="component">
  {{ content }}
</figure>

<!-- Pardon my (awful) blocking synchronous JS. That's not what this article is about! -->
<script src="{{ pathToJS }}"></script>
<script>
	initializeComponent( document.querySelector('#{{ id }}') );
</script>
```
{% endraw %}

Most components should be written in such a way that they can coexist on the same page. In the above example, I really just need to load the CSS and JS file once, not every time the component shows up on the page!

One way of keeping track of this is through a custom Twig filter. Let's call it *first_on_page*, have it accept one argument (the name of the component), and return True the first time it's called and False every time after that. 

```php
$twig->addFilter(new Twig_SimpleFilter('first_on_page', function ( $componentName ) 
{
	global $myComponentsOnPage;

	if ( !isset($myComponentsOnPage) ) $myComponentsOnPage = ',';

	// Components is not already on this page. Add it to global var.
	if ( strpos($myComponentsOnPage, ','. $componentName .',') === false ) {
		$myComponentsOnPage .= $componentName . ',';
		return true;
	}

	return false;
}));
```
There are probably *classier* ways of doing this (how about that pun!), but a global variable gets the job done here.

{% raw %}
```twig
{% if 'myComponent'|first_on_page %}
	<link href="{{ pathToCSS }}" rel="stylesheet" />
{% endif %}

{% set id = 'component--' ~ random() %}
<figure id="{{ id }}" class="component">
  {{ content }}
</figure>

{% if 'myComponent'|first_on_page %}
	<script src="{{ pathToJS }}"></script>
{% endif %}

<script>
	initializeComponent( document.querySelector('#{{ id }}') );
</script>
```
{% endraw %}

## Critical CSS and Asynchronous Stylesheets

Critical CSS (inlining a minimal set of above-the-fold styles) can dramatically improve perceived performance, especially on high latency connections. 

<em>I've also heard eating salad every day will dramatically improve my perceived healthiness, but that doesn't mean I'm doing it.</em>

I get it: transitioning from a huge *style.css* file in your <code><head></code> to a bunch of smaller CSS files, each split into critical and non-critical sections can be tricky! There are tools that will automatically generate critical CSS for you (e.g. [Penthouse](https://github.com/pocketjoso/penthouse), [critical](https://github.com/addyosmani/critical)), but I found that generating HTML files for these tools to operate on was error-prone. Ultimately, I went with a DIY solution that split up my CSS based on comments: [postcss-critical-split](https://github.com/mrnocreativity/postcss-critical-split) by [Ronny Welter](https://github.com/mrnocreativity). The silver lining: I got *very* well acquainted with the styles on my site and was able to remove a lot of unused rules.

At the end of the day I we have 3 files. Let's call them:

- Original
- Critical
- Asynchronous

Where Original = Critical + Asynchronous

Then we can inline the Critical CSS in the <code><head></code> and use a JavaScript library called [loadCSS by Filament Group](https://github.com/filamentgroup/loadCSS) to load Asynchronous CSS. Chances are we'll screw something up in the process so it's handy to have the Original CSS available to compare against. 

Here's a Twig template that deals exclusively with stylesheet loading. I'm assuming our 3 files are in the same folder, called *style-original.css*, *style-critical.css*, and *style-async.css*, and we've created a *css_uri* Twig filter that simply converts a filename (e.g. "style-original") into a path (e.g. "/wp-content/themes/your-theme/styles/dist/style-original.css") and a *css_inline* filter which simply outputs the contents of a CSS file.

{% raw %}
```twig
{% spaceless %}

{% set path = name|css_uri %}
	
{# Critical (inline) CSS paired with asynchronous stylesheet #}
{% if type == 'critical+async' %}
	{% if not omitCriticalCSS %}
		{# Load critical (inline) CSS part #}
		<style>{{(name ~ '-critical')|css_inline}}</style>
		{# Omit async CSS part if onlyCriticalCSS is specified #}
		{% if not onlyCriticalCSS %}
			<link rel="preload" href="{{path}}" as="style" onload="this.rel='stylesheet'">
			<noscript><link rel="stylesheet" href="{{path}}"></noscript>
		{% endif %}
	{% else %}
		{# Load compelete synchronously (for debugging) if omitCriticalCSS is specified #}
		<link rel="stylesheet" href="{{(name ~ '-original')|css_uri}}">
	{% endif %}

{# Typical synchronous CSS #}
{% elseif type == 'sync' or type == '' %}
	<link rel="stylesheet" href="{{path}}">
	
{# Invalid stylesheet type provided. Display error #}
{% else %}
	<!-- Invalid stylesheet type specified: type="{{type}}", name="{{name}}" -->

{% endif %}
{% endspaceless %}
```
{% endraw %}


{% raw %}
{% include 'components/stylesheet.twig' with { 'name': 'carousel' } only %}

{# Critical and Asynchronous CSS (generated from a single 'original' CSS file) #}
{% for fileName in criticalCSS %}
	{% include 'components/stylesheet.twig' with { 'type': 'critical+async', 'name': fileName } %}
{% endfor %}

{% if isLoggedIn %}
	{% include 'components/stylesheet.twig' with { 'type': 'async', 'name': 'admin' } %}
{% endif %}

...

{% endraw %}