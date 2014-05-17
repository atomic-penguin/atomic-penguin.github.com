---
layout: post
title: "Stupid Jenkins Tricks, Part 1: Rubocop"
date: 2014-04-28 21:36:13 -0700
comments: true
categories: CI Chef cookbooks Ruby Rubocop 
---

I have spent the last three months working on a Masters thesis, and just finished up last week after Chefconf.  Trust me, you probably don't want to read that long-winded monstrosity which is short on technical details.  The high-level view of that project was building a continuous integration (CI) pipeline for internal Cookbook development at Marshall.

I learned quite a few tricks during that project, and figured out a few ways to expand on those ideas from [Chefconf 2014](http://chefconf.opscode.com/chefconf).  The hack day event(s), hallway track, and a few of the talks were all insightful towards this end.  In this first part, I am going to tackle some low hanging fruit and demonstrate how to track Rubocop warnings on Jenkins, much like one would track Foodcritic warnings.

<!-- more -->

For as long as I can remember, Andrew Crump has had documentation on tracking [Foodcritic warnings](https://acrmp.github.io/foodcritic/#ci) on Jenkins.  If you haven't already, you will want to check that out for doing internal cookbook development with Jenkins.  I am simply going to show you how one might achieve the same kind of Cookbook/Ruby metric with [Rubocop](https://github.com/bbatsov/rubocop).

I started out using the [rubocop-checkstyle_formatter](https://github.com/eitoball/rubocop-checkstyle_formatter) during the course of my project.  The `checkstyle_formatter` outputs a `checkstyle.xml` file which then gets parsed by the Violations plugin.  Either I was using the `checkstyle_formatter` component wrong, or parsing a checkstyle file works inconsistently compared to scanning logs for warnings.  Perhaps, the Violations plugin expects the checkstyle.xml file to always be present, when I expect that the software project should be wiped and checked out before every touchstone build.

The Warnings plugin ships with a number of pre-defined console log Warning scans, but does not have one for Rubocop or Foodcritic.  In hindsight the Violations plugin did not work as well as the Warnings plugin did with our custom Foodcritic scan.  It was fairly easy to construct a warnings scan based on Andrew's demonstration, however.  As a bonus with a Warnings scan, one can force the plugin to count warnings in the logs even when a build job fails.  This would become important when one wants to track historical trending of warnings.

#### Configuring the Warnings plugin for Rubocop's default output

1. First off, you need Jenkins and the [Warnings plugin](https://wiki.jenkins-ci.org/display/JENKINS/Warnings+Plugin).  (Install plugin, check the box to restart Jenkins, get a cup of coffee).

2. Find your `Jenkins` -> `Manage Jenkins` -> `Configure System` page from the breadcrumb menu.  On a host named Jenkins running on port 8080, that would be at http://jenkins:8080/configure.

3. Find the section labeled `Compiler Warnings`.  Click the `Add` button in this section.

4. Configure the `Compiler Warnings` plugin like so.

    * Name: `Rubocop`
    * Link name: `https://github.com/bbatsov/rubocop`
    * Trend report name: `Ruby Lint Warnings`
    * Regular Expression: `^([^:]+):(\d+):\d+: ([^:]): ([^:]+)$`
    * Example Log Message: `attributes/default.rb:21:78: C: Use %r only for regular expressions matching more than 1 '/' character.`
    * Mapping script:

```java
    import hudson.plugins.warnings.parser.Warning

    String fileName = matcher.group(1)
    String lineNumber = matcher.group(2)
    String category = matcher.group(3)
    String message = matcher.group(4)

    return new Warning(fileName, Integer.parseInt(lineNumber), "Ruby Lint Warnings", category, message);
```

#### Adding the Rubocop scan to a build job.

```ruby
# Rakefile

# -f correctness only fails the build on syntax violations.
# The warnings scan will still count and graph other violations.
desc 'Foodcritic correctness linting task'
task :foodcritic do
  sh 'foodcritic -f correctness .'
end

# --fail-level E will only fail Ruby errors.
# The warnings scan will still count and graph [W]arning and [C]op violations.
desc 'Rubocop linting task'
task :rubocop do
  sh 'rubocop --fail-level E'
end

# touchstone task
task jenkins: %w(rubocop foodcritic)
```

1. If you have not already done so, add a `rubocop` task for your project.  Perhaps one might run `foodcritic` and `rubocop` tasks as part of a static analysis touchstone build in their pipeline.  Running separate build staged build jobs shortens development feedback loops before kicking off a longer deployment and integration test.  In the example Rakefile above, I combined both `rubocop` and `foodcritic` into a single touchstone task called `jenkins`.

2. Click `Add build step` -> `Invoke Rake`.  Set `Rake Version` to an appropriate selection for your system.  Set Tasks to `jenkins` or `rubocop` dependent on the tasks you added in your Rakefile.  If necessary, you may need to precede this build step with a Shell step to `bundle install` your Cookbook project, and check the `bundle exec` option for the Rake invocation.

3. Click `Add post-build action` -> `Scan for compiler warnings` on the build job configuration page.  Select `Rubocop` as the `Parser`.  You can add multiple Warning Parsers for this build step (i.e. both Foodcritic and Rubocop).

4. Click the `Advanced` button and check the option labeled `Run always` so that warnings are recorded even on failed builds.

5. Click `Apply` on the Build Job configuration page and this should track Foodcritic and Rubocop warnings, even for failing build jobs.
