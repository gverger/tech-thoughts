---
title: Use Ruby Tags with Vim
description:
date: 2017-07-20 04:20:12 UTC
tags: vim, tags, ruby
author: guillaume
---

We have a Rails application which starts to be larger and larger. In order to navigate easily between the different files we have, I like to use **tags**. It allows for instance to jump directly to the definition of a class or a method living in another part of the codebase, whithout having to navigate to the correct file.

In this document, I will explain how we can get these tags to be generated, and how it integrates in my current workflow.


# What is a tagfile

Vim uses tagfiles to retrieve all tags it needs to navigate. Here is an example of what a tagfile looks like:

```
!_TAG_FILE_FORMAT	2	/extended format; --format=1 will not append ;" to lines/
!_TAG_FILE_SORTED	1	/0=unsorted, 1=sorted, 2=foldcase/
Assignment	../lib/cbc_meal_assigner/models/assignment.rb	/^class Assignment$/;"	c
AssignmentList	../lib/cbc_meal_assigner/models/assignments_list.rb	/^class AssignmentList$/;"	c
CbcMealAssigner	../lib/cbc_meal_assigner.rb	/^module CbcMealAssigner$/;"	m
CbcMealAssigner	../lib/cbc_meal_assigner/version.rb	/^module CbcMealAssigner$/;"	m
Client	../lib/cbc_meal_assigner/models/client.rb	/^class Client$/;"	c
Collection	../lib/cbc_meal_assigner/utils/collection.rb	/^module Collection$/;"	m
Conflict	../lib/cbc_meal_assigner/models/conflict.rb	/^class Conflict$/;"	c
Order	../lib/cbc_meal_assigner/models/order.rb	/^class Order$/;"	c
PastAssignment	../lib/cbc_meal_assigner/models/past_assignment.rb	/^class PastAssignment$/;"	c
ProblemCreator	../lib/cbc_meal_assigner/interactors/problem_creator.rb	/^class ProblemCreator$/;"	c
ProblemSolver	../lib/cbc_meal_assigner/interactors/problem_solver.rb	/^class ProblemSolver$/;"	c
Restaurant	../lib/cbc_meal_assigner/models/restaurant.rb	/^class Restaurant$/;"	c
Solution	../lib/cbc_meal_assigner/models/solution.rb	/^class Solution$/;"	c
VERSION	../lib/cbc_meal_assigner/version.rb	/^  VERSION = "0.1.0"$/;"	C	class:CbcMealAssigner
```

There is a tag per line, except the first 2 lines which are metadata. On a tag line, there is:

1.  The tag
2.  the file that defines it
3.  The line defining it. Not the number, the actual line
4.  the type of tag (for instance `c` for class)
5.  The scope within which it is defined

These informations help Vim to know exactly if the word under the cursor is a tag, and where to precisely find it. With this file you can [browse your codebase](http://vim.wikia.com/wiki/Browsing_programs_with_tags) more efficiently!


# Generating tags

I use [ripper-tags](https://github.com/tmm1/ripper-tags) (v0.3.4) as my ruby tags generator.

1.  It takes into account the modules like `MyModule::MyClass`, which is not something [Exuberant Ctags](https://github.com/tmm1/ripper-tags) does.
2.  It is fast enough: for our 49000 lines codebase, it takes 5 seconds to create the tags file

The command I use is

```bash
ripper-tags -f .git/tags -R --tag-relative
```

-   `-f .git/tags` because I want to put it in the `.git` folder. If I put it directly at the root of the project, Spring goes crazy and consumes my CPU for a couple of minutes.
-   `-R` to make it look at files recursively
-   `--tag-relative` so that Vim can find the tags later on, they are stored with relative paths.


# Use the tags in Vim

We need to tell Vim to look at the `tags` file in the `.git` folder. Just put this line in your `~/.vimrc`:

```
set tags=.git/tags,tags;
```

It will tell Vim to check in the `.git/tags` file first, then the `tags` file in the current directory, and then `tags` files up to the root directory.


# Checking the changes in the codebase

The tags can get outdated when any of the `*.rb` files is added, removed, or updated. That is when we should rerun the tags generator. Of course we don't want to rerun it manually each time we make a change, so we will set up an automatic check.


## File change events

We want to detect anytime a file changes in our codebase. One option would be to hook ripper-tags to a buffer save in vim, but it will not detect when we checkout another git branch. We need something outside of our editor to do that.

Enter [fswatch](https://github.com/emcrisostomo/fswatch):

-   it receives notifications when the content of the specified directories change
-   it can respond to certain types of files

To get notified when a ruby file is changed here, we can do:

```bash
fswatch -o -e ".*" -i "\.rb$" ./
```

-   `-o` tries to group all events into one event: we will be notified only once if we do a `git checkout another-branch`
-   `-e ".*"` excludes all files from being watched: we don't want to watch non-ruby files
-   `-i "\.rb$"` includes all ruby files
-   `./` checks the current directory


## Run ripper-tags

Now we have an update event, we want to run ripper-tags to replace our old `.git/tags` file. The workflow is:

1.  Run `ripper-tags` to create a new tags file
2.  Replace the old tags file with the new one

The reason behind not using `ripper-tags` to directly write the `.git/tags` file is that it creates the file at the beginning of its execution, and then write the tags at the end. This would let us with an empty tags file during 5 seconds every time we save a file in Vim.

```bash
run_ripper() {
    echo `date`: creating tags
    ripper-tags -f .git/tags.new -R --tag-relative
    mv .git/tags.new .git/tags
    echo `date`: done
}
```


## Wire everything up

Now we will make sure `run_ripper` is called everytime `fswatch` receives an event:

```bash
export -f run_ripper # We will run the function inside xargs
fswatch -0 -o -e ".*" -i "\.rb\$" ./ | xargs -0 -n1 bash -c 'run_ripper'
```

I usually put every script I want to be able to run from everywhere in my `~/bin/` folder, which is in my `PATH`. In this case, I put everything in `~/bin/refresh_tags` file, I just have to invoke:

```shell
$ refresh_tags
Jeu 20 jul 2017 10:47:20 PDT: creating tags
Jeu 20 jul 2017 10:47:28 PDT: done
Jeu 20 jul 2017 11:07:55 PDT: creating tags
Jeu 20 jul 2017 11:08:02 PDT: done
Jeu 20 jul 2017 17:25:20 PDT: creating tags
Jeu 20 jul 2017 17:25:28 PDT: done
Jeu 20 jul 2017 17:25:28 PDT: creating tags
Jeu 20 jul 2017 17:25:34 PDT: done
Jeu 20 jul 2017 17:25:34 PDT: creating tags
Jeu 20 jul 2017 17:25:40 PDT: done
Jeu 20 jul 2017 17:25:40 PDT: creating tags
Jeu 20 jul 2017 17:25:45 PDT: done
Jeu 20 jul 2017 17:25:45 PDT: creating tags
Jeu 20 jul 2017 17:25:50 PDT: done
Jeu 20 jul 2017 17:25:50 PDT: creating tags
Jeu 20 jul 2017 17:25:55 PDT: done
```

Don't forget to let it run while you are coding!
