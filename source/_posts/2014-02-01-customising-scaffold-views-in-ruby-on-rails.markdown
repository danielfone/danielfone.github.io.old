---
layout: post
title: "Customising Scaffold Views in Ruby on Rails"
date: 2014-02-01 22:24
comments: true
categories: 
---

> In Rails 3.0 and above, generators don't just look in the source root for templates,
> they also search for templates in other paths. And one of them is lib/templates

This fact, taken from the [RailsGuide on Generators][guide], makes it very easy for us to change the views that are generated when we run `rails generate scaffold …`. Let's say we want to change the scaffold's form template so that it uses a select box for a belongs_to relationship.

{% img /images/2014-02-01-customising-scaffold-views-in-ruby-on-rails/select-box.png %}

Here are the steps we need to take:

### Find the original template

These live in the lib/rails/generators/erb/scaffold/templates folder of the railties gem. To find the path of the gem, we can run `bundle show railties`. Alternatively we can run `bundle open railties` and navigate to the _form.html.erb file. Once we've found this, we'll want to copy it.

### Copy it into our application

As quoted above, generators will search for templates in lib/templates. For this template, we'll need to copy it into lib/templates/erb/scaffold/_form.erb.html within our application's root.

### Change the template

At this point, we can make whatever changes we like! By way of illustration, here's one way to use a select box for a belongs_to relationship.

{% codeblock lib/templates/erb/scaffold/_form.html.erb.diff %}
@@ -23,7 +23,7 @@
 <% else -%>
   <%- if attribute.reference? -%>
     <%%= f.label :<%= attribute.column_name %> %><br>
-    <%%= f.<%= attribute.field_type %> :<%= attribute.column_name %> %>
+    <%%= f.collection_select :<%= attribute.column_name %>, <%= attribute.name.camelize %>.all, :id, :name, prompt: true  %>
   <%- else -%>
     <%%= f.label :<%= attribute.name %> %><br>
     <%%= f.<%= attribute.field_type %> :<%= attribute.name %> %>
{% endcodeblock %}

I've uploaded an [example app][repo] to github, where you can see the [customised template][template] in full and also [the view][view] it generates.

Now when we run `rails generate scaffold user name:string user_type:references`, our form will use a helpful select box instead of an empty text field.

---

_Edit: This questions was originally prompted by [a comment][comment] on another post, referencing a [StackOverflow question][soq]._


[guide]: http://guides.rubyonrails.org/generators.html#customizing-your-workflow-by-changing-generators-templates
[view]: https://github.com/danielfone/rails4-custom-scaffold-test/blob/master/app/views/smart_users/_form.html.erb
[template]: https://github.com/danielfone/rails4-custom-scaffold-test/blob/master/lib/templates/erb/scaffold/_form.html.erb
[repo]: https://github.com/danielfone/rails4-custom-scaffold-test
[comment]: http://daniel.fone.net.nz/blog/2013/10/19/prototyping-web-applications-in-rails-4/#comment-1225579568
[soq]: http://stackoverflow.com/questions/21486137/rails-scaffold-references-with-select-input-and-entity-label-with-generators/21496682