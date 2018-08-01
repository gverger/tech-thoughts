---

title: Intelligent Snippets with Vim
date: 2018-08-01 13:40 UTC
tags: vim

---

I am not a big fan of commenting my code. I really am not. So when we took the decision to comment
our app source code with YARD, I was not very happy. In my opinion, a comment is about to be
outdated and misleading soon.

Code doesn't lie. If you understand the code, you don't need these comments.
But when you provide other people your library/app/piece of code, it is considered nice to provide
some documentation. That will help them use your code without having to browse the source code.

Fair enough, so let's document the public part of the code then.

We are building a ruby library that will be used by others at some point. We want to provide a web
interface to the library so that any user can check what methods they can use, and how to use them.

We decided to use [YARD](https://yardoc.org/) mainly because of how easy it is to generate web
pages directly from the comments.

With YARD, you will comment classes, modules and methods. In this post I will focus on commenting
methods, because there are some stuff you can extract automatically from the method's signature to
put it in your comment.

And because I am lazy, I prefer to build a solution that does my job semi-automatically, even if it
takes me a long time to build it. Eventually this time will be paid of, and that will be a win: fun
during the creation of the tool, and ease of use.

# YARD, the format

Before we dive into technical details, let me present you the format I will be focusing on today.
Here is the beast:

```ruby
# Write a blog post and author it with your name
# @param title [String] The post's title
# @param author [String] Who is writing it
# @return [String] A shiny blog post
def write_a_blog_post(title = 'Intelligent snippets with Vim', author: 'Guillaume')
  # Sorry the implementation will remain a secret of mine
end
```

Here you can see the nice comments on top of the method `write_a_blog_post`. We can specify a
description, then the parameters the method takes, and the return value.

Here is how is the page generated:

![Created documentation page](2018-08-01/yard-page-example.png)

# My development setup

As you could deduce from the title, I will be using Vim to write these comments.
I have been pairing Vim with [Ultisnips](https://github.com/SirVer/ultisnips) for a while, creating
simple custom snippets when I was lacking some functionality.

What I want to have is an easy way to create a part of my method's documentation in a couple of
keystrokes. Did I tell you I was lazy?

Let's start with the end: how will this work?

![Using the snippet](2018-08-01/ultisnips-use.gif)

Type `ym` before a method definition and voilà! Now jump between the tabstops and fill out the
info.

# But... How?

The only tricky part here is to find the method's parameters. That what we want to be automatically
filled up. This allows us not to misspell variable names.
We will need Ultisnips, and we will define our own snippet. We will use Python to navigate the file
and find the parameters.

Let's start with a custom snippet:
From Vim `:UltisnipsEdit` brings you to a custom snippet file (invoke this command from a ruby file)
Bear with me, this will be pretty ugly.

```ultisnips
snippet "(\s*)ym\s*(.*)" "YARD method" br
`!p snip.rv = f"{match.group(1)};{match.group(2)}"`
endsnippet
```

This snippet alone doesn't do much. It will replace in place

```
  ym method_name
```

with

```
  ;method_name
```

On the left side of `;`, we will have the number of spaces before `ym`. We will use this spaces to
indent the comments correctly.
On the right side, we have the method name. We will use it to parse the buffer and find the method.

The rest will be done by a python script. The idea is to find the first method corresponding to the
provided name (or the first method, if no name is given), parse the parameters, and format our YARD
comment.

We will use a trick from ultisnips: `post_jump`. This directive will run a python script after we
first jump to our snippet. We will then replace our line with the comments with placeholders. We
are basically generating a dynamic snippet body.

We want our dynamic snippet to look like:

```ultisnips
snippet dumb
# ${0:description}
# @param title [${1:Type}]$2
# @param author [${3:Type}]$4
# @return [${5:TYPE}] ${6:value returned}
endsnippet
```
With such a snippet, we could navigate between tabstops ($1, $2, $3...) to fill out our
documentation.


Let's navigate the vim buffer in search for our definition line.

```python
def params(snip, method_name):
	for idx in range(snip.buffer.cursor.line, len(snip.buffer)):
    line = snip.buffer[idx]
		if "def " + method_name in line:
			return params_for_line(line)
	return None
```
This is some Python. I don't do Python much. The idea is to start from the line where we are in Vim
and check below if a method corresponds to our needs. If no method_name is given, we'll take the
first method, whatever its name.

Once found we call `params_for_line`:

```python
def params_for_line(line):
	import re
	arguments = re.search("def .*\((.*)\)", line)
	if arguments:
		lst = arguments.group(1).split(',')
		lst = list(map(lambda s: re.split('[:=]', s)[0].strip(), lst))
		return lst
	return []
```

This one takes a line, and takes all parameters from it. It will return the list of parameters, and
discard default values.

Our journey is almost finished. Now let's `post_jump`!

```
post_jump "yard_method_with_params(snip)"
```

This line has to go just above the first snippet. It will invoke a yet-to-be-written python
function which has the lovely name of `yard_method_with_params`:

```python
def yard_method_with_params(snip):
	spaces, method = snip.buffer[snip.line].split(';')
	snip.buffer[snip.line] = ''
	param_list = params(snip, method)

	if param_list is None:
		snip.expand_anon(spaces + 'ym ' + method + ' # NO SUCH METHOD')
		return

	body = f'{spaces}# ${{0:{method} description}}'
	body += ''.join([f"\n{spaces}# @param {p.strip()} [${{{str(2*i+1)}:Type}}] ${2*i+2}" for i, p in enumerate(param_list)])
	body += f'\n{spaces}# @return [${{{str(len(param_list) * 2 + 1)}:TYPE}}] ${{{str(len(param_list) * 2 + 2)}:value returned}}'

	snip.expand_anon(body)
```

- The first 3 lines get the indentation level, and the parameters. It also wipe out the current
line.
- Next 3 lines are a guard when we cannot find the method. We will just render the user's input
with a comment at the end for some feedback
- Next 3 lines build the YARD documentation, with tabstops.
- Last line tells Ultisnips to replace the current snippet with the `body` string we just created.

Here we are! There are some cases where this snippet won't work:
- When the definition spans over several lines
- When another method between the cursor and the method we want starts with the same name
- Probably some other cases?

But this covers all my needs for now.

# The end?

This is the end for these snippets, but being able to create such beauty opens my snippet world to
a myriad of possibilities. We can probably reuse the `params` method in a lot of different
scenarios.

So, this is more the beginning of an even smarter Vim. I love this editor.

And I am now happy inserting YARD comments in my code, it feels very nice to see this magic
happening!


# Full code

```python
global !p
def params_for_line(line):
	import re
	arguments = re.search("def .*\((.*)\)", line)
	if arguments:
		lst = arguments.group(1).split(',')
		lst = list(map(lambda s: re.split('[:=]', s)[0].strip(), lst))
		return lst
	return []

def params(snip, method_name):
	for idx in range(snip.buffer.cursor.line, len(snip.buffer)):
		line = snip.buffer[idx]
		if "def " + method_name in line:
			return params_for_line(line)
	return None

def yard_method_with_params(snip):
	spaces, method = snip.buffer[snip.line].split(';')
	snip.buffer[snip.line] = ''
	param_list = params(snip, method)

	if param_list is None:
		snip.expand_anon(spaces + 'ym ' + method + ' # NO SUCH METHOD')
		return

	body = f'{spaces}# ${{0:{method} description}}'
	body += ''.join([f"\n{spaces}# @param {p.strip()} [${{{str(2*i+1)}:Type}}] ${2*i+2}" for i, p in enumerate(param_list)])
	body += f'\n{spaces}# @return [${{{str(len(param_list) * 2 + 1)}:TYPE}}] ${{{str(len(param_list) * 2 + 2)}:value returned}}'

	snip.expand_anon(body)
endglobal

post_jump "yard_method_with_params(snip)"
snippet "(\s*)ym\s*(.*)" "YARD method" br
`!p snip.rv = f"{match.group(1)};{match.group(2)}"`
endsnippet
```


