# Test scaffolding

In this tutorial, we're going to write a test for etcd: a distributed consensus
system. I want to encourage you to *type* the code yourself, even if you don't
understand everything yet. It'll help you learn faster, and you won't get as lost when we start updating small pieces of larger functions.

We'll begin by creating a new Leiningen project in any directory.

```bash
$ lein new jepsen.etcdemo
Generating a project called jepsen.etcdemo based on the 'default' template.
The default template is intended for library projects, not applications.
To see other templates (app, plugin, etc), try `lein help new`.
$ cd jepsen.etcdemo
$ ls
CHANGELOG.md  doc/  LICENSE  project.clj  README.md  resources/  src/  test/
```

Like any fresh Clojure project, we have a blank changelog, a directory for
documentation, a copy of the Eclipse Public License, a `project.clj` file,
which tells `leiningen` how to build and run our code, and a README. The
`resources` directory is a place for us to data files--for instance, config
files for a database we want to test. `src` has our source code, organized into
directories and files which match the namespace structure of our code. `test`
is for testing our code. Note that this *whole directory* is a "Jepsen test";
the `test` directory is a convention for most Clojure libraries, and we won't
be using it here.

We'll start by editing `project.clj`, which specifies the project's
dependencies and other metadata. We'll add a `:main` namespace, which is how
we'll run the test from the command line. In addition to depending on the
Clojure language itself, we'll pull in the Jepsen library, and
Verschlimmbesserung: a library for talking to etcd.

```clj
(defproject jepsen.etcdemo "0.1.0-SNAPSHOT"
  :description "A Jepsen test for etcd"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :main jepsen.etcdemo
  :dependencies [[org.clojure/clojure "1.11.1"]
                 [jepsen "0.3.2"]
                 [verschlimmbesserung "0.1.3"]])
```

Let's try running this program with `lein run`.

```bash
$ lein run
Execution error at user/eval140 (form-init15294711163060466740.clj:1).
Cannot find anything to run for: jepsen.etcdemo
…
```

Ah, yes. We haven't written anything to run yet. We need a main function in the `jepsen.etcdemo` namespace, which will receive our command line args and run the test. In `src/jepsen/etcdemo.clj`:

```clj
(ns jepsen.etcdemo)

(defn -main
  "Handles command line arguments. Can either run a test, or a web server for
  browsing results."
  [& args]
  (prn "Hello, world!" args))
```

Clojure, by default, calls our `-main` function with any arguments we passed on
the command line--whatever we type after `lein run`. It takes a variable number
of arguments (that's the `&` symbol), and calls that argument list `args`. We
print that argument list after "Hello World":

```bash
$ lein run hi there
"Hello, world!" ("hi" "there")
```

Jepsen includes some scaffolding for argument handling, running tests, handling
errors, logging, etc. Let's pull in the `jepsen.cli` namespace, call it `cli` for short, and turn our main function into a Jepsen test runner:

```clj
(ns jepsen.etcdemo
  (:require [jepsen.cli :as cli]
            [jepsen.tests :as tests]))


(defn etcd-test
  "Given an options map from the command line runner (e.g. :nodes, :ssh,
  :concurrency, ...), constructs a test map."
  [opts]
  (merge tests/noop-test
         {:pure-generators true}
         opts))

(defn -main
  "Handles command line arguments. Can either run a test, or a web server for
  browsing results."
  [& args]
  (cli/run! (cli/single-test-cmd {:test-fn etcd-test})
            args))
```

`cli/single-test-cmd` is provided by `jepsen.cli`: it parses command line
arguments for a test and calls the provided `:test-fn`, which should return a
map containing all the information Jepsen needs to run a test. In this case,
our test function is `etcd-test`, which takes options from the command line
runner, and uses them to fill in position in an empty test that does nothing:
`noop-test`.

```bash
$ lein run
Usage: lein run -- COMMAND [OPTIONS ...]
Commands: analyze, test
```

With no args, `cli/run!` provides a basic help message, informing us it takes a
command as its first argument. Let's try the test command we added:

Let's give it a shot!

```bash
$ lein run test
08:44:44.491 [main] INFO jepsen.cli - Test options:
 {:concurrency 5,
 :leave-db-running? false,
 :logging-json? false,
 :ssh
 {:dummy? false,
  :username "root",
  :password "root",
  :strict-host-key-checking false,
  :private-key-path nil},
 :argv ("test"),
 :nodes ["n1" "n2" "n3" "n4" "n5"],
 :test-count 1,
 :time-limit 60}

INFO [2023-04-24 08:44:44,552] jepsen test runner - jepsen.core Test version ce80a8b7152e3e70c023a181a84a8891fd51d91f (plus uncommitted changes)
INFO [2023-04-24 08:44:44,552] jepsen test runner - jepsen.core Command line:
lein run test
INFO [2023-04-24 08:44:44,577] jepsen test runner - jepsen.core Running test:
{:remote
 #jepsen.control.retry.Remote{:remote #jepsen.control.scp.Remote{:cmd-remote #jepsen.control.sshj.SSHJRemote{:concurrency-limit 6,
                                                                                                             :conn-spec nil,
                                                                                                             :client nil,
                                                                                                             :semaphore nil},
                                                                 :conn-spec nil},
                              :conn nil}
 :concurrency 5
 :db
 #object[jepsen.db$reify__8900 "0x4f356b98" "jepsen.db$reify__8900@4f356b98"]
 :leave-db-running? false
 :name "noop"
 :logging-json? false
 :start-time
 #object[org.joda.time.DateTime "0x4037cdb0" "2023-04-24T08:44:44.493Z"]
 :net
 #object[jepsen.net$reify__12476
         "0x27055a2a"
         "jepsen.net$reify__12476@27055a2a"]
 :client
 #object[jepsen.client$reify__12223
         "0x33e4068"
         "jepsen.client$reify__12223@33e4068"]
 :barrier
 #object[java.util.concurrent.CyclicBarrier
         "0x9499643"
         "java.util.concurrent.CyclicBarrier@9499643"]
 :pure-generators true
 :ssh
 {:dummy? false,
  :username "root",
  :password "root",
  :strict-host-key-checking false,
  :private-key-path nil}
 :checker
 #object[jepsen.checker$unbridled_optimism$reify__11892
         "0x776d8097"
         "jepsen.checker$unbridled_optimism$reify__11892@776d8097"]
 :argv ("test")
 :nemesis #unprintable "jepsen.nemesis$reify__12579@7a34505a"
 :nodes ["n1" "n2" "n3" "n4" "n5"]
 :test-count 1
 :generator nil
 :os #object[jepsen.os$reify__4576 "0xd5a72cd" "jepsen.os$reify__4576@d5a72cd"]
 :time-limit 60}

INFO [2023-04-24 08:44:48,081] jepsen test runner - jepsen.db Tearing down DB
INFO [2023-04-24 08:44:48,083] jepsen test runner - jepsen.db Setting up DB
INFO [2023-04-24 08:44:48,086] jepsen test runner - jepsen.core Relative time begins now
INFO [2023-04-24 08:44:48,112] jepsen test runner - jepsen.core Run complete, writing
INFO [2023-04-24 08:44:48,144] jepsen test runner - jepsen.core Analyzing...
INFO [2023-04-24 08:44:48,145] jepsen test runner - jepsen.core Analysis complete
INFO [2023-04-24 08:44:48,146] jepsen results - jepsen.store Wrote /jepsen/jepsen.etcdemo/store/noop/20230424T084444.493Z/results.edn
INFO [2023-04-24 08:44:48,164] jepsen test runner - jepsen.core {:valid? true}


Everything looks good! ヽ(‘ー`)ノ
```

We can see Jepsen tear down and set up the (non-existent) DB and start the test
run, which finishes immediately as we haven't given our worker nodes anything
to do. It then analyzes the run using the
[unbridled-optimism](https://jepsen-io.github.io/jepsen/jepsen.checker.html#var-unbridled-optimism)
checker -- which, as the name implies, doesn't do any checking -- and writes
the test result to the `store` directory, before printing a brief summary of
the analysis.

`noop-test` uses nodes named `n1`, `n2`, ... `n5` by default. If your nodes
have different names, this test will fail to connect to them. That's OK! You can change that by passing node names on the command line:

```bash
$ lein run test --node foo.mycluster --node 1.2.3.4
```

... or by passing a filename that has a list of nodes in it, one per line. If
you're using the AWS Marketplace cluster, you've already got a file called
`nodes` in your home directory, ready to go.

```bash
$ lein run test --nodes-file ~/nodes
```

If you're still hitting SSH errors at this point, you should check that your
SSH agent is running and has keys for all your nodes loaded. `ssh some-db-node`
should work without a password. You can override the username, password, and
identity file at the command line as well; see `lein run test --help` for
details.

```bash
$ lein run test --help
#object[jepsen.cli$test_usage 0x7ddd84b5 jepsen.cli$test_usage@7ddd84b5]

  -h, --help                                                  Print out this message and exit
      --test-count NUMBER         1                           How many times should we repeat a test?
      --no-ssh                    false                       If set, doesn't try to establish SSH connections to any nodes.
      --nodes-file FILENAME                                   File containing node hostnames, one per line.
      --time-limit SECONDS        60                          Excluding setup and teardown, how long should a test run for, in seconds?
      --logging-json              false                       Use JSON structured output in the Jepsen log.
      --username USER             root                        Username for logins
      --concurrency NUMBER        1n                          How many workers should we run? Must be an integer, optionally followed by n (e.g. 3n) to multiply by the number of nodes.
  -n, --node HOSTNAME             ["n1" "n2" "n3" "n4" "n5"]  Node(s) to run test on. Flag may be submitted many times, with one node per flag.
      --leave-db-running          false                       Leave the database running at the end of the test., so you can inspect it.
      --password PASS             root                        Password for sudo access
      --strict-host-key-checking  false                       Whether to check host keys
      --ssh-private-key FILE                                  Path to an SSH identity file
      --nodes NODE_LIST                                       Comma-separated list of node hostnames.
```

We'll use `lein run test ...` throughout this guide to re-run our Jepsen test. Each time we run a test, Jepsen will create a new directory in `store/`. You can see the latest results in `store/latest`:

```bash
$ ls store/latest/
history.edn  history.txt  jepsen.log  results.edn  test.jepsen
```

`history.edn` and `history.txt` show the operations the test performed--ours are
empty, since the noop test doesn't perform any ops. `jepsen.log` has a copy of
the console log for that test. `results.edn` shows the analysis of the test,
which we see at the end of each run. Finally, `test.jepsen` has the raw data
for the test, including the full machine-readable history and analysis, if you
need to perform post-hoc analysis.

Jepsen also comes with a built-in web browser for browsing these results. Let's add it to our `main` function:

```clj
(defn -main
  "Handles command line arguments. Can either run a test, or a web server for
  browsing results."
  [& args]
  (cli/run! (merge (cli/single-test-cmd {:test-fn etcd-test})
                   (cli/serve-cmd))
            args))
```

This works because `cli/run!` takes a map of command names to specifications of
how to run those commands. We're merging those maps together with `merge`.

```bash
$ lein run serve
13:29:21.425 [main] INFO jepsen.web - I'll see YOU after the function
13:29:21.428 [main] INFO jepsen.cli - Listening on http://0.0.0.0:8080/
```

We can open `http://localhost:8080` in a web browser to explore the history of
our test results. Of course, the serve command comes with its own options and
help message:

```bash
$ lein run serve --help
Usage: lein run -- serve [OPTIONS ...]

  -h, --help                  Print out this message and exit
  -b, --host HOST    0.0.0.0  Hostname to bind to
  -p, --port NUMBER  8080     Port number to bind to
```

Open up a new terminal window, and leave the web server running there. That way
we can see the results of our tests without having to start and stop it
repeatedly.

With this groundwork in place, we'll write the code to [set up and tear down the database](02-db.md).
