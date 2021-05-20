# lucid

A minimalist, high-performance fish prompt with async git dirty checks that just work.

![Preview](https://user-images.githubusercontent.com/56996/98549007-9520be80-22dd-11eb-884f-b688c9fd26cf.png)

## Features

* Classy, minimal left prompt that surfaces only actionable information
* Asynchronous git dirty state prevents prompt-induced lag even on [massive repositories](https://github.com/llvm/llvm-project)
* Shows current git branch/detached HEAD commit, action (if any), and dirty state
* Restrained use of color and Unicode symbols
* Single file, well-commented, zero-dependency implementation

## Installation

### System Requirements

* [Fish](https://fishshell.com/) ≥ 3.1.0

Install with [Fisher](https://github.com/jorgebucaran/fisher):

```console
fisher install mattgreen/lucid.fish
```

## Performance

Initial rendering is fast enough, considering that the LLVM repo has over **361,000 commits**:

```shell
~/Projects/llvm-project on master •
❯ time fish_prompt

~/Projects/llvm-project on master •
❯
________________________________________________________
Executed in   14.05 millis    fish           external
   usr time    5.96 millis    2.10 millis    3.86 millis
   sys time    6.87 millis    2.54 millis    4.33 millis
```

lucid fetches most git information synchronously. This minimizes the amount of flicker induced by prompt redraws, which can be distracting. This initial time encompasses:

1. getting the git working directory
2. retrieving the current branch
3. figuring out the current action (merge, rebase)
4. starting the async dirty check

This information is memoized to avoid re-computation during prompt redraws, which occur upon completion of the git dirty check, or window resizes.

## Customization

* `lucid_dirty_indicator`: displayed when a repository is dirty. Default: `•`
* `lucid_clean_indicator`: displayed when a repository is clean. Should be at least as long as `lucid_dirty_indicator` to work around a fish bug. Default: ` ` (a space)
* `lucid_cwd_color`: color used for current working directory. Default: `green`
* `lucid_git_color`: color used for git information. Default: `blue`
* `lucid_git_status_in_home_directory`: if set, git information is also
   displayed in the home directory. Default: not set
* `lucid_skip_newline`: if set, doesn't insert a newline before the prompt.
   Default: not set
* `lucid_prompt_symbol`: the prompt symbol. Default: `❯`
* `lucid_prompt_symbol_error`: the prompt symbol when an error occurs.
   Default: `❯`
* `lucid_prompt_symbol_color`: the color of the prompt symbol.
   Default: `$fish_color_normal`
* `lucid_prompt_symbol_error_color`: the color of the prompt symbol when an
   error occurs. Default: `$fish_color_normal`

## Design

Each prompt invocation launches a background job responsible for checking dirty status. If the previous job did not complete, it is killed prior to starting a new job. The dirty check job relays the dirty status back to the main shell via an exit code. This works because there's only three distinct states that can result from a dirty check: dirty, not dirty, or error. Systems programming FTW!

After launching the job, the parent process immediately registers a completion handler for the job. In there, we scoop up the exit status, then update the prompt based on what was found.

The rest is book-keeping and careful coding. There may be a few more opportunities for optimization. Send a PR if you find any!

### Previous Design Iterations

The tough problem here is keeping the execution time low, and communicating the dirty status from the job back to the parent.

Here are a few other approaches that didn't pan out:

* **IPC via universal variables**: variable contents are round-tripped by disk, which is a non-starter
* **IPC via named FIFO**: fish is unable to hold the reader side of a named FIFO open, causing writers to block indefinitely
* **IPC via tmpfs**: macOS does not have a `tmpfs` filesystem available by default
* **Other POSIX IPC facilities**: these would necessitate a third-party binary dependency

## Known Issues

* fish has a bug involving multi-line prompts not being redrawn correctly. You usually see this when invoking `fzf`.
* lucid uses a background job to asynchronously fetch dirty status. If you try to exit while a dirty status has not completed, fish will warn you it is still running. Unfortunately, lucid is not able to `disown` the job because it needs to collect the exit status from it.

## License

[MIT](LICENSE)
