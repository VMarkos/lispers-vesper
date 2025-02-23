---
layout: page
title: "Simple Client-Server"
categories:
  - Lisp
  - Development
tags:
  - Lisp
  - Client-Server
  - Development
  - Emacs
---

In order to create a simple client-server network I will first copy--paste an existing project and then try to make sense of it. Then I will also build something on top of it, like a client that spams the server with many messages and a server that echoes those messages back. Or something like that.

*For the final version of the scripts, look at the end of this post!*

<!--more-->

## Getting Started

I will start by utilising [this](https://github.com/ArchI3Chris/SBCL-Basic-Client-Server-Example) seemingly simple approach, which also appears to be well-commented.

I download all files, put them in a nice directory and then created a directory for my own project named `source`. Then I open emacs, which I have carefully avoided all my years as a (non-lisp) developer. It feels like you really have to get accustomed to a vocally strange universe using emacs, with all those strange keyboard shortcuts. Anyway, I hit `Alt + x` and then write `slime` to run slime. But, why do I need that?

Well, I realise I do not, so I look around for a way to open a directory in emacs: `Ctrl + x + d` does the trick. A dialogue at the bottom of the screen prompts me to enter the path of the directory I want to open; so I do. Then, `Ctrl + x Ctrl + f` does the trick to create a new file ('visit', as they seem to say?). In any case, in that file I write the following:

```lisp
#!/usr/bin/sbcl --script
(load ~/.sbclrc) ;; This loads sbclrc

(require :usocket) ;; This loads a CL library that handles sockets and socket operations

(defparameter *sock* nil) ;; Dynamically scoped parameter, initialised to NIL
(defparameter *sock-listen* nil)
(defparameter *my-strem* nil)
```

*Hint:* To copy and paste stuff in emacs:

* Move your cursor to the start of the desired block;
* Hit `Ctrl + Space`;
* Move your cursor to the end of the block;
* Hit `Alt + w` and you are ready to paste!

The above, essentially, makes some necessary imports and definitions, nothing further, as I can read from the corresponding documentation and comments. The first line looks like a shebang line which tells the system how to run this script, much like with such lines, e.g., in bash scripts.

Before I proceed, I hate this 8 spaces tab so I open my `init.el` file, located at `~/.emacs.d/init.el` and add this line:

```lisp
(setq-default indent-tabs-mode nil)
(setq-default tab-width 4)
(setq indent-line-function 'insert-tab)
```

(I used `nano` to edit this thing, which I like more, so far, to be fair).

This, as I understand, deletes the previous setting of the TAB character, sets it to 4 and also implements the auto-indentation of when changing line (quite convenient). [[source](https://stackoverflow.com/questions/69934/set-4-space-indent-in-emacs-in-text-mode)]

## The Actual Server

Then I closed and re-opened emacs and played around with some more code (`Ctrl + x Ctrl + s` saves your code on emacs, just `Ctrl + s` opens up search, just FYI):

```lisp
(defun communicate ()
    ;; This binds the socket to a simple localhost address, on port 65432
    (setf *sock* (usocket:socket-listen "127.0.0.1" 65432))

    ;; listen for incoming connectins by the client
    (setf *sock-listen* (usocket:socket-accept *sock*) :element-type 'character)
)
```

This is essentially what one would do in, say, Python, using `sockets` and `.bind()` / `.accept()` so, nothing strange on the socket setup end. However, what is this `setf` thing? - I have read online that the asterisk is for dynamic scoping, so we are fine on that.

After a brief online search it turned out that `setf` is much like an assignment operator for a typical l-value in other "common" languages. So, `setf *socket-listen* ...` in the above is much like `socket-listen = ` in a language that uses `=` as a typical `lvalue = rvalue` assignment operator. Also, as I read [here](https://stackoverflow.com/questions/869529/difference-between-set-setq-and-setf-in-common-lisp), `setf` seems to have also covered the usecases of `set` and `setq`. Anyway, I will probably encounter them in the near future again.

Now, I am wondering what this `:element-type` thing is, followed by `'character`. Reading online again (and according to its name), `:element-type` determines the type of a variable, so `*sock-listen*` here is of type `character`. 

Then I proceed to write this thing into my `coomunicate()` function:

```lisp
(defun communicate ()
    ;; This binds the socket to a simple localhost address, on port 65432
    (setf *sock* (usocket:socket-listen "127.0.0.1" 65432))

    ;; listen for incoming connectins by the client
    (setf *sock-listen* (usocket:socket-accept *sock*) :element-type 'character)

    ;; open a stream
    (setf *stream* (usocket:socket-stream *sock-listen*))

    ;; print stuff
    (format t "~a~%" (read *stream*))
    (force-output)

    ;; echo the same message back
    (print *stream* *stream*)
    (force-output *stream*)
)
```

The intention here is to capture the stream of the listening socket, using `usocket:socket-stream` and then to print this thing out using `format`. I am blindly following my resource on that and I chose to echo the same message back to the client. I have a gut feeling that my approach on that is not correct, since `read` might consume the contents of `*stream*`, hence it might then be empty. But, why worry, I will figure things out once I am done with the client too!

Next, I defined this strange function (?)

```lisp
(unwind-protect (communicate)
    (format t "Closing socket connectio...~%")
    (usocket:socket-close *sock-listen*)
    (usocket:socket-close *sock*)
)
```

`unwind-protect`, as I read [here](https://stackoverflow.com/questions/12999691/unwind-protect-how-does-it-work) is much like a `finally` clause in typical exception handling, so it is like trying to run the `communicate()` function in a `try` clause - or, at least, that's how I understand it at the moment; time will show.

Next, we print a message and close the listener and then the socket itself, so all seems a bit clear, at the moment.

Actually, no, not everything! What is the proper way to use `format`? I see this is something like the almighty `printf` of C, but, still, I have systematically avoided learning the deepest secrets of `printf` until now, so, I have then to look up for CL's `format`.

Reading online I found many things, among others, that CL's `format` even has its own Wikipedia [page](https://en.wikipedia.org/wiki/Format_(Common_Lisp)), where I read that:

> Format is a function in Common Lisp that can produce formatted text using a format string similar to the print format string. It provides more functionality than print, allowing the user to output numbers in various formats (including, for instance: hex, binary, octal, roman numerals, and English), apply certain format specifiers only under certain conditions, iterate over data structures, output data tabularly, and even recurse, calling format internally to handle data structures that include their own preferred formatting strings. This functionally originates in MIT's Lisp Machine Lisp,[1] where it was based on Multics.

As for the syntax, I found [this](https://gigamonkeys.com/book/a-few-format-recipes) excellent resource, which says quite a few things:

* To begin with, the first parameter, `t` in the above script, is roughly the output channel, with `t` standing for `stdout`.
* Providing `nil` instead of `t` as I understand returns the output as a string instead of printing it, so this is also like `sprintf` of C (amazing?).
* All `format` directives start with a tilde, i.e., `~`.
* `~%` is an argument that emits a newline character, so consuming no arguments from the ones provided next (nice trick there).
* `~$` is usd to print flaots, by default printing two decimal digits after the decimal point. We can modify this to any accuracy by just writing `~3$` for 3 digits, `~6$` for 6, etc.
* You can get the number of any remaining arguments using `~#$` (what the heck, this is getting weirder and weirder...)
* `~A` is a general purpose directive that consumes a format argument and prints it in a human readable form (*aesthetic*, as I see they call it).
* `~a` used in the above examples seems to be the same, so all those directives must be case insensitive.
* `~S` seems to be much like `~A` but it also generates output that can be parsed again using `read`. I don't exactly get what this means at the moment, but time will probably make this relevant, again...

There are many more things discussed here, but I don't feel they are relevant at the moment. Bookmarked, however, for further usage.

## The Client

Okay, the server is, probably, there, so let's start with the client part. I create a new file, `client.cl` and put those lines in it:

```lisp
#!/usr/bin/sbcl --script
(load "~/.sbclrc")

(require :usocket)

;; Create a data array to send to server
(defparameter *data-array* (make-array 16))
(setf (aref *data-array* 5) 118)
(setf (aref *data-array* 7) "some")
(setf (aref *data-array* 9) 'some)

(defparameter *sock* nil)
(defparameter *stream* nil)
```

Things here are more or less the same, it is only this `make-array` thing that boggles me? Maybe `16` is the number of bytes allocated? Or maybe I am too C-minded for CL at the moment?

Again, looking around saves the day. The first argument of `make-array` can be a list of integers corresponding to the dimensions of the array. So, `16` is the number of elements that can fit into this array and not the bytes allocated to it (too low level, maybe, even as a thought).

Then, what is this `aref` thing? My guess, based on the above, would be that this gets an array as a first argument and a list of indices (in the case of multidimensional arrays, maybe) that determine the specific element of the array. So, this is then returned as what would be in other languages an l-value, to which we then assign. So, further unravelling my thoughts, this might result into an array looking like this:

```
[nil, nil, nil, nil, nil, 118, nil, "some", nil, 'some, nil, nil, nil, nil, nil, nil]
```

Now, looking online I find exactly this thing [[source](https://jtra.cz/stuff/lisp/sclr/aref.html)], so we are fine on that. However, what is the difference between `"some"` and `'some`?

Well, I first asked Google's Gemini to help me with this think and give me some pointers to start with. The core part of its response was:

> In essence:
>   * "foo" is a sequence of characters.
>   * 'foo is a symbolic object.
> 
> Here's a simple way to think about it:
>   * If you want to work with the letters "f", "o", "o" as a piece of text, you'd use "foo".
>   * If you want to use "foo" as a name or a distinct identifier within your Lisp program, you'd use 'foo.

So, it seems that `'some` in the above is a different kind of object. [This](https://stackoverflow.com/questions/134887/when-to-use-or-quote-in-lisp) Stack Overflow answer clarified it for me. Using the `quote` (`'`) directive then the following argument(s) are returned without being evaluated, which is in some times important, since CL by default evaluates everything it parses, as I understand. So, in this sense, `'some` actually means "fetch me this new symbol, `some` **without evaluating it**", which makes sense in this case, since it does have no value at the moment.

The rest of the code is mostly as with the server:

```lisp
(defun communicate()
    ;; socket binding
    (setf *sock* (usocket:socket-connect *sock*))

    ;; open stream
    (setf *stream* (usocket:socket-stream *sock*))

    ;; stream to server
    (print *data-array* *stream*)
    (force-output *stream*)

    ;; catch and print server reply
    (format t "~a~%" (read *stream*))
    (force-output)
)

(unwind-protect (communicate)
    (format t "~a~%" "Closing Connection...")
    (usocket:socket-close *sock*)
)
```

The only thing I don't quite get is this `force-output` thing. So, let's look again online for that. At [this](https://stackoverflow.com/questions/2078490/lisp-format-and-force-output) SO answer I see that when it comes to IO in CL, we have three functions:

* `finish-output`, which makes sure that all output is done and then returns.
* `force-output`, which returns immediately, without waiting for all output.
* `clear-output`, which deletes any pending output.

I am not sure I get it, but the aim in the above is to directly flush the content to the other end. Maybe if I tried to actually implement a TCP/IP connection with proper (sliding) windows and all that stuff I should not just `force-output` but first chunk at then force, or something like that? Who knows?

## Running This Thing!

Okay, I am done with studying the code, so let's get things running! I suspect I first have to run the server and then the client, so my gut feeling is to open up a terminal and start the server process. I found a lot of things on that, maybe the most inclusive being [this](https://stackoverflow.com/questions/2992925/how-can-i-simply-run-lisp-files) SO answer, which provides a few different ways to run CL scripts. Since my scripts both have a shebang line on top, my gut feeling was correct (maybe) and the seemingly most natural way would be to directly run them from the terminal. So, I open up a terminal at my `source` directory and hit:

```sh
$ ./server.cl
```

This returned (silly me):

```sh
bash: ./server.cl: Permission denied
```

Well, evidently, I first have to `chmod` so:

```sh
$ chmod +x server.cl && ./server.cl
```

This did not succeed, printing on the terminal:

```sh
Unhandled UNBOUND-VARIABLE in thread #<SB-THREAD:THREAD "main thread" RUNNING
                                        {1001348003}>:
  The variable ~/.SBCLRC is unbound.
```

Followed by a vast stacktrace (*backtrace*, as it says). 

So, I copy this thing and look it up online, but I do not get something directly relatable, so I look up at the backtrace, the first few lines of which look like this:

```sh
Backtrace for: #<SB-THREAD:THREAD "main thread" RUNNING {1001348003}>
0: (SB-INT:SIMPLE-EVAL-IN-LEXENV ~/.SBCLRC #<NULL-LEXENV>)
1: (SB-INT:SIMPLE-EVAL-IN-LEXENV (LOAD ~/.SBCLRC) #<NULL-LEXENV>)
2: (EVAL-TLF (LOAD ~/.SBCLRC) 0 NIL)
3: ((LABELS SB-FASL::EVAL-FORM :IN SB-INT:LOAD-AS-SOURCE) (LOAD ~/.SBCLRC) 0)
```

The `2:` part above is a bit enlightening: the issue is at the `load ~/.sbclrc` part, so I open up the `server.cl` script again. As a side note, to open an additional window stacked vertically on top of the others, I used the `Ctrl + x 2` key binding.

Looking through the script, I se I have actually written `(load ~/.sbclrc)`, as I expect, so I look up at my home directory whether `.sbclrc` is there; and it actually is. So, what am I missing here?

*[Take some time to meditate...]*

After a few (thankfully) moments, I realise I have missed the double-quotes around, so changing it to:

```lisp
(load "~/.sbclrc")
```

things should work as expected (fingers crossed!). Rerunning the script, I get another error:

```sh
Unhandled SB-INT:EXTENSION-FAILURE in thread #<SB-THREAD:THREAD "main thread" RUNNING
                                                {10048C81E3}>:
  Don't know how to REQUIRE USOCKET.
See also:
  The SBCL Manual, Variable *MODULE-PROVIDER-FUNCTIONS*
  The SBCL Manual, Function REQUIRE
```

Evidently, I have to install `usocket` somehow. After a quick tour around the web, I found out that the easiest way is *Quicklisp* (QL), which is the most popular package manager, which, however, servers things over HTTP (no TLS / SSL), so it is vulnerable to Man In The Middle attacks (a risk I can take, at the moment, given my low importance as a target).

So, I open up a terminal window and hit the following commands [[source](https://www.quicklisp.org/beta/#installation)]:

```sh
$ curl -O https://beta.quicklisp.org/quicklisp.lisp
$ curl -O https://beta.quicklisp.org/quicklisp.lisp.asc
$ gpg --verify quicklisp.lisp.asc quicklisp.lisp
$ sbcl --load quicklisp.lisp
$ (quicklisp-quickstart:install)
```

Having QL, I then look up for `usocket`:

```sh
$ (ql:system-apropos "usocket")
```

Since it is there, I try to install it by loading it which, as I understand, implies installation in case it is not there:

```sh
$ (ql:quickload "usocket")
```

Then I add this to the SBCL init file (I think this is the part that makes it "visible" to my CL installation):

```sh
$ (ql:add-to-init-file)
```

Then I quit the session by evaluating `quit`:

```sh
$ (quit)
```

Now, I execute my server again, which runs with an error and a few warnings. To begin with, I get the following two:

```sh
; in: DEFUN COMMUNICATE
;     (SETF *SOCK-LISTEN* (USOCKET:SOCKET-ACCEPT *SOCK*)
;           :ELEMENT-TYPE 'CHARACTER)
; --> SETF 
; ==>
;   (SETQ :ELEMENT-TYPE 'CHARACTER)
; 
; caught ERROR:
;   :ELEMENT-TYPE is a constant and thus can't be set.

; in: DEFUN COMMUNICATE
;     (PRINT *STREAM* *STREAM*)
; 
; caught WARNING:
;   undefined variable: COMMON-LISP-USER::*STREAM*

;     (READ *STREAM*)
```

The second one is something that caught my eye directly: there was a typo as I transcribed the source code of my server:

```diff
;; server.cl

- (defparameter *my-strem* nil)
+ (defparameter *stream* nil)
```

Before I get to understand the other error and see what happens, I terminate and rerun my server, since this might resolve everything (fingers crossed again). Not oddly enough, I get the same error as before (no wranings though):

```sh
; in: DEFUN COMMUNICATE
;     (SETF *SOCK-LISTEN* (USOCKET:SOCKET-ACCEPT *SOCK*)
;           :ELEMENT-TYPE 'CHARACTER)
; --> SETF 
; ==>
;   (SETQ :ELEMENT-TYPE 'CHARACTER)
; 
; caught ERROR:
;   :ELEMENT-TYPE is a constant and thus can't be set.
; 
; compilation unit finished
;   caught 1 ERROR condition
```

Okay, this was another syntax error of mine, since I misplaced parentheses, as shown below:

```diff
;; server.cl

- (setf *sock-listen* (usocket:socket-accept *sock*) :element-type 'character)
+ (setf *sock-listen* (usocket:socket-accept *sock* :element-type 'character))
```

Rerunning the project triggers no error, so I hope everything is as expected. Actually, to make sure, I append a few `format`s to check that things are as they should:

```diff
;; server.cl

;; This binds the socket to a simple localhost address, on port 65432
+ (format t "Binding at: ~A:~A...~%" "127.0.0.1" 65432)
(setf *sock* (usocket:socket-listen "127.0.0.1" 65432))

;; listen for incoming connectins by the client
+ (format t "Listening at: ~A:~A...~%" "127.0.0.1" 65432)
(setf *sock-listen* (usocket:socket-accept *sock* :element-type 'character))
+ (format t "Accepted connection!~%")
```

Yay! Things seem to work, so let us open our client, running this at a separate terminal:

```sh
$ chmod +x client.cl && ./client.cl
```

Hell yeah, a huge... backtrace is there for us! But, this time things seem a bit clear:

```sh
The value of SB-BSD-SOCKETS::ADDRESS is (NIL NIL), which is not of type
```

then followed by a wall of error messages. So, let us look back at the socket configuration in `client.cl`:

```diff
;; socket binding
- (setf *sock* (usocket:socket-connect *sock*))
+ (setf *sock* (usocket:socket-connect "127.0.0.1" 65432))
```

So, okay, now things should run...

```sh
illegal sharp macro character: #\<
```

Looking around, this seems to be a generic way SBLC has to inform us that there is a character it cannot parse [[source](https://stackoverflow.com/questions/60977210/simple-reader-error-illegal-sharp-macro-character-subcharacter-not-define)]. Seeing the server terminal, we can observe that things have run as expected, so one potential cause for this error might be the way we are printing back the message from the server:

```lisp
;; server.cl

(print *stream* *stream*)
```

To debug this, let's make a simple change:

```diff
;; server.cl

- (print *stream* *stream*)
+ (print "Hi!" *stream*)
```

Rerunning our server and then our client, things work as expected! So, how can we reply with the same message to the client from our server? Let us refactor our server logic a bit:

```diff
(defparameter *sock* nil) ;; Dynamically scoped parameter, initialised to NIL
(defparameter *sock-listen* nil)
(defparameter *stream* nil)
+ (defparameter *message* nil)

(defun communicate ()
    ;; This binds the socket to a simple localhost address, on port 65432
    (format t "Binding at: ~A:~A...~%" "127.0.0.1" 65432)
    (setf *sock* (usocket:socket-listen "127.0.0.1" 65432))

    ;; listen for incoming connectins by the client
    (format t "Listening at: ~A:~A...~%" "127.0.0.1" 65432)
    (setf *sock-listen* (usocket:socket-accept *sock* :element-type 'character))
    (format t "Accepted connection!~%")

    ;; open a stream
    (setf *stream* (usocket:socket-stream *sock-listen*))

    ;; store client message
+   (setf *message* (read *stream*))
        
    ;; print stuff
-   (format t "~a~%" (read *stream*))
+   (format t "~a~%" *message*)
    (force-output)

    ;; echo the same message back
    (print "Hi!" *stream*)
    (force-output *stream*)
)
```

This runs as expected, so, let us try to reply back to the client by using our freshly baked `*message*` variable:

```diff
;; server.cl

;; echo the same message back
- (print "Hi!" *stream*)
+ (print *message* *stream*)
(force-output *stream*)
```

Run this one more time and we are there!

Server output:

```sh
Binding at: 127.0.0.1:65432...
Listening at: 127.0.0.1:65432...
Accepted connection!
#(0 0 0 0 0 118 0 some 0 SOME 0 0 0 0 0 0)
Closing socket connection...
```

Client output:

```sh
#(0 0 0 0 0 118 0 some 0 SOME 0 0 0 0 0 0)
Closing Connection...
```

Okay, that's perfect!

## Final Version

Below are the final versions of the two CL files we have generated:

Server:

```lisp
#!/usr/bin/sbcl --script
(load "~/.sbclrc") ;; This loads sbclrc

(require :usocket) ;; This loads a CL library that handles sockets and socket operations

(defparameter *sock* nil) ;; Dynamically scoped parameter, initialised to NIL
(defparameter *sock-listen* nil)
(defparameter *stream* nil)
(defparameter *message* nil)

(defun communicate ()
    ;; This binds the socket to a simple localhost address, on port 65432
    (format t "Binding at: ~A:~A...~%" "127.0.0.1" 65432)
    (setf *sock* (usocket:socket-listen "127.0.0.1" 65432))

    ;; listen for incoming connectins by the client
    (format t "Listening at: ~A:~A...~%" "127.0.0.1" 65432)
    (setf *sock-listen* (usocket:socket-accept *sock* :element-type 'character))
    (format t "Accepted connection!~%")

    ;; open a stream
    (setf *stream* (usocket:socket-stream *sock-listen*))

    ;; store client message
    (setf *message* (read *stream*))
        
    ;; print stuff
    (format t "~a~%" *message*)
    (force-output)

    ;; echo the same message back
    (print *message* *stream*)
    (force-output *stream*)
)

;; this looks a bit like a "main" function, or, the actually callable part
(unwind-protect (communicate)
    (format t "Closing socket connection...~%")
    (usocket:socket-close *sock-listen*)
    (usocket:socket-close *sock*)
)
```

Client:

```lisp
#!/usr/bin/sbcl --script
(load "~/.sbclrc")

(require :usocket)

;; Create a data array to send to server
(defparameter *data-array* (make-array 16))
(setf (aref *data-array* 5) 118)
(setf (aref *data-array* 7) "some")
(setf (aref *data-array* 9) 'some)
;; (defparameter *data* "Hi Bob, It's Alice!")

(defparameter *sock* nil)
(defparameter *stream* nil)

(defun communicate ()
    ;; socket binding
    (setf *sock* (usocket:socket-connect "127.0.0.1" 65432))

    ;; open stream
    (setf *stream* (usocket:socket-stream *sock*))

    ;; stream to server
    (print *data-array* *stream*)
    (force-output *stream*)

    ;; catch and print server reply
    (format t "~a~%" (read *stream*))
    (force-output)
)

(unwind-protect (communicate)
    (format t "~a~%" "Closing Connection...")
    (usocket:socket-close *sock*)
)
```

## What's Next?

Okay, now that we have a simple client and server up and running, why not explore some typical networking concepts? To begin with, next time we will try to get a dummy encryption working and then, why not, proceed to handle multiple clients.

Till then... I don't know, I am not good at this thing, just have fun!
