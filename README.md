# dull

A simple yet powerful build tool designed to be unexciting.

## Status

Pre-alpha, doesn't work yet.

## Samples

### Build bonfire docker images

```
$ dull build-docker-images 

1. [.] Build docker images        [====--] (1 / 2)
  1. [✓] Classic
    1. [✓] amd64
    2. [✓] x86
  2. [.] Breadpub                 [===---] (1 / 2)
    1. [✓] amd64
    2. [.] x86

Overall: [==============---] (5 / 6)
```

Config to generate this:

```
(def known-flavours
  [{:name  "Classic"  :value :classic}
   {:name  "Breadpub" :value :breadpub}])

(deftask build-docker-images
  [flavours :type [:keyword] :default (map :value flavours)
   arches   :type [:keyword] :default (supported-arches)]
  (group "Build docker images"
    (map flavours
      (fn [flavour]
        (let [flavour (get known-flavours flavour)]
          (group (:name flavour)
            (map arches
              (fn [arch]
                (step (unkeyword arch)
                  (collapse ; causes nested steps to be hidden and replaced with just a progress bar.
                    (build-docker-image :flavour flavour :arch arch))))))))))

(deftask build-docker-image
  [arch    :type    :keyword
           :default :amd64
           :options [:amd64 :x86]
           :desc    "Architecture to build images for"
   flavour :type    :keyword
           :default :classic
           :options flavours
           :desc    "Flavour to build images for"]
  ...) ;; call docker with the appropriate args
```



### Bonfire first use


```
$ dull hello

Hello, welcome to bonfire! Since you're new here, let's get you set up!

Please answer the following questions, we promise it won't take long!

Which base flavour would you like to use?

  (*) Classic: A fully featured bonfire distribution with all the toys.
  ( ) Breadpub: A bonfire distribution for community sharing of resources.

How would you like to run the postgresql database server?

  (*) Automatically through a container
  ( ) Manual configuration

How would you like to run the application server in development?

  (*) Automatically through a container
  ( ) On my local machine

How do you want to build and run containers?

  Note that whatever you choose should already be installed and configured.

  (*) Rootless podman and buildah. 
  ( ) Docker. It should already be set up.

Do you want to customise which extensions are enabled?

  [✓] foo, it foos
  [✓] bar, likes drinking
  [✓] baz, if your name is baz
  [ ] quux, lorem ipsum

Configuring with settings:

Flavour:  Classic.
Postgres:  Automatically through a container.
Appserver:  Automatically through a container.
Container backend:  Rootless podman and buildah.
Extensions: foo, bar, baz

Is this correct? [Y/n] Y

1. [.] First time setup                     (0 / 1)
  1. [.] Download container images [------] (0 / 1)
2. [ ] Build bonfire
  1. [ ] Fetch elixir dependencies
  2. [ ] Compile elixir code
  3. [ ] Self test

Build status: [==============---] (5 / 6) 
```

Which could be generated potentially by something like this:

```

(def known-flavours
  [{:name  "Classic"
    :value :classic
    :dir   "flavours/classic"
    :desc  "A fully featured bonfire distribution with all the toys."}
   {:name  "Breadpub"
    :value :breadpub
    :dir   "flavours/valueflows"
    :desc  "A bonfire distribution for community sharing of resources."}])

(defconfig flavour
  :desc    "Which base flavour would you like to use?"
  :type    :keyword
  :default :classic
  :options flavours)

(defconfig postgres
  :desc    "How would you like to run the postgresql database server?"
  :type    :keyword
  :default :managed
  :options
  [{:name "Automatically through a container"  :value :managed}
   {:name "Manual Configuration"  :value :manual}])

(defconfig appserver
  :desc    "How would you like to run the application server in development?"
  :type    :keyword
  :default :managed
  :options
  [{:name "Automatically through a container"  :value :managed}
   {:name "On my local machine which is configured correctly"  :value :local}])

(defconfig container-backend
  :desc    "How would you like to run the application server in development?"
  :note    "Note that whatever you choose, it should be set up separately."
  :type    :keyword
  :default :docker
  :options
  [{:name "Docker"  :value :docker}
   {:name "Rootless podman and buildah"  :value :rootless}])

(defconfig extensions
  :desc    "How would you like to run the application server in development?"
  :type    (list-of keyword)
  :default :docker
  :options (selected-flavour-extensions))

(defn selected-flavour-extensions []
  ...) ; todo
  

(def greeting
"""

Hello, welcome to bonfire! Since you're new here, let's get you set up!

Please answer the following questions, we promise it won't take long!
""")

(deftask hello []
  (println greeting)
  (let [basic [flavour postgres appserver]
        _     (ask basic)
        rest  (if (any (curry = :managed) [@postgres @appserver])
                  [container-backend extensions]
                  [extensions])]
    (ask rest)
    (save-defaults (concat basic rest))))

```

### Simple rust project

```
(def rust (cargo "."))

(deftask build []
  (step "Build rust"
    (cargo rust :build)))

(deftask test [] ; shows the test output as it happens
  (follow (cargo rust :test) :stdout :stderr))

(deftask dev []
  (let [sources (cargo rust :sources)]
    (watch sources
      (sequence test beep)))
```

## Features that seem useful

### Config variables

Sometimes we do actually want global configuration. A good example of
this is bonfire where we are trying to solve a lot of problems for a
lot of people and so you need to fill out a questionnaire to get
started.

* Interactive configuration wizard saves default settings to a file.
* Settings can be overridden with command line flags or environment vars for a run.

A configuration option can be specified with `defconfig`:

```
(defconfig container-backend
  :desc     "How would you like to run the application server in development?"
  :note     "Note that whatever you choose, it should be set up separately."
  :type     :keyword
  :default  :docker
  :options
  [{:name "Docker" :value :docker}
   {:name "Rootless podman and buildah" :value :rootless} ])
```

The user can be prompted to provide values for them interactively:

```
(ask [container-backend])
(println (repr @container-backend)) ; e.g. :docker
```

Values can be saved to a defaults file and loaded again

```
(load-defaults) ; you probably want this at the start of your file.
... ; do something
(save-defaults [container-backend])
```

### Tasks

A task is a function with named arguments.

Tasks may be invoked from the command line like make tasks. arguments may be provided with the `--arg=value` format.

```
(deftask build-docker-image
  """
  Builds a single docker image
  
  Example usage:
  
    dull build-docker-image
    dull build-docker-image --arch=x86 # for x86 builds on amd64
    dull build-docker-image --flavour=breadpub # for a specified flavour
    dull build-docker-image --arch=x86 --flavour=breadpub # for both

  """
  [arch    :type    :keyword
           :default (current-arch)
           :options [:amd64 :x86]
           :desc    "Architecture to build image for."
   flavour :type    :keyword
           :default @flavour
           :options (map :value flavours)
           :desc    "Flavour to build image for."]
  
  (shell-step ["docker" "do-something"])) ;; substitute a real command
   
```

This could be invoked from the cli as:

```
$ dull build-docker-image --arch=x86 --flavour=breadpub
```

And the help:

```
$ dull help build-docker-image # or dull build-docker-image --help

Task build-docker-image:

  Builds a single docker image.

  Example usage:
  
    dull build-docker-image
    dull build-docker-image --arch=x86 # for x86 builds on amd64
    dull build-docker-image --flavour=breadpub # for a specified flavour
    dull build-docker-image --arch=x86 --flavour=breadpub # for both

  Arguments

    --arch=KEYWORD # Architecture to build image for.
        default: amd64
        valid:   amd64 x86

    --flavour=KEYWORD: Flavour to build image for.
        default: classic
        valid:   classic breadpub

```


## Other

```
$ dull repl

[✓] Load project file (./project.dull)

Welcome to the dull repl. You may evaluate dull expressions by
typing them followed by the RETURN or ENTER key.

Press Ctrl-D to quit. If you get stuck, call `(help)`.

1 > (help)

Syntax

symbol:  `symbol may-have-dashes`
keyword: `:keyword :may-have-dashes`
integer: `123`
string:  `"string"`

Repl primitives

  (h INT)            Gets the result of evaluating the INTth input expression
  (cd STRING)        Changes the current working directory
  (shell STRING ...) Runs a command in $SHELL

More Help

  (help :primitives) ; gets help about the available primitives
  (help :tasks)      ; gets help about the available build tasks
  (help TASK-NAME)   ; gets help about a particular task

nil
```




## Abstracting out parameters

```
(module bonfire
  (def known-flavours [:classic :breadpub])

  (defparam flavour
    :default  :classic
    :read     string->keyword
    :validate (fn [s] (elem known-flavours s)))

  (deftask build-docker-image
    :with [flavour]
    ...))
```
