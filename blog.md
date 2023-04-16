[Git]: https://git-scm.com/
[Haskell]: https://www.haskell.org/
[Jira]: https://jira.atlassian.com/
[Kubuntu]: https://kubuntu.org/
[Markdown]: https://daringfireball.net/projects/markdown/
[Org-mode]: https://orgmode.org/
[Pandoc]: https://pandoc.org/
[pandoc-github]: https://github.com/jgm/pandoc
[pandoc-installation]: https://pandoc.org/installing.html
[Stack]: https://docs.haskellstack.org/en/stable/
[stack_quick_start_guide]: https://docs.haskellstack.org/en/stable/#quick-start-guide

# Haskell Programming

Exploring [Haskell] by looking at source code, building libraries and executables, making changes, and writing about it.

## Let's Get Started

1. I'm using a fresh installation of [Kubuntu] on my laptop: 
   `cat /etc/  issue` gives: `Ubuntu 22.04.2 LTS`.

2. Next, I installed [Stack]:

       $ curl -sSL https://get.haskellstack.org/ | sh

   At first, this failed, because `curl` was not installed, 
   `sudo apt install curl` fixed it.

3. To test if this worked, I ran the 
   [Stack Quick Start guide][stack_quick_start_guide].

### `pandoc`: A Universal Markup Converter Written in Haskell

[Pandoc] is an open-source tool to convert between different _Markup_ formats. Besides being my first choice whenever I need to save my notes written in [Markdown] or [Org-mode] into _Markup_ for a tool like [Jira], it is written in [Haskell]; runs on many platforms, including Windows, Mac, and Linux; has over 16000 commits on Github; ~117 releases; and more than 400 contributors.

The source code of [Pandoc][pandoc] can be cloned, see [pandoc-github]: 

    $ git clone https://github.com/jgm/pandoc

This creates a `pandoc` directory from where you ca build [Pandoc]. As explained in the installation notes ([pandoc-installation]), it can be built with [Stack]. The commands are:

    $ cd pandoc
    $ stack setup
    $ stack install pandoc-cli

My first attempts to build [Pandoc] failed. To fix the build issues, I tried:

1. Check out a release of [Pandoc] and not the latest commit on the `master`
   branch. There is a note in the [pandoc-installation] suggesting the very latest development commit is not guaranteed to build successfully. The current release is pandoc 3.1.2 (2023-03-26) (see https://pandoc.org/releases.html). So, checkout the commit corresponding to that release:

       $ git checkout 3.1.2    # results in a detached HEAD
       $ git branch vb         # so, create a new branch
       $ git switch vb         # and switch to that branch

   Note that creating and switching to a new branch are not necessary to build [Pandoc]. However, my intention is to track my changes to [Pandoc] source code in future commits and then it is better not to work on a detached HEAD (see the _DETACHED HEAD_ section for `git checkout` command at https://git-scm.com/docs/git-checkout).

   The build still failed after this change.

2. After inspecting the output of the `stack install pandoc-cli` command in
   more detail, I found out that I was missing two Linux libraries: `pkg-config` and `zlib1g-dev`. Both can be installed with:

       sudo apt install pkg-config zlib1g-dev

   Now the build was successful, but took quite long. It installed `pandoc` locally:

       $ which pandoc
       /home/victor/.local/bin/pandoc

   The version of this `pandoc` installation is, as expected, 3.1.2:

       $ pandoc --version
       pandoc 3.1.2
       <<<cut>>>

So far, I have not shown any [Haskell] of code [Pandoc].

### The `main` of [Pandoc]

Each [Haskell] program has a function `main :: IO ()` that is/specifies the program's behaviour. Let's see if there are any `main`'s in the [Pandoc] source code (execute the `grep` command in the `pandoc` directory created by cloning the [Pandoc] repository):

    $ grep -nr 'main :: IO ()' *
    benchmark/benchmark-pandoc.hs:91:main :: IO ()
    doc/using-the-pandoc-api.md:65:main :: IO ()
    doc/using-the-pandoc-api.md:219:main :: IO ()
    doc/using-the-pandoc-api.md:292:main :: IO ()
    doc/filters.md:149:main :: IO ()
    doc/filters.md:294:main :: IO ()
    doc/filters.md:386:main :: IO ()
    pandoc-cli/src/pandoc.hs:48:main :: IO ()
    pandoc-lua-engine/test/test-pandoc-lua-engine.hs:9:main :: IO ()
    pandoc-server/README.md:17:main :: IO ()
    src/Text/Pandoc.hs:32:> main :: IO ()
    test/test-pandoc.hs:103:main :: IO ()

Great, there are quite a few `main`'s, apparently. Since we build `pandoc-cli` above, let's open `pandoc-cli/src/pandoc.hs` and have a look at the `main` at line 48:

```haskell
main :: IO ()
main = E.handle (handleError . Left) $ do
    prg <- getProgName
    rawArgs <- map UTF8.decodeArg <$> getArgs
    let hasVersion = getAny $ foldMap
            (\s -> Any (s == "-v" || s == "--version"))
            (takeWhile (/= "--") rawArgs)
    when hasVersion versionInfo
    case prg of
        "pandoc-server.cgi" -> runCGI
        "pandoc-server"     -> runServer rawArgs
        "pandoc-lua"        -> runLuaInterpreter prg rawArgs
        _ ->
            case rawArgs of
                "lua" : args   -> runLuaInterpreter "pandoc lua" args
                "server": args -> runServer args
                args           -> do
                    engine <- getEngine
                    res <- parseOptionsFromArgs options defaultOpts prg args
                    case res of
                        Left e -> handleOptInfo engine e
                        Right opts -> convertWithOpts engine opts
```

Observations/assumptions:

1. `E.handle (handleError . Left)` takes care of error handling. Exactly how
   this works is a topic of another day.

2. The program name (name of the executble) is read into variable `prg` 
   and, a few lines later, used to determine the desired behaviour of the executable. There are 4 possibilities: `pandoc-server.cgi`, `pandoc-server`, `pandoc-lua`, and "anything else".

   In the `pandoc --version` example above, `prg` will get the value `"pandoc"` which selects the 4th possibility.

3. The command line option for version information (`-v` or `--version`)
   gets a special treatment compared to other command line options. The variable `hasVersion` indicates if the version option was present. It is
   used in a `when` statement few lines later.
   
   The effect of the `when hasVersion versionInfo` line is that version information is printed before the other command line arguments and switches are processed, regardless of the position of the version switch on the command line.

4. The line `(takeWhile (/= "--") rawArgs)` is needed to stop command
   line options processing when the string `"--"` is detected. The 
   arguments following this symbol are not interpreted as command line
   options to [Pandoc].

5. An "engine" is determined with `getEngine`. It is unclear from this
   source code what the engine does and how it is selected, but my guess is that the engine is selected based on the `pandoc` input and output formats specified by the user or by default settings `defaultOpts`.

   Note that `getEngine`, which I assume is a function, is not applied in the code above; it just gets assigned `engine <- getEngine`. In the final two lines, `engine` is an argument to both `handleOptInfo` and `converWithOpts`.

6. If option parsing identified errors, case `Left e`, there is a call to 
   `handleOptInfo`. The error(s) `e` and `engine` are arguments. This suggests that there is some form of `engine` specific error handling. It makes sense, because command line switches depend on selected input and output formats.

7. If option parsing was successful, case `Right opts`, there is a call to 
   `convertWithOpts`. Again, the `engine` and the parsed options (now called `opts`) are passed as arguments. It seems the `convertWithOpts` is responsible for the actual conversion between the input and output formats. However, if our assumtions above are correct, it will be a generic functions that leaves the actual format dependent behaviour to the `engine`.

 